# Resumen de Arquitectura: Marketplace Serverless en Java

## 1. Objetivo del Sistema

Este proyecto es una Prueba de Concepto (PoC) diseñada para evaluar el rendimiento, los tiempos de arranque en frío (cold start) y el consumo de memoria de **Spring Cloud Function** frente a **Quarkus Funqy** ejecutándose en AWS Lambda (Java 21). El dominio de negocio es un marketplace simulado de una sola página (SPA) que presenta productos generados por IA a través de AWS Bedrock.

## 2. Stack Tecnológico

* **Frontend:** HTML5, JavaScript puro (Vanilla), Tailwind CSS (vía CDN).
* **Gestión de API:** Amazon API Gateway (HTTP API v2).
* **Cómputo (Backend):** AWS Lambda (Runtime Java 21).
* **Frameworks:** Spring Boot 3 (Spring Cloud Function) y Quarkus 3 (Funqy).
* **Base de Datos:** Amazon DynamoDB (Diseño de Tabla Única, Bajo Demanda / On-Demand).
* **Integración de IA:** Amazon Bedrock (Anthropic Claude 3 Haiku).
* **FinOps y Seguridad:** AWS Budgets, Amazon SNS, IAM.

---

## 3. Arquitectura de Alto Nivel (Modelo C4 - Nivel de Contenedores)

La arquitectura se divide en dos planos lógicos distintos: el **Núcleo de la Aplicación (App Core)** que maneja la lógica de negocio, y la **Capa Global (Global Overlay)** que aplica las políticas de FinOps y seguridad.

### 3.1. Núcleo de la Aplicación (App Core)

```text
[ Navegador del Cliente ]
        │
        ▼
[ Amazon S3 ] <─── (Hosting SPA Estático)
        │
        ▼
[ API Gateway (HTTP API) ] ── [ Límite de Peticiones: 5 req/s ]
        │
        ├────────────────────────────────┬────────────────────────────────┐
        ▼                                ▼                                ▼
[ Lambda: Moderación e IA ]     [ Lambda: Spring Cloud ]         [ Lambda: Quarkus ]
  (Filtro Prompt + LLM)         (Procesamiento Checkout)       (Procesamiento Checkout)
        │                                │                                │
        └────────────────────────────────┴────────────────────────────────┘
                                         │
                                         ▼
                              [ Amazon DynamoDB ]
                   (Tabla de Catálogo y Métricas de Compras)

```

### 3.2. Gobernanza y FinOps (Capa Global / Global Overlay)

```text
[ AWS Budgets ] ──► (Alerta: $7.00 USD) ──► [ Tema Amazon SNS ]
                                                  │
                                                  ▼
                                       [ Lambda: Kill-Switch ]
                                                  │
                    ┌─────────────────────────────┴─────────────────────────────┐
                    ▼                                                           ▼
      [ Fijar Concurrencia a 0 ]                                [ Límite de Ruta a 0 ]
(Desactiva Lambdas Spring, Quarkus y Mod.)               (Bloquea tráfico entrante de API Gateway)

```

---

## 4. Desglose de Componentes y Responsabilidades

1. **Frontend (Amazon S3):** Aloja la aplicación estática de una sola página (SPA). Se utiliza S3 en lugar de CloudFront para minimizar la complejidad de configuración y los costos base para un entorno experimental de corta duración.
2. **API Gateway (HTTP API v2):** Actúa como punto de entrada, manejando CORS y aplicando un límite de tasa global estricto (5 peticiones/segundo) para mitigar abusos.
3. **Lambda de Moderación (Java/Python):** Intercepta las solicitudes de generación de productos. Valida las entradas contra una lista negra de expresiones regulares (regex) en memoria. Si es válida, interactúa con AWS Bedrock para generar los metadatos del producto (JSON).
4. **Lambdas de Spring y Quarkus (Java 21):** Funciones gemelas que implementan una lógica de checkout idéntica. Persisten las órdenes en DynamoDB y devuelven métricas de tiempo de ejecución (tiempo de ejecución, indicador de cold start) al cliente para la evaluación comparativa (benchmarking).
5. **DynamoDB:** Configurado en modo de Pago por Solicitud (On-Demand). Almacena tanto el catálogo de productos como los registros de transacciones.
6. **CloudWatch Logs:** Configurado con una estricta **política de retención de 3 días** para evitar la acumulación ilimitada de costos de almacenamiento.

---

## 5. Decisiones Clave de Arquitectura (ADRs)

* **ADR-001: HTTP API sobre REST API (ALB).**
* *Contexto:* Los Application Load Balancers incurren en costos de provisión por hora.
* *Decisión:* Se seleccionó HTTP API Gateway para lograr un modelo 100% de pago por uso ($1.00 USD por millón de peticiones).


* **ADR-002: Generación Híbrida de Imágenes.**
* *Contexto:* La generación nativa de imágenes a través de AWS Bedrock (Titan/Nova Canvas) cuesta ~$0.04 USD por imagen, lo que corre el riesgo de agotar rápidamente el presupuesto.
* *Decisión:* Bedrock genera únicamente palabras clave (keywords) en texto. El backend construye dinámicamente URLs que apuntan a un CDN gratuito (Unsplash) basándose en esas palabras clave.


* **ADR-003: Kill-Switch Automatizado de FinOps.**
* *Contexto:* Restricción de presupuesto estricta de $10.00 USD.
* *Decisión:* AWS Budgets activa un tema de SNS a los $7.00 USD. Un Lambda suscriptor ejecuta comandos del SDK de AWS para reducir los límites del API Gateway y la concurrencia de las Lambdas a 0, deteniendo por completo las capacidades de facturación del sistema.



---

## 6. Proyecciones Financieras y Presupuesto

Duración objetivo del despliegue: 1 Semana.
Tráfico estimado: 10,000 peticiones.
Límite estricto de presupuesto (Hard Cap): $10.00 USD.

| Servicio AWS | Métrica de Facturación | Uso Estimado | Costo Proyectado (USD) |
| --- | --- | --- | --- |
| **Amazon S3** | Almacenamiento y peticiones GET | 10 MB / 10k GETs | < $0.01 |
| **API Gateway** | Peticiones HTTP API | 15,000 llamadas | < $0.02 |
| **AWS Lambda** | Invocaciones y GB-segundos | ~10k llamadas (512 MB) | $0.00 (Capa Gratuita) |
| **Amazon DynamoDB** | WRU / RRU (On-Demand) | 5k WRU / 20k RRU | < $0.02 |
| **AWS Bedrock** | Tokens Entrada/Salida (Claude 3 Haiku) | ~500 prompts | ~$1.50 |
| **CloudWatch Logs** | Ingesta y Almacenamiento (Retención 3 días) | ~100 MB | ~$0.05 |
| **Total Estimado** |  |  | **~$1.60 Máx.** |