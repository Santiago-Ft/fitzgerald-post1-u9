# Recordatorios Citas Service

![CI](https://github.com/Santiago-Ft/fitzgerald-post1-u9/actions/workflows/ci.yml/badge.svg)

## Descripción
Repositorio del post-contenido de la Unidad 9 de Patrones de Diseño de Software. Proyecto Spring Boot que gestiona y envía recordatorios de citas médicas a través del canal de preferencia del paciente (Email o SMS), estructurado bajo los patrones **Strategy** (`ReminderChannel`) y **Factory Method** (`ReminderChannelFactory`) aplicando el Principio de Inversión de Dependencias (DIP) mediante inyección por constructor.

## Parte 1 — Pruebas Unitarias con JUnit 5 y Mockito
La suite `CitaReminderServiceImplTest` evalúa en total aislamiento la lógica de negocio de `CitaReminderServiceImpl`, sustituyendo sus colaboradores por Test Doubles seleccionados según el rol que cumple cada dependencia en la prueba.

## Parte 2 — Refactor DIP, Integración y CI/CD
Se incorporó la funcionalidad `enviarRecordatorioUrgente(Long citaId)`. Su versión inicial instanciaba directamente sus dependencias con `new`, impidiendo simular fallos de infraestructura. Mediante refactorización se trasladó la creación a `config/CanalUrgenteConfig.java` y se inyectó la abstracción por constructor. Adicionalmente, se implementaron pruebas de persistencia con `@DataJpaTest` (`CitaRepositoryTest`), pruebas de contexto completo con `@SpringBootTest` (`CitaReminderServiceIntegrationTest`) sobre H2 en memoria, y un flujo de Integración Continua en `.github/workflows/ci.yml`.

## Decisiones de diseño

* **Por qué Strategy facilitó las pruebas:**  
  Al encapsular cada mecanismo de notificación detrás de la interfaz común `ReminderChannel`, el servicio desconoce los detalles técnicos de transporte (clientes SMTP o llamadas HTTP a APIs de mensajería). Sin este patrón, el código de red estaría acoplado dentro del servicio y las pruebas unitarias exigirían interceptar librerías externas de bajo nivel. Gracias a Strategy, basta con inyectar un único `@Mock ReminderChannel` para verificar de forma uniforme el despacho sobre cualquier canal.

* **Por qué Factory Method facilitó las pruebas:**  
  Delegar la elección del canal en `ReminderChannelFactory` libera al servicio de estructuras condicionales (`if/else` o `switch`) acopladas a las implementaciones concretas. Sin la fábrica inyectada, comprobar que se eligió el medio correcto solo podría hacerse de forma indirecta inspeccionando efectos colaterales. Con Factory Method, la decisión se vuelve una interacción explícita y verificable mediante `verify(reminderChannelFactory).obtenerCanal(CanalNotificacion.EMAIL)`.

* **Elección justificada de Mock vs. Stub vs. Fake (Parte 1):**
  * **`CitaRepository` (Mock):** El servicio no solo consulta citas pendientes (entrada indirecta), sino que muta su estado a `recordatorioEnviado = true` y ordena persistirlas con `save(cita)` (salida indirecta). Un simple Stub solo devolvería datos pero no permitiría comprobar si el guardado ocurrió. Se eligió un **Mock** porque es indispensable verificar la interacción saliente (`verify(citaRepository, times(1)).save(cita)`).
  * **`ReminderChannelFactory` y `ReminderChannel` (Mock):** El método `enviar()` es un comando `void` cuyo propósito es producir un efecto en un colaborador externo. Al no existir un valor de retorno que validar por estado, la prueba requiere verificar el comportamiento y el paso de mensajes entre objetos (`verify`), razón por la cual ambos se modelan como **Mocks**.
  * **`Clock` (Fake — `Clock.fixed`):** En lugar de mockear `java.time.Clock` con Mockito —lo cual obligaría a configurar llamadas internas como `instant()` y `getZone()` con el riesgo de alterar el funcionamiento de `LocalDateTime.now(clock)`—, se utilizó `Clock.fixed(...)`. Esta es una implementación real, determinista y completa provista por la librería estándar de Java que congela el tiempo y garantiza que el cálculo de la ventana de 24 horas sea exacto y reproducible en cualquier máquina.

* **Diagnóstico y Refactor DIP en `enviarRecordatorioUrgente` (Parte 2):**  
  * **Antes del refactor:** El método contenía la instrucción `ReminderChannel canalUrgente = new SmsReminderChannel(new TwilioNotificadorSmsGateway());` en su interior. Este acoplamiento duro violaba el Principio de Inversión de Dependencias (DIP) e impedía sustituir el canal por un doble de prueba, haciendo imposible verificar qué ocurría si el proveedor de SMS lanzaba una excepción.
  * **Después del refactor:** Se extrajo la instanciación hacia el *Composition Root* en `CanalUrgenteConfig` (`@Bean canalUrgente`) y se inyectó la abstracción `ReminderChannel` por constructor usando `@Qualifier("canalUrgente")`. Esto habilitó el punto de sustitución necesario para escribir la prueba `enviarRecordatorioUrgente_gatewayFalla_noMarcaComoEnviado()`, demostrando que al fallar el envío se propaga la excepción y nunca se invoca `citaRepository.save()`.

* **H2 en memoria junto con `@MockBean` en Integración y CI:**  
  En `CitaReminderServiceIntegrationTest` conviven la base de datos H2 real en memoria y `@MockBean NotificadorSmsGateway`. H2 actúa como un Fake fiel para validar el mapeo JPA y las consultas SQL sin infraestructura externa (lo que además simplifica el pipeline de GitHub Actions al no requerir contenedores Docker adicionales), mientras que el gateway de SMS se mockea por tratarse de un servicio de terceros no determinista.

## Ejecutar las pruebas

```bash
mvn test                  # Pruebas unitarias
mvn verify                # Unitarias + integración
```

## Herramientas utilizadas
- Java 21, Spring Boot 3.3, Spring Data JPA, H2, JUnit 5, Mockito
- Apache Maven, GitHub Actions, Git, GitHub

## Conclusiones
En este laboratorio se evidencio la testabilidad del sistema en base a su arquitectura y utilizando el principio de DIP, adicionalmente el uso de patrones de diseño para poder aislar las mecanicas de negocios y realizar pruebas. Tambien cabe recalcar los pasos de refactorizacion de la Parte 2 donde se elimino el uso de new para proteger los datos de una mejor manera y utilizando un pipeline en GitHub Actions para garantizar las pruebas en el Spring Boot.

