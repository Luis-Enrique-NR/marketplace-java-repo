# Especificación del Sistema (SYSTEM-SPEC.md)

## 1. Propósito y Contexto

El sistema es una Prueba de Concepto (PoC) técnica diseñada para demostrar y comparar el rendimiento de dos frameworks Java (Spring Boot y Quarkus) en un entorno Serverless. Adicionalmente, busca demostrar la integración de capacidades de Inteligencia Artificial Generativa bajo estrictos controles de costos (FinOps). El objetivo principal es recolectar métricas observables (latencia, tiempos de arranque en frío) de ambos frameworks procesando cargas de trabajo idénticas, y presentar estos resultados al usuario en tiempo real.

## 2. Alcance

* **Incluye:** Visualización de un catálogo de productos, simulación de carrito de compras, generación dinámica de productos ficticios mediante IA basada en prompts de texto, recolección de métricas de ejecución por transacción, y un mecanismo de corte automático de servicio por presupuesto.
* **Excluye:** Pasarelas de pago reales, autenticación/autorización de usuarios (login), gestión de envíos reales, persistencia de sesiones a largo plazo, y generación de imágenes nativas por IA (reemplazado por resolución estática vía CDN).

## 3. Actores y Permisos

* **Usuario Anónimo (Comunidad):** Puede visualizar el catálogo, generar productos mediante prompts de texto y realizar simulaciones de compra. No requiere autenticación.
* **Sistema de IA (Actor Externo):** Recibe prompts validados y devuelve atributos estructurados (JSON) y palabras clave de productos.
* **Monitor FinOps (Sistema Interno):** Evalúa el gasto acumulado y tiene permisos absolutos para suspender las operaciones del sistema.

## 4. Historias de Usuario (US)

* **US-01 [Catálogo]:** Como usuario, quiero ver una lista de productos disponibles para poder agregarlos a mi carrito de compras.
* **US-02 [Generación IA]:** Como usuario, quiero ingresar una breve descripción en texto para que la IA genere un producto ficticio y lo añada al catálogo global.
* **US-03 [Checkout & Métricas]:** Como usuario, quiero finalizar mi compra para recibir un reporte instantáneo que indique qué framework (Spring o Quarkus) procesó mi pedido, el tiempo de ejecución y si fue un arranque en frío.
* **US-04 [Protección FinOps]:** Como administrador, quiero que el sistema rechaze todas las peticiones automáticamente si se alcanza el umbral de gasto para evitar cobros indeseados.

## 5. Requisitos Funcionales (RF)

* **RF-01:** El sistema debe proveer una operación para listar los productos activos en el catálogo.
* **RF-02:** El sistema debe aceptar una cadena de texto (prompt) y utilizar un LLM para generar: título, descripción, precio, categoría y palabras clave asociadas.
* **RF-03 [Prompt Guard]:** El sistema debe interceptar y evaluar el prompt contra una lista negra de palabras (ej. armas, NSFW) antes de procesarlo.
* **RF-04:** El sistema debe generar una URL de imagen válida utilizando un servicio CDN de terceros (ej. Unsplash) basándose en las palabras clave generadas, sin invocar generación de imágenes por IA.
* **RF-05:** El sistema debe registrar la transacción de compra simulada.
* **RF-06:** El sistema debe enrutar las peticiones de compra de manera distribuida (Round Robin u otro mecanismo de alternancia) entre los dos motores backend (Spring y Quarkus).
* **RF-07:** El sistema debe adjuntar metadatos de ejecución en la respuesta de cada compra: identificador del framework, latencia en milisegundos y bandera booleana de arranque en frío (*cold start*).

## 6. Requisitos No Funcionales (RNF)

* **RNF-01 [Rendimiento]:** Las operaciones en caliente (*warm start*) del backend deben responder en menos de 200 ms.
* **RNF-02 [Escalabilidad y Límites]:** El sistema debe limitar el tráfico de entrada a un máximo de 5 peticiones por segundo (req/s) de forma global para mitigar abusos (Throttling).
* **RNF-03 [Disponibilidad y FinOps]:** El sistema debe suspender todas las operaciones de escritura/lectura y devolver un error HTTP cuando el costo acumulado alcance los $7.00 USD.
* **RNF-04 [Observabilidad]:** Todo registro o log generado por el sistema debe ser retenido por un máximo de 3 días calendario.

## 7. Reglas de Negocio (RN)

* **RN-01:** Los productos generados por IA solo pueden pertenecer a categorías predefinidas: tecnología, hogar, oficina, ropa.
* **RN-02:** Si un prompt contiene palabras prohibidas, la solicitud debe ser rechazada inmediatamente sin penalizar el presupuesto del sistema.
* **RN-03:** El costo total de operación del sistema durante la vigencia del demo no debe exceder bajo ninguna circunstancia los $10.00 USD.

## 8. Flujos y Escenarios

* **Flujo Principal 1 (Checkout Exitoso):** El usuario envía un carrito válido -> El sistema enruta a Spring o Quarkus -> Se registra la orden -> Se calcula el tiempo de ejecución y estado del contenedor -> El usuario recibe el reporte de métricas.
* **Flujo Principal 2 (Generación IA):** El usuario envía prompt -> Pasa el filtro de moderación -> La IA devuelve atributos -> El sistema construye la URL de imagen (híbrida) -> El producto se guarda en el catálogo -> Se devuelve el producto creado.
* **Escenario Alternativo 1 (Prompt Rechazado):** El usuario envía prompt con palabra en lista negra -> El interceptor rechaza la solicitud -> Retorna HTTP 400 con mensaje genérico de error de moderación.
* **Escenario Límite (Presupuesto Excedido):** El monitor FinOps detecta $7.00 USD -> Se activa el Kill-Switch -> El usuario intenta comprar -> Retorna HTTP 429 o 503.

## 9. Criterios de Aceptación (CA)

* **CA-01 (Asociado a US-03):** La respuesta del endpoint de checkout debe ser un JSON válido que contenga explícitamente los campos `framework_used` (String), `execution_time_ms` (Integer) y `is_cold_start` (Boolean).
* **CA-02 (Asociado a US-02):** Un producto generado debe poseer una `image_url` funcional apuntando a un CDN público que contenga los keywords del producto en su cadena, y no a un bucket privado.
* **CA-03 (Asociado a US-04):** Al simular una alerta de límite de presupuesto, los endpoints públicos deben dejar de procesar lógica de negocio en un tiempo razonable (< 2 minutos) devolviendo códigos de error HTTP de limitación.

## 10. Datos y Estados

| Entidad | Atributos Principales | Estados Posibles |
| --- | --- | --- |
| **Producto** | ID, Título, Descripción, Precio, URL Imagen, Origen (Base / IA) | Activo |
| **Orden** | ID, Fecha, Lista de Ítems, Total, Métricas de Ejecución | Procesada |
| **Sistema** | Presupuesto Consumido, Límite Actual | Operativo, Suspendido (Kill-Switch) |

## 11. Interfaces y Contratos

*Las estructuras de datos exactas están externalizadas en `api-contracts.json`. Se definen las siguientes operaciones principales:*

* `GET /api/products`: Devuelve la lista de productos del catálogo.
* `POST /api/products/generate`: Recibe `{ "prompt": "string" }`. Devuelve la entidad Producto.
* `POST /api/checkout`: Recibe una lista de IDs de productos y cantidades. Devuelve el recibo y objeto de métricas.

## 12. Manejo de Errores

* **HTTP 400 (Bad Request):** Errores de validación de entrada (ej. carrito vacío, prompt vacío, prompt con palabras prohibidas).
* **HTTP 429 (Too Many Requests):** Cuando el usuario excede el límite de 5 req/s (Rate Limiting).
* **HTTP 503 (Service Unavailable):** Cuando el sistema ha sido apagado intencionalmente por el mecanismo FinOps.
* **Idempotencia:** No se requiere control estricto de idempotencia para la creación de órdenes dada la naturaleza efímera del demo.

## 13. Restricciones y Supuestos

* **Supuesto 1:** El LLM siempre responderá en un formato estructurado válido (JSON) gracias al System Prompt. En caso de alucinación del formato, la petición fallará y no se reintentará automáticamente.
* **Restricción 1:** El tiempo de desarrollo efectivo está limitado a bloques de 15 minutos diarios.
* **Restricción 2:** Todo el entorno dejará de existir o quedará inoperativo máximo una semana después de su lanzamiento público.
* *Pregunta Abierta:* ¿Cómo se manejará el estado de la concurrencia del catálogo si dos usuarios generan productos simultáneamente? *(Supuesto provisional: La base de datos sin esquema estricto permitirá escrituras concurrentes sin bloqueo).*

## 14. Validación Técnica

Para validar el sistema previo al lanzamiento público:

1. **Prueba de Carga Básica:** Ejecutar 50 peticiones simultáneas para verificar el comportamiento del rate limiter (debe arrojar HTTP 429 para las excedentes).
2. **Prueba de Cold Start:** Enviar una petición tras 15 minutos de inactividad a ambos frameworks. El campo `is_cold_start` debe retornar `true` y `execution_time_ms` debe reflejar la diferencia temporal natural entre Spring y Quarkus.
3. **Prueba de FinOps:** Inyectar un evento simulado de Amazon SNS con la alarma de presupuesto. Verificar que las invocaciones posteriores fallen por limitación de capacidad.

## 15. Trazabilidad

| Objetivo de Negocio / Técnico | Historia de Usuario | Requisito Funcional | Criterio de Aceptación |
| --- | --- | --- | --- |
| Demostrar capacidades de IA Generativa | US-02 | RF-02, RF-03, RF-04 | CA-02 |
| Benchmarking Spring vs Quarkus | US-03 | RF-05, RF-06, RF-07 | CA-01 |
| Control FinOps estricto (< $10) | US-04 | RF-06 | CA-03 |