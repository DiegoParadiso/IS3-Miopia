# ADR-008: Implementación del Patrón Circuit Breaker para Resiliencia ante Fallos de APIs Externas

**Estado:** Aceptado  
**Fecha:** 2026-06-17  
**Decisores:** Amarilla Aldana Abigail Itatí, González Diego Agustín, Ibañez Adrian Jose María, Machado Victor Leandro
**Relacionado:** Issue #16

---

## Contexto

El sistema de gestión de eventos depende de servicios externos para funcionalidades críticas. El caso más inmediato es el envío de correos electrónicos vía SMTP (Módulo 7: Notificaciones y Recordatorios), que utiliza Nodemailer para comunicarse con un servidor de correo externo. A futuro, el sistema podría integrar pasarelas de pago, APIs de mensajería SMS, servicios de geocodificación o proveedores de almacenamiento en la nube.

**Problemas identificados en el escenario actual:**

- **SMTP como punto único de falla:** Si el servidor SMTP se vuelve lento o no responde, el `notification-service` que invoca `sendEmail()` quedaría bloqueado esperando la respuesta. En el diseño actual con cola BullMQ (ADR-004), el worker reintenta hasta 3 veces con backoff exponencial, pero durante esos reintentos el worker consume recursos y retrasa otros trabajos encolados.

- **Efecto cascada:** Si el servicio SMTP está caído, todos los workers de email fallan secuencialmente. BullMQ reintenta cada trabajo individualmente, pero la presión sobre el worker se mantiene, y el sistema sigue intentando conexiones que probablemente fallarán, desperdiciando recursos.

- **Tiempos de espera prolongados:** Las operaciones de red contra servicios externos pueden exceder el tiempo de espera del worker, manteniendo ocupados los hilos del proceso de Node.js y degradando la capacidad de procesamiento de toda la cola.

- **Futuras integraciones:** A medida que el sistema evolucione, es probable que se integren más servicios externos. Sin un patrón de resiliencia uniforme, cada integración requeriría su propia lógica de manejo de fallos, llevando a código duplicado y comportamiento inconsistente.

**Restricciones que aplican:**

- El sistema utiliza Node.js con un modelo de concurrencia basado en un solo hilo de event-loop; las operaciones bloqueantes o con timeouts largos afectan la capacidad de respuesta global.
- La cola BullMQ (ADR-004) ya proporciona reintentos, pero no protege contra llamadas continuas a un servicio que se sabe que está caído.
- El equipo necesita una solución liviana que no introduzca dependencias pesadas ni requiera infraestructura adicional.
- La solución debe ser configurable por servicio externo, permitiendo distintos umbrales de fallo y tiempos de recuperación según la criticidad de cada integración.
- No se dispone de un service mesh o proxy sidecar (como Istio o Linkerd); la lógica de resiliencia debe implementarse a nivel de aplicación.

---

## Decisión

Se implementa el **patrón Circuit Breaker** utilizando la librería **Opossum** (`opossum ^6.4.0`) para envolver todas las llamadas a servicios externos del sistema.

**Alcance:**

- Cubre el envío de correos electrónicos vía SMTP/Nodemailer en el módulo de notificaciones (M7).
- Cubre cualquier futura integración con APIs externas (pasarelas de pago, SMS, almacenamiento externo, etc.).
- Cubre la configuración de umbrales de fallo, tiempos de recuperación y estrategias de timeout por servicio.
- No cubre la resiliencia de llamadas a la base de datos PostgreSQL (gestionado por Sequelize con pool de conexiones y reintentos controlados).
- No cubre la comunicación entre instancias de la propia aplicación (comunicación interna síncrona no existe en la arquitectura monolítica actual).

**Comportamiento del Circuit Breaker:**

```
         ┌──────────┐
         │  CERRADO  │
         │  (CLOSED) │
         └─────┬─────┘
               │ fallos > umbral
               ▼
         ┌──────────┐
         │  ABIERTO  │
         │  (OPEN)   │ ←── solicitudes rechazadas inmediatamente
         └─────┬─────┘
               │ timeout de recuperación
               ▼
         ┌───────────┐
         │ SEMI-ABIERTO│
         │(HALF-OPEN) │ ←── 1 solicitud de prueba
         └─────┬──────┘
             / \
        éxito   fallo
        /           \
       ▼             ▼
   ┌──────────┐  ┌──────────┐
   │ CERRADO  │  │ ABIERTO  │
   └──────────┘  └──────────┘
```

**Configuración propuesta para cada breaker:**

```javascript
const circuitOptions = {
  timeout: 10000,           // 10 segundos de timeout por llamada
  errorThresholdPercentage: 50,  // Abrir tras 50% de fallos
  resetTimeout: 30000,      // 30 segundos hasta half-open
  volumeThreshold: 5,       // Mínimo 5 llamadas para evaluar
  name: 'smtp-service'      // Nombre identificatorio
};
```

**Integración con el worker de BullMQ (ADR-004):**

```
Trabajo encolado → Worker BullMQ → Circuit Breaker → SMTP/Nodemailer
                                        │
                                   si OPEN →
                                        │
                                   Rechazo inmediato → BullMQ retry con backoff
                                        │
                                   El breaker registra el fallo pero NO
                                   realiza la llamada real al SMTP
```

Cuando el breaker está en estado ABIERTO, el worker de BullMQ falla rápido (fail-fast) sin consumir el recurso SMTP. BullMQ reintenta según su política de retry (3 reintentos con backoff exponencial), pero el breaker previene que esos reintentos lleguen al SMTP mientras esté caído. Cuando el breaker pasa a SEMI-ABIERTO, permite un intento de prueba; si tiene éxito, el breaker se cierra y BullMQ puede procesar los trabajos normalmente.

---

## Alternativas consideradas

### Opción A: Opossum (librería Circuit Breaker para Node.js)
- ✅ Librería madura y liviana (~5 KB), diseñada específicamente para Node.js y su modelo de event-loop.
- ✅ API basada en Promesas, compatible con async/await y con el flujo BullMQ.
- ✅ Soporte nativo de timeout por llamada, umbral de fallos porcentual, volumen mínimo de muestras y timeout de recuperación.
- ✅ Eventos emitidos en cada cambio de estado (`open`, `close`, `halfOpen`) que permiten logging y alertas.
- ✅ Permite envolver funciones existentes sin modificar su implementación interna.
- ❌ No incluye dashboards ni UI de monitoreo (debe implementarse aparte con los eventos emitidos).

### Opción B: Implementación manual del patrón sin librería
- ✅ Control total sobre la lógica y el comportamiento.
- ✅ Sin dependencias externas adicionales.
- ❌ Riesgo de errores en la implementación de la máquina de estados (transiciones, condiciones de carrera, consistencia en entornos multi-proceso).
- ❌ Mayor tiempo de desarrollo y pruebas.
- ❌ Mantenimiento y evolución del patrón recae completamente en el equipo.

### Opción C: @acoboy/opossum (fork) u otras alternativas (cockatiel, brake)
- **cockatiel:** ✅ Bien diseñada con TypeScript y políticas de retry y circuit breaker combinables. ❌ Menos adoptada en proyectos JavaScript puro; documentación principalmente orientada a TypeScript.
- **brake:** ✅ Liviana y simple. ❌ Menos configurable (sin soporte de volumen mínimo de muestras ni timeout de recuperación configurable).
- **built-in de BullMQ (retry):** ❌ Los reintentos de BullMQ (ADR-004) reintentan el trabajo completo, no previenen llamadas fallidas al SMTP. El circuit breaker complementa a BullMQ, no lo reemplaza.

Se elige **Opossum** por su madurez, simplicidad, compatibilidad con JavaScript/Node.js y soporte explícito para todos los requisitos del patrón.

---

## Consecuencias

### Positivas

1. **Protección contra fallos en cascada:** Cuando un servicio externo (SMTP, pasarela de pago, etc.) está degradado o caído, el Circuit Breaker pasa a estado ABIERTO y rechaza las llamadas de forma inmediata (fail-fast). Esto evita que el sistema consuma tiempo de CPU, conexiones de red y hilos del worker en operaciones que probablemente fallarán, protegiendo los recursos del sistema para las operaciones que sí pueden completarse exitosamente.

2. **Tiempo de recuperación acelerado:** Al no bombardear al servicio externo con solicitudes mientras está caído, se le da oportunidad de recuperarse sin carga adicional. Una vez que el breaker detecta recuperación (estado SEMI-ABIERTO con una solicitud de prueba exitosa), reanuda el tráfico normal de forma controlada, evitando el efecto "thundering herd" cuando el servicio vuelve a estar disponible.

3. **Visibilidad y monitoreo del estado de integraciones externas:** Opossum emite eventos (`open`, `close`, `halfOpen`, `success`, `failure`, `timeout`) que pueden ser escuchados para registrar métricas, generar alertas y alimentar dashboards de monitoreo. Esto proporciona visibilidad en tiempo real sobre la salud de cada integración externa, permitiendo al equipo operador detectar problemas antes de que afecten a los usuarios.

4. **Configuración granular por servicio:** Cada servicio externo puede tener su propia configuración de Circuit Breaker (timeout, umbral de fallos, tiempo de recuperación). Por ejemplo, el envío de emails (menos crítico) puede tener un timeout de 10 segundos y un umbral del 50%, mientras que una futura pasarela de pago (crítica) podría tener un timeout más agresivo y un umbral más bajo para detectar fallos rápidamente.

### Negativas

1. **Complejidad adicional en el flujo de workers BullMQ:** La integración del Circuit Breaker con BullMQ añade una capa de lógica que debe diseñarse cuidadosamente. Es necesario decidir si los trabajos que fallan por breaker abierto deben reintentarse (BullMQ retry) o enviarse a una cola de "dead letter" para revisión manual. Si se configuran mal los reintentos, los trabajos podrían acumularse en la cola sin progresar, generando una falsa sensación de procesamiento.

2. **Posibles falsos positivos en umbrales mal configurados:** Si el `volumeThreshold` es demasiado bajo o el `errorThresholdPercentage` demasiado sensible, el breaker puede abrirse por picos transitorios de latencia (un timeout puntual), interrumpiendo el servicio innecesariamente. Esto requiere ajuste fino de los parámetros durante las pruebas de integración y monitoreo continuo en producción para calibrar los valores óptimos por servicio.

3. **Estado del breaker en memoria, no persistido:** El estado del Circuit Breaker (abierto/cerrado/semi-abierto) se mantiene en memoria dentro del proceso de Node.js. Si el worker se reinicia (por deploy, crash o escalado), el breaker arranca en estado CERRADO, permitiendo tráfico hacia un servicio que podría seguir caído. Para mitigarlo, se puede implementar una estrategia de "enfriamiento" al arranque: retrasar las primeras llamadas o arrancar en estado ABIERTO temporal. Sin embargo, esta mitigación añade complejidad adicional.

---

## Plan de implementación

1. Instalar la dependencia:
   ```bash
   npm install opossum ^6.4.0
   ```

2. Crear `src/utils/circuit-breaker.js` con una fábrica de breakers configurable:
   ```javascript
   const CircuitBreaker = require('opossum');

   function createServiceBreaker(fn, options = {}) {
     const defaults = {
       timeout: 10000,
       errorThresholdPercentage: 50,
       resetTimeout: 30000,
       volumeThreshold: 5,
       name: 'unnamed-service'
     };

     const breaker = new CircuitBreaker(fn, { ...defaults, ...options });

     breaker.on('open', () => console.warn(`[CB] ${breaker.name} — ABIERTO`));
     breaker.on('halfOpen', () => console.info(`[CB] ${breaker.name} — SEMI-ABIERTO`));
     breaker.on('close', () => console.info(`[CB] ${breaker.name} — CERRADO`));

     return breaker;
   }

   module.exports = { createServiceBreaker };
   ```

3. Envolver la función `sendEmail` del `EmailService` con un breaker:
   ```javascript
   const { createServiceBreaker } = require('../utils/circuit-breaker');
   const { sendEmailRaw } = require('./email-provider');

   const emailBreaker = createServiceBreaker(sendEmailRaw, {
     name: 'smtp-service',
     timeout: 8000,
     errorThresholdPercentage: 40
   });

   async function sendEmail(to, subject, body) {
     return emailBreaker.fire(to, subject, body);
   }
   ```

4. Integrar con el worker de BullMQ (ADR-004):
   ```javascript
   const emailQueue = require('../queues/email-queue');

   emailQueue.process(async (job) => {
     const { to, subject, body } = job.data;
     try {
       await sendEmail(to, subject, body);
       return { sent: true };
     } catch (err) {
       if (err.code === 'EOPENBREAKER') {
         // Breaker abierto: el trabajo se reintentará según política BullMQ
         throw err; // BullMQ maneja el retry automáticamente
       }
       throw err;
     }
   });
   ```

5. Configurar logging de eventos del breaker para monitoreo:
   ```javascript
   breaker.on('open', () => {
     logger.warn({ service: breaker.name }, 'Circuit Breaker OPEN — servicio no disponible');
   });
   ```

**Dependencias:** `opossum ^6.4.0`, Node.js 18+.

**Métrica de éxito:**
- Cuando el servicio SMTP se simula como caído (en pruebas de integración), el breaker abre después de N fallos y las llamadas fallan inmediatamente sin consumir tiempo de red.
- Cuando el servicio se restablece, el breaker detecta la recuperación en el primer intento en estado SEMI-ABIERTO y vuelve a estado CERRADO.
- El worker BullMQ no se bloquea ni acumula backpressure cuando el breaker está abierto.

---

## Triggers de revisión

- Si el sistema evoluciona hacia una arquitectura de microservicios (evaluar si el Circuit Breaker debe moverse a un API Gateway o sidecar proxy como Envoy).
- Si se requiere persistencia del estado del breaker entre reinicios (evaluar almacenamiento en Redis del estado del breaker, compartiendo la misma instancia de Redis usada por BullMQ).
- Si el número de servicios externos supera los 5 y se necesita una UI de administración (evaluar dashboards con los eventos emitidos vía WebSocket o logs estructurados hacia un sistema de monitoreo como Grafana).
- Si el equipo decide migrar a TypeScript (evaluar `cockatiel` como alternativa con tipado más fuerte y APIs más modernas).

**Fecha sugerida de revisión:** 2027-06-01
