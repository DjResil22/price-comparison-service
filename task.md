# Plan de Trabajo: Intelligent Product Price Aggregator API (Agente Autónomo)

> **Rol:** Desarrollador Único (Full-Stack Backend / Software Engineer)  
> **Objetivo:** Implementar la API REST en Java 21 / Spring Boot 3.3+ con Arquitectura Hexagonal, OpenAPI Contract-First, Agente de IA Autónomo con Tool-Calling (construcción dinámica de URLs y extracción semántica de contenido scrapeado), Redis Fuzzy Caching con llaves compuestas, Resilience4j, Spring Security y Testcontainers.  
> **Regla Crítica del Proyecto:** El código fuente **NO DEBE CONTENER COMENTARIOS** (ni inline, ni bloques, ni Javadoc). El código debe ser 100% autodocumentado mediante nombres claros y estructura limpia.

---

## Índice de Fases

1. [Fase 1: Configuración del Entorno, Dependencias y Build Tool](#fase-1-configuración-del-entorno-dependencias-y-build-tool)
2. [Fase 2: Definición de Contratos OpenAPI (Contract-First sin productUrl)](#fase-2-definición-de-contratos-openapi-contract-first-sin-producturl)
3. [Fase 3: Capa de Dominio (Domain Core - Zero Framework Dependencies)](#fase-3-capa-de-dominio-domain-core---zero-framework-dependencies)
4. [Fase 4: Capa de Aplicación (Use Cases & Orquestación con Agente de IA)](#fase-4-capa-de-aplicación-use-cases--orquestación-con-agente-de-ia)
5. [Fase 5: Infraestructura - Persistencia, Cache Compuesto y Fuzzy Matching en Redis](#fase-5-infraestructura---persistencia-cache-compuesto-y-fuzzy-matching-en-redis)
6. [Fase 6: Infraestructura - Adaptador de Web Scraping Resiliente (Resilience4j)](#fase-6-infraestructura---adaptador-de-web-scraping-resiliente-resilience4j)
7. [Fase 7: Infraestructura - Agente Autónomo de IA: Construcción de URLs y Extracción Semántica](#fase-7-infraestructura---agente-autónomo-de-ia-construcción-de-urls-y-extracción-semántica)
8. [Fase 8: Infraestructura - Adaptadores Web REST (OpenAPI Controllers) y Compresión HTTP](#fase-8-infraestructura---adaptadores-web-rest-openapi-controllers-y-compresión-http)
9. [Fase 9: Infraestructura - Seguridad, RBAC y Anti-DDoS (Bucket4j)](#fase-9-infraestructura---seguridad-rbac-y-anti-ddos-bucket4j)
10. [Fase 10: Infraestructura - Sincronización Programada de Precios (@Scheduled)](#fase-10-infraestructura---sincronización-programada-de-precios-scheduled)
11. [Fase 11: Infraestructura - Observabilidad, Logging Estructurado y Actuator](#fase-11-infraestructura---observabilidad-logging-estructurado-y-actuator)
12. [Fase 12: Suite de Pruebas Unitarias y de Integración (QA & Verification)](#fase-12-suite-de-pruebas-unitarias-y-de-integración-qa--verification)
13. [Fase 13: Despliegue Local, Docker Compose y Verificación Final](#fase-13-despliegue-local-docker-compose-y-verificación-final)

---

## Fase 1: Configuración del Entorno, Dependencias y Build Tool

- [x] **1.1. Ajuste de versión de Spring Boot y Java en `pom.xml`**
  - Configurar Java 21 LTS y Spring Boot 3.3.x.
- [x] **1.2. Incorporación de dependencias principales en `pom.xml`**
  - Spring AI (`spring-ai-openai-spring-boot-starter` o LangChain4j con soporte de Function Calling).
  - Spring Boot Starter Data Redis y soporte para Lettuce / RediSearch.
  - Spring Security y `spring-boot-starter-oauth2-resource-server` (JWT).
  - Bucket4j con backend Redis (`bucket4j-redis` / `bucket4j-core`).
  - Resilience4j (`resilience4j-spring-boot3`, `resilience4j-circuitbreaker`, `resilience4j-retry`, `resilience4j-timelimiter`).
  - Web scraping / HTTP client (Jsoup y `RestClient`).
  - Jackson JSR310 y OpenAPI Tools (`jackson-databind-nullable`).
  - Spring Boot Starter Actuator y Micrometer Tracing.
- [x] **1.3. Incorporación de dependencias de prueba**
  - JUnit 5, Mockito, AssertJ.
  - Testcontainers (`testcontainers`, `junit-jupiter`, `redis`).
  - WireMock (`wiremock-standalone` / `wiremock-spring-boot`).
- [x] **1.4. Configuración del plugin `openapi-generator-maven-plugin`**
  - Configurar plugin en `pom.xml` apuntando a `src/main/resources/openapi/openapi.yaml`.
  - Configurar generación en modo `spring` con `interfaceOnly: true`, `useSpringBoot3: true`, y generación de DTOs inmutables (Java Records).
- [x] **1.5. Configuración de perfiles de propiedades (`application.yml`)**
  - Habilitar compresión HTTP GZIP para minimizar payloads:
    ```yaml
    server:
      compression:
        enabled: true
        mime-types: application/json
        min-response-size: 512
    ```
  - Configurar perfiles `application.yml`, `application-dev.yml` y `application-test.yml`.
- [x] **1.6. Configuración de Docker Compose local**
  - Crear `docker-compose.yml` en la raíz con Redis Stack (incluyendo RediSearch).

---

## Fase 2: Definición de Contratos OpenAPI (Contract-First sin productUrl)

- [ ] **2.1. Creación del archivo de especificación `openapi.yaml`**
  - Ubicación: `src/main/resources/openapi/openapi.yaml`.
- [ ] **2.2. Definición del endpoint de consulta de precios con nuevos parámetros**
  - Ruta: `GET /api/v1/products/prices`.
  - Query parameters:
    - `productName` (obligatorio, string, regex `^[a-zA-ZáéíóúÁÉÍÓÚñÑüÜ\s]+$`).
    - `brand` (opcional, string, regex `^[a-zA-Z0-9\s]+$`).
    - `city` (opcional, string, regex `^[a-zA-ZáéíóúÁÉÍÓÚñÑüÜ\s]+$`).
    - `limit` (opcional, integer, default: `10`, min: `1`, max: `50`).
  - Respuestas:
    - `200 OK`: Lista de tiendas agrupadas con su ciudad y lista de productos (**sin `productUrl`**).
    - `400 Bad Request`: Parámetro con formato inválido o inyección detectada.
    - `401 Unauthorized`: Token ausente o inválido.
    - `429 Too Many Requests`: Límite de tasa excedido.
    - `500 Internal Server Error`: Error interno del servidor.
- [ ] **2.3. Definición de endpoints administrativos de tiendas**
  - `POST /api/v1/admin/stores`: Registrar tienda con `name`, `baseUrl` (ej. `https://www.farmatodo.com.ve`), `supportedCities` y `active`.
  - `PUT /api/v1/admin/stores/{storeId}`: Actualizar configuración de tienda.
  - `DELETE /api/v1/admin/stores/{storeId}`: Eliminar tienda.
  - `GET /api/v1/admin/stores`: Listar configuraciones de tiendas registradas.
- [ ] **2.4. Definición de esquemas de datos estándar (DTOs)**
  - `StoreCatalogResponseDto`: `store` (string), `city` (string), `products` (array de `ProductItemDto`).
  - `ProductItemDto`: `name` (string), `brand` (string), `priceUSD` (string), `priceVES` (string). *(Nota: `productUrl` explícitamente omitido).*
  - `StoreConfigDto`: `id`, `name`, `baseUrl`, `supportedCities`, `active`.
  - `ErrorResponseDto`: código de error, mensaje y timestamp.
- [ ] **2.5. Ejecución y validación de generación de código**
  - Ejecutar `./mvnw clean compile` y verificar generación de interfaces controladoras y DTOs en `target/generated-sources/openapi`.

---

## Fase 3: Capa de Dominio (Domain Core - Zero Framework Dependencies)

- [ ] **3.1. Estructura de paquetes de dominio**
  - Paquetes: `com.pricecomparison.domain.model`, `com.pricecomparison.domain.port.in`, `com.pricecomparison.domain.port.out`, `com.pricecomparison.domain.exception`.
  - Cero dependencias de frameworks externos en este módulo.
- [ ] **3.2. Modelos de Dominio y Value Objects (Inmutables con Records)**
  - Crear `ProductSearchCriteria`: Value Object con `productName`, `brand`, `city`, `limit`.
    - Sanitización y normalización automática: minúsculas, trim y desacentuación (`"Jabón Líquido"` $\to$ `"jabon liquido"`).
    - Generación de llave compuesta de cache (ej. `prices:{city}:{brand}:{productName}`).
  - Crear `ProductItem`: Value Object con `name`, `brand`, `priceUSD`, `priceVES` (sin URLs).
  - Crear `StoreCatalog`: Modelo de dominio con `storeName`, `city`, y `List<ProductItem>`.
  - Crear `StoreConfig`: Entidad de dominio con `id`, `name`, `baseUrl`, `supportedCities`, `active`.
- [ ] **3.3. Excepciones de Dominio**
  - `InvalidProductSearchException`, `StoreNotFoundException`, `ProductNotFoundException`, `ScrapingException`, `AiProcessingException`.
- [ ] **3.4. Definición de Puertos de Salida (Output Ports - Interfaces)**
  - `ProductCachePort`:
    - `Optional<List<StoreCatalog>> findSimilarComparison(ProductSearchCriteria criteria)`
    - `void saveComparison(ProductSearchCriteria criteria, List<StoreCatalog> catalogs)`
    - `List<String> findAllCachedKeys()`
  - `StoreRepositoryPort`:
    - `List<StoreConfig> findActiveStores(Optional<String> city)`
    - `StoreConfig save(StoreConfig storeConfig)`
    - `Optional<StoreConfig> findById(String storeId)`
    - `void deleteById(String storeId)`
  - `WebScraperPort`:
    - `String scrapeWebPage(String targetUrl)`
  - `AiAgentPort`:
    - `List<StoreCatalog> executeComparisonAgent(ProductSearchCriteria criteria)`
- [ ] **3.5. Definición de Puertos de Entrada (Input Ports - Use Case Interfaces)**
  - `GetProductPricesUseCase`: `List<StoreCatalog> getPrices(ProductSearchCriteria criteria)`
  - `ManageStoresUseCase`
  - `SyncProductPricesUseCase`
- [ ] **3.6. Auditoría de código de dominio**
  - Confirmar ausencia total de comentarios en código.

---

## Fase 4: Capa de Aplicación (Use Cases & Orquestación con Agente de IA)

- [ ] **4.1. Estructura de paquetes de aplicación**
  - Crear paquete `com.pricecomparison.application.service`.
- [ ] **4.2. Implementación de `GetProductPricesService` (Use Case principal)**
  - Validar y normalizar el criterio de entrada (`ProductSearchCriteria`).
  - Consultar `ProductCachePort.findSimilarComparison(criteria)`.
  - **Manejo de Cache Hit:** Retornar inmediatamente los datos cacheados en Redis.
  - **Manejo de Cache Miss:**
    1. Delegar en el Agente de IA (`AiAgentPort.executeComparisonAgent(criteria)`).
    2. El agente orquesta autónomamente la consulta de tiendas, formulación de URLs de búsqueda, llamadas a la herramienta de scraping y extracción estructurada de productos.
    3. Retornar los catálogos agregados.
    4. Disparar persistencia asíncrona (`@Async`) hacia Redis.
- [ ] **4.3. Implementación de `ManageStoresService`**
  - Implementar CRUD de tiendas configuradas en Redis.
- [ ] **4.4. Implementación de `SyncProductPricesService`**
  - Iterar sobre las llaves de cache activas en Redis, ejecutar el agente de IA para refrescar precios de tiendas y actualizar registros en Redis atómicamente.

---

## Fase 5: Infraestructura - Persistencia, Cache Compuesto y Fuzzy Matching en Redis

- [ ] **5.1. Configuración de conexión y serialización Redis**
  - Configurar `RedisTemplate` con serialización JSON para `StoreCatalog`.
  - Soporte para búsqueda fuzzy / Levenshtein distance sobre llaves compuestas normalizadas.
- [ ] **5.2. Implementación de `RedisProductCacheAdapter` (implementa `ProductCachePort`)**
  - Implementar consulta con tolerancia a typos usando coincidencia difusa en Redis.
  - Persistencia con TTL configurable (ej. 24 horas).
- [ ] **5.3. Implementación asíncrona de persistencia (`@Async`)**
  - Configurar pool de hilos dedicado para persistencia en segundo plano sin penalizar la latencia HTTP.
- [ ] **5.4. Implementación de `RedisStoreRepositoryAdapter` (implementa `StoreRepositoryPort`)**
  - Almacenar tiendas (`id`, `name`, `baseUrl`, `supportedCities`, `active`) en Redis Hash.

---

## Fase 6: Infraestructura - Adaptador de Web Scraping Resiliente (Resilience4j)

- [ ] **6.1. Implementación de `HttpWebScraperAdapter` (implementa `WebScraperPort`)**
  - Ejecutar peticiones HTTP sobre la URL construida por el Agente de IA usando Jsoup o `RestClient`.
  - Configuración de User-Agent, timeouts de conexión y lectura.
  - Limpieza de tags irrelevantes (scripts, styles, SVGs) para devolver texto y estructura HTML limpia que optimice tokens del LLM.
- [ ] **6.2. Configuración de Resilience4j para Scraping**
  - Proteger llamadas HTTP con `@CircuitBreaker`, `@Retry` y `@TimeLimiter`.
- [ ] **6.3. Manejo de Fallos Parciales**
  - Si una tienda falla (HTTP 5xx, timeout, CAPTCHA), registrar log estructurado en nivel `ERROR` con la URL construida por el agente y retornar string de error controlado para que el agente continúe con las tiendas restantes.

---

## Fase 7: Infraestructura - Agente Autónomo de IA: Construcción de URLs y Extracción Semántica

- [ ] **7.1. Configuración de Spring AI Client**
  - Configurar `ChatClient` con modelo de lenguaje y temperatura baja (0.0) para máxima precisión y determinismo.
- [ ] **7.2. Implementación de Herramientas de Function Calling del Agente**
  - Implementar e inyectar las 4 Tools accesibles para el agente:
    1. `checkProductInRedis(String normalizedProductName, String brand, String city)`
    2. `fetchRegisteredStoresFromRedis(String city)`
    3. `scrapeWebPage(String targetUrl)`
    4. `saveProductToRedisAsync(String cacheKey, List<StoreCatalogDto> data)`
- [ ] **7.3. Configuración del System Prompt del Agente (Autonomía de URLs y Scraping)**
  - Instruir al agente para que:
    - Analice el `baseUrl` y nombre de cada tienda registrada.
    - **Formule de forma inteligente la URL de búsqueda específica** para esa tienda usando el nombre de producto, marca y ciudad.
    - Invoque la herramienta `scrapeWebPage(targetUrl)` con la URL formulada.
    - **Analice la respuesta HTML/texto**: identifique los productos reales, filtre por la marca indicada, distinga precios en USD y en VES, y seleccione los productos más relevantes hasta el `limit`.
    - Estructure el resultado final estrictamente bajo el esquema `StoreCatalogDto` (con `name`, `brand`, `priceUSD`, `priceVES`, **sin `productUrl`**).
- [ ] **7.4. Implementación de Guardrails de Seguridad (Anti-Prompt Injection)**
  - **System Prompt Lock:** Fijar el propósito exclusivo a la búsqueda y comparación de precios.
  - **Jailbreak Prevention:** Regla inmutable de ignorar instrucciones de usuario que intenten alterar las directivas del agente.
  - **Scope Restriction:** Si el contenido scrapeado o la consulta no representan un producto comercial válido, abortar sin conversación extra ni invocación innecesaria de tools.
- [ ] **7.5. Enforzamiento de Salida Estructurada**
  - Configurar `BeanOutputConverter` para deserializar la respuesta del LLM directamente en `List<StoreCatalogDto>`.

---

## Fase 8: Infraestructura - Adaptadores Web REST (OpenAPI Controllers) y Compresión HTTP

- [ ] **8.1. Implementación de `ProductApiController`**
  - Implementar la interfaz generada por OpenAPI para `GET /api/v1/products/prices`.
  - Mapear query parameters (`productName`, `brand`, `city`, `limit`) a `ProductSearchCriteria`.
  - Delegar en `GetProductPricesUseCase` y retornar HTTP 200 con la estructura agrupada por tienda.
- [ ] **8.2. Implementación de `AdminStoresApiController`**
  - Implementar endpoints administrativos para CRUD de tiendas (`POST`, `PUT`, `DELETE`, `GET /api/v1/admin/stores`).
- [ ] **8.3. Manejador Global de Excepciones (`@RestControllerAdvice`)**
  - Mapear `ConstraintViolationException` y validaciones regex a HTTP 400.
  - Mapear `InvalidProductSearchException` a HTTP 400.
  - Mapear `StoreNotFoundException` a HTTP 404.
  - Mapear accesos denegados a HTTP 403 y falta de credenciales a HTTP 401.

---

## Fase 9: Infraestructura - Seguridad, RBAC y Anti-DDoS (Bucket4j)

- [ ] **9.1. Configuración de Spring Security Stateless**
  - Configurar `SecurityFilterChain` sin estado (`SessionCreationPolicy.STATELESS`).
  - Deshabilitar CSRF.
  - Configurar cabeceras de seguridad estrictas (CORS, CSP, `X-Content-Type-Options: nosniff`).
- [ ] **9.2. Autenticación JWT y RBAC**
  - Configurar `OAuth2ResourceServer` / JWT.
  - Restringir `/api/v1/products/**` a `ROLE_USER` y `ROLE_ADMIN`.
  - Restringir `/api/v1/admin/**` exclusivamente a `ROLE_ADMIN`.
- [ ] **9.3. Rate Limiting con Bucket4j respaldado por Redis**
  - Configurar límite distribuido en Redis (20 peticiones/minuto por token/IP).
  - Retornar HTTP 429 Too Many Requests cuando se sobrepase el bucket.

---

## Fase 10: Infraestructura - Sincronización Programada de Precios (@Scheduled)

- [ ] **10.1. Habilitación de tareas programadas**
  - `@EnableScheduling` en la configuración general.
- [ ] **10.2. Implementación de `PriceSynchronizationScheduler`**
  - Configurar `@Scheduled(cron = "0 0 6 * * *", zone = "America/Caracas")`.
  - Ejecución diaria a las 06:00 AM (hora Caracas).
  - Invocar al agente de IA para re-evaluar tiendas y actualizar registros en Redis de manera atómica.

---

## Fase 11: Infraestructura - Observabilidad, Logging Estructurado y Actuator

- [ ] **11.1. Configuración de Logging Estructurado JSON**
  - Configurar `logback-spring.xml` para logs JSON con SLF4J.
- [ ] **11.2. Correlation ID con MDC**
  - Inyectar `X-Correlation-ID` en el MDC y propagarlo a `@Async`, hilos del agente y tareas programadas.
- [ ] **11.3. Configuración de Spring Boot Actuator**
  - Exponer `/actuator/health` con estados de Redis y Circuit Breakers de Resilience4j.

---

## Fase 12: Suite de Pruebas Unitarias y de Integración (QA & Verification)

- [ ] **12.1. Pruebas Unitarias (Cobertura >85% con JUnit 5 y Mockito)**
  - [ ] Test de `ProductSearchCriteria`: validación regex de `productName`, `brand`, `city`, y generación de llave compuesta.
  - [ ] Test de `GetProductPricesService`: lógica de cache hit y delegación a agente.
  - [ ] Test de Guardrails y prompts del Agente de IA (verificar que no incluya `productUrl` en el output).
- [ ] **12.2. Pruebas de Integración con Testcontainers y WireMock**
  - [ ] Contenedor real de Redis (`RedisContainer`).
  - [ ] WireMock simulando páginas de búsqueda de tiendas y respuestas del LLM con tool-calling.
  - [ ] **Escenario 1 (Cache Hit exacto y fuzzy):** Búsqueda con typos (`"champoo"`, `"pantene"`, `"caracas"` $\to$ resuelve a `"shampoo"`).
  - [ ] **Escenario 2 (Cache Miss con Agente Autónomo):** El agente formula URLs, invoca scraping en WireMock, analiza el HTML, extrae `name`, `brand`, `priceUSD`, `priceVES` (sin `productUrl`), responde al cliente y persiste en Redis.
  - [ ] **Escenario 3 (Resiliencia ante fallo parcial):** 1 tienda falla (HTTP 500 / timeout) y 2 responden con éxito; verificar que el agente devuelva datos de las tiendas saludables y se registre log de error.
  - [ ] **Escenario 4 (Seguridad y Rate Limiting):** Validación de regex (400), token inválido (401), falta de rol admin (403), y ráfaga de >20 peticiones (429).
  - [ ] **Escenario 5 (Job Programado):** Ejecución de la tarea programada y actualización de Redis.
- [ ] **12.3. Verificación de Regla "NO COMMENTS"**
  - Verificación estricta de que ningún archivo fuente de `src/` contenga comentarios o Javadocs.

---

## Fase 13: Despliegue Local, Docker Compose y Verificación Final

- [ ] **13.1. `Dockerfile` multi-stage con Eclipse Temurin 21**
- [ ] **13.2. `docker-compose.yml` completo con Redis Stack y el servicio**
- [ ] **13.3. Documentación en `README.md`**
  - Parámetros de búsqueda (`productName`, `brand`, `city`, `limit`), estructura enriquecida de respuesta sin `productUrl`, comandos de ejecución y tests (`./mvnw clean verify`).
