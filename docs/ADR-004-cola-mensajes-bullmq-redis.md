# ADR-004 — Implementación de Cola de Mensajes (BullMQ + Redis) para el Envío Asíncrono de Notificaciones
 
**Estado:** Aceptado  
**Fecha:** 2026-06-17  
**Decisores:** Amarilla Aldana Abigail Itatí, González Diego Agustín, Ibañez Adrian Jose María, Machado Victor Leandro  
**Relacionado:** Issue #14 · spec_m7.md · src/services/notification-service.js · src/workers/email-worker.js · config/redis.js
 
---
 
## Contexto
 
El módulo de **Notificaciones y Recordatorios (M7)** implementa la generación y envío de notificaciones tanto internas (in-app) como externas a través de correo electrónico (utilizando Nodemailer con SMTP). El envío de correos electrónicos a través de proveedores externos es una operación que introduce alta latencia (de 1 a 3 segundos por envío) y es susceptible a fallos de red temporales o límites de tasa (rate limits) del servidor SMTP.
 
**Problemas identificados:**
- **Latencia y bloqueo del hilo principal:** Enviar notificaciones masivas de forma síncrona (por ejemplo, ante la cancelación de un evento con 1,000 participantes inscritos) bloquearía el servidor Express por varios minutos, provocando timeouts en los clientes y degradando severamente la experiencia de usuario.
- **Falta de tolerancia a fallos:** Si el servidor SMTP externo experimenta una caída temporal o un rechazo por exceso de envíos, las notificaciones de correo fallarían de forma definitiva sin posibilidad de reintento automático.
- **Escalabilidad de la aplicación:** La generación y el envío de notificaciones deben estar desacoplados del flujo principal de solicitudes HTTP para permitir escalar el procesamiento de emails independientemente del servidor web.
 
**Restricciones que aplicaban:**
- El procesamiento en segundo plano no debe afectar la consistencia de las notificaciones in-app persistidas en PostgreSQL.
- Se requiere control de concurrencia y tasa de envío para no saturar el servidor SMTP.
- La infraestructura de colas debe ser lo suficientemente ligera para ejecutarse de manera local y en entornos académicos.
 
---
 
## Decisión
 
Se adopta **BullMQ** sobre **Redis** como el sistema de colas de tareas (message queue) y procesamiento en segundo plano para gestionar el envío asíncrono de notificaciones y correos electrónicos.
 
**Alcance:**
- Todas las peticiones de envío de correos electrónicos en `NotificationService` se encolarán en una cola dedicada (`email-queue`) administrada por BullMQ.
- Un proceso worker dedicado (`src/workers/email-worker.js`) consumirá las tareas de la cola de forma asíncrona.
- Se configurará una política de reintentos automáticos con backoff exponencial (ej: reintentar hasta 3 veces con intervalos crecientes) en caso de fallos SMTP.
- La persistencia de las notificaciones in-app en PostgreSQL se seguirá ejecutando de manera síncrona dentro de `NotificationService.notify` antes de encolar el email, garantizando la consistencia inmediata del estado in-app.
 
---
 
## Alternativas consideradas
 
**Opción A: BullMQ + Redis [Adoptada]**
- ✅ **Alto rendimiento:** Redis opera en memoria, lo que permite un encolamiento extremadamente rápido (<1ms).
- ✅ **Robustez y características:** BullMQ ofrece soporte nativo para reintentos automáticos, retrasos de tareas (delays), prioridad y limitación de tasa (rate limiting).
- ✅ **Ecosistema:** Librería madura, muy activa y bien integrada con Node.js y patrones de programación asíncrona.
- ❌ **Dependencia adicional:** Requiere instalar y mantener un servidor Redis en los entornos de desarrollo y producción.
 
**Opción B: RabbitMQ**
- ✅ Estándar de la industria para arquitecturas de microservicios complejas y mensajería empresarial.
- ✅ Gran flexibilidad en el enrutamiento de mensajes (exchanges, bindings).
- ❌ **Sobredimensión:** Demasiado complejo para las necesidades actuales del proyecto (mensajería punto a punto simple).
- ❌ **Curva de aprendizaje:** Requiere mayor configuración y administración de infraestructura comparado con Redis.
 
**Opción C: Procesamiento asíncrono simple en memoria (`setImmediate` o Promises flotantes)**
- ✅ Sin dependencias de infraestructura de terceros (no requiere Redis).
- ❌ **Pérdida de datos:** Si el proceso de la aplicación Node.js se reinicia o se cae, todas las tareas de email encoladas en memoria se perderán definitivamente.
- ❌ **Falta de control:** No posee mecanismos integrados para gestionar reintentos, limitar la tasa de envío ni monitorear el estado de los mensajes pendientes.
 
---
 
## Consecuencias
 
**Beneficios esperados:**
- **Respuestas inmediatas:** Los endpoints que gatillan notificaciones (como registro a eventos, acreditación, feedback) responden al instante al usuario, mejorando los tiempos de respuesta de la API a < 50ms.
- **Tolerancia a fallos:** Los correos que fallen por problemas temporales del SMTP se reintentarán automáticamente sin intervención manual y sin interrumpir otros servicios.
- **Desacoplamiento:** El flujo web y el flujo de envío de correos quedan completamente desacoplados. Los workers pueden correr en procesos o contenedores independientes si es necesario.
 
**Costos o riesgos que se aceptan:**
- Se introduce **Redis** como una dependencia de infraestructura obligatoria. El entorno de desarrollo local y producción debe proveer una instancia activa.
- Mayor complejidad en el monitoreo del sistema: ahora se debe supervisar el estado de Redis y de las colas de BullMQ (además de la base de datos PostgreSQL).
 
**Impacto en operación y equipo:**
- Se deben actualizar los archivos `package.json` para incluir `bullmq` e `ioredis`.
- El archivo `.env` debe incluir la URI de conexión de Redis (`REDIS_URL`).
- Docker Compose (M12) debe incluir un servicio de Redis.
 
---
 
## Plan de implementación
 
1. Agregar `bullmq ^4.12.0` e `ioredis ^5.3.0` a las dependencias del proyecto.
2. Crear la configuración de conexión de Redis en `src/config/redis.js`.
3. Definir la cola `emailQueue` y exportar un helper en `src/queues/email-queue.js` para añadir trabajos a la cola.
4. Crear el archivo `src/workers/email-worker.js` que inicialice un Worker de BullMQ para procesar los trabajos de `emailQueue`, llamando a la lógica de envío de Nodemailer.
5. Refactorizar el método `sendEmail` y `notify` en `src/services/notification-service.js` para que, en lugar de realizar el envío síncrono del correo, añada un trabajo a `emailQueue`.
6. Configurar el inicio de los workers en el punto de entrada de la aplicación (`src/app.js` o `src/server.js`).
 
**Dependencias:** Node.js 18+, Redis 7+, `bullmq ^4.12.0`, `ioredis ^5.3.0`, Nodemailer.
 
**Métrica de éxito:**
- El tiempo de respuesta de `POST /api/v1/events/:id/register` es inferior a 80ms promedio bajo carga.
- Los trabajos fallidos por error de SMTP se reintentan exitosamente al restaurar la conexión del servicio de correo.
- No se pierden trabajos encolados tras un reinicio controlado del servidor de aplicación Node.js.
 
---
 
## Triggers de revisión
 
- Si el volumen de tareas en Redis satura la memoria disponible del servidor Redis (evaluar políticas de desalojo de trabajos completados o escalado de Redis).
- Si la complejidad de la mensajería del sistema evoluciona hacia una arquitectura dirigida por eventos con múltiples tópicos y patrones publish-subscribe avanzados (evaluar migrar a RabbitMQ o Apache Kafka).
 
**Fecha sugerida de revisión:** 2026-12-30
