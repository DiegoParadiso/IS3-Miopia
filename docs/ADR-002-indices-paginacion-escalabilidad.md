# ADR-002 — Estrategia de indexación y paginación para escalabilidad de la base de datos

**Estado:** Aceptado  
**Fecha:** 2026-06-16  
**Decisores:** Amarilla Aldana Abigail Itatí, González Diego Agustín, Ibañez Adrian Jose María, Machado Victor Leandro  
**Relacionado:** Issue #13 · spec_m2.md · spec_m3.md · spec_m5.md · spec_m6.md · spec_m7.md · ADR-001 (PostgreSQL) 

---

## Contexto

A medida que el sistema opera en producción y los datos crecen, se identificaron **requerimientos derivados del crecimiento de la base de datos** que afectan la performance de consultas frecuentes. El análisis de las queries más ejecutadas revela dos problemas:

**Problema 1 — Consultas sin índices en columnas frecuentemente filtradas:**
Las tablas `event_attendees`, `notifications` y `feedback` no tienen índices en las columnas más usadas como filtro (`userId`, `eventId`, combinación `(eventId, userId)`). Con PostgreSQL, sin índices, estas queries realizan sequential scans que escalan O(n) con el tamaño de la tabla.

Proyección de crecimiento:
- `event_attendees`: 500 eventos × 200 participantes promedio = 100,000 filas en el primer año.
- `notifications`: cada usuario genera ~10 notificaciones por evento = potencialmente 1,000,000 de filas.
- `feedback`: hasta 200 registros por evento.

**Problema 2 — Endpoints sin paginación retornan colecciones completas:**
Los endpoints `GET /api/v1/events` (M2), `GET /api/v1/notifications` (M7) y `GET /api/v1/events/:id/attendees` (M2) actualmente retornan todos los registros en una sola respuesta. Con miles de registros, esto genera responses HTTP de varios MB, alto uso de memoria en el servidor y timeouts en clientes con conexiones lentas.

**Restricciones que aplican:**

- Los índices deben definirse sin alterar la lógica de negocio existente.
- La paginación debe ser consistente en toda la API (mismo esquema de query params y response shape).
- La solución debe implementarse en el ORM (Sequelize) sin requerir SQL raw en la mayoría de los casos.
- Los cambios de esquema (agregar índices) deben gestionarse vía migraciones para reproducibilidad.

**Datos del proyecto que sustentan la decisión:**

- `Notification.findAll({ where: { userId } })` en M7: sin índice en `userId`, hace full scan en tabla con potencial de >1M filas.
- `EventAttendee.findAll({ where: { eventId } })` en M3 y M6: usado en cada check-in, informe de asistencia y generación de certificados.
- `Feedback.findAll({ where: { eventId } })` en M5 y M6: usado en cálculo de métricas de satisfacción.
- `Event.findAll()` sin filtros en M2 puede retornar cientos de eventos con sus relaciones incluidas.

---

## Decisión

Se implementa una **estrategia combinada de indexación de columnas de alta frecuencia y paginación obligatoria** en todos los endpoints de listado del sistema.

### Parte 1: Índices a agregar

| Tabla | Columna(s) | Tipo | Justificación |
|-------|-----------|------|---------------|
| `event_attendees` | `(eventId)` | B-tree | Queries de asistentes por evento (M3, M6) |
| `event_attendees` | `(userId)` | B-tree | Queries de eventos del usuario (M8) |
| `event_attendees` | `(eventId, userId)` | B-tree único | Validación de elegibilidad M3 y M4 (ya existe constraint, formalizar como índice explícito) |
| `notifications` | `(userId)` | B-tree | Queries de notificaciones del usuario (M7) |
| `notifications` | `(userId, isRead)` | B-tree | Filtro de no leídas (M7, query más frecuente) |
| `notifications` | `(createdAt)` | B-tree | Ordenamiento descendente en listados |
| `feedback` | `(eventId)` | B-tree | Queries de feedback por evento (M5, M6) |
| `feedback` | `(eventId, userId)` | B-tree único | Validación de feedback duplicado (M5) |
| `events` | `(organizerId)` | B-tree | Queries de eventos del organizador (M8) |
| `events` | `(date, status)` | B-tree compuesto | Filtros por fecha y estado (M2 getAll, cron M7) |
| `talks` | `(eventId)` | B-tree | Queries de charlas por evento (M2, M6) |
| `talks` | `(speakerId)` | B-tree | Queries de charlas del disertante (M8) |

### Parte 2: Esquema de paginación estándar

**Query params para todos los endpoints de listado:**
```
page: integer (default: 1, min: 1)
limit: integer (default: 20, min: 1, max: 100)
```

**Shape de respuesta con paginación:**
```json
{
  "success": true,
  "data": {
    "items": [...],
    "pagination": {
      "total": 150,
      "page": 2,
      "limit": 20,
      "totalPages": 8,
      "hasNextPage": true,
      "hasPrevPage": true
    }
  }
}
```

**Implementación Sequelize:**
```javascript
const offset = (page - 1) * limit;
const { count, rows } = await Model.findAndCountAll({ where, limit, offset, order });
```

**Endpoints que deben implementar paginación:**
- `GET /api/v1/events` (M2)
- `GET /api/v1/events/:id/attendees` (M2)
- `GET /api/v1/events/:id/feedback` (M5)
- `GET /api/v1/notifications` (M7)
- `GET /api/v1/profile/events` (M8)
- `GET /api/v1/profile/feedback` (M8)

**Alcance:**
- Cubre la adición de índices vía migraciones Sequelize.
- Cubre la implementación de paginación offset-based en todos los endpoints de listado.
- No cubre paginación cursor-based (evaluar en el futuro si se requiere mayor performance en tablas muy grandes).
- No cubre particionado de tablas (medida a considerar si se superan los 10M de filas).

---

## Alternativas consideradas

**Opción A: Indexación + paginación offset (adoptada)**
- ✅ Implementación directa en Sequelize con `findAndCountAll`.
- ✅ Sencillo para el cliente: parámetros `page` y `limit` son intuitivos.
- ✅ Compatible con todos los clientes sin cambios en el protocolo.
- ❌ La paginación offset puede presentar inconsistencias si hay inserciones mientras se pagina (problema clásico de offset pagination).
- ❌ Para tablas muy grandes (>10M filas), el `OFFSET` alto sigue siendo costoso aunque haya índice.

**Opción B: Paginación cursor-based (keyset pagination)**
- ✅ Performance constante independientemente del offset; escala mejor a grandes volúmenes.
- ✅ Sin el problema de registros duplicados o saltados al insertar durante la paginación.
- ❌ Mayor complejidad de implementación (el cliente debe gestionar un cursor opaco).
- ❌ Más difícil de implementar con Sequelize sin helpers adicionales.
- ❌ No permite saltar a una página arbitraria (útil en UIs con selector de página).

**Opción C: Solo índices, sin paginación**
- ✅ Sin cambios en los contratos de API.
- ❌ Las responses con todos los registros seguirán siendo problematicas a gran escala.
- ❌ No resuelve el problema de uso de memoria en el servidor ni timeouts en clientes lentos.
- ❌ Solución incompleta; aplaza el problema en lugar de resolverlo.

---

## Consecuencias

**Beneficios esperados:**
- Las queries de `event_attendees` por `(eventId, userId)` bajan de O(n) a O(log n) con el índice B-tree.
- Las queries de notificaciones por `(userId, isRead)` son especialmente beneficiadas (tabla de alto volumen).
- Los endpoints de listado retornan respuestas de tamaño predecible y controlado (máximo 100 registros por página).
- El uso de memoria del servidor se reduce significativamente al no serializar colecciones completas en memoria.

**Costos o riesgos que se aceptan:**
- Los índices ocupan espacio en disco (~10-20% adicional sobre el tamaño de la tabla) y añaden overhead en operaciones de INSERT/UPDATE.
- La paginación offset puede retornar duplicados o saltear registros si hay inserciones concurrentes durante la paginación (aceptable para el caso de uso actual).
- Los clientes existentes que consumen los endpoints sin paginación necesitarán actualizar su integración para manejar el nuevo formato de respuesta.

**Impacto en operación y equipo:**
- Se deben crear migraciones Sequelize para agregar cada índice (nunca modificar directamente la BD).
- Los modelos existentes deben actualizarse con la definición de `indexes` en su configuración Sequelize.
- El helper de paginación debe centralizarse en `src/utils/pagination.js` para reutilización.

---

## Plan de implementación

1. Crear `src/utils/pagination.js`:
   ```javascript
   const paginate = (page = 1, limit = 20) => {
     const parsedLimit = Math.min(parseInt(limit), 100);
     const parsedPage = Math.max(parseInt(page), 1);
     return { limit: parsedLimit, offset: (parsedPage - 1) * parsedLimit };
   };

   const paginatedResponse = (count, rows, page, limit) => ({
     items: rows,
     pagination: {
       total: count,
       page: parseInt(page),
       limit: parseInt(limit),
       totalPages: Math.ceil(count / limit),
       hasNextPage: page * limit < count,
       hasPrevPage: page > 1
     }
   });

   module.exports = { paginate, paginatedResponse };
   ```

2. Crear migraciones para índices:
   ```javascript
   // Ejemplo: migration para event_attendees
   await queryInterface.addIndex('event_attendees', ['eventId'], { name: 'idx_event_attendees_event_id' });
   await queryInterface.addIndex('event_attendees', ['userId'], { name: 'idx_event_attendees_user_id' });
   await queryInterface.addIndex('notifications', ['userId', 'isRead'], { name: 'idx_notifications_user_read' });
   ```

3. Actualizar los modelos Sequelize con la propiedad `indexes` en su definición.

4. Refactorizar los endpoints de listado en M2, M5, M7 y M8 para usar `findAndCountAll` con el helper de paginación.

5. Actualizar `contracts.md` con el nuevo schema de respuesta paginada y los query params soportados.

**Dependencias:** PostgreSQL 14+ (ADR-001), Sequelize 6+ (ADR-004). Sin dependencias nuevas de npm.

**Métrica de éxito:**
- `EXPLAIN ANALYZE` en PostgreSQL confirma uso de índices (Index Scan) en lugar de Seq Scan para las queries objetivo.
- Tiempo de respuesta de `GET /api/v1/notifications` con 10,000 registros: < 50ms con índice (vs ~800ms sin índice).
- El tamaño de respuesta de `GET /api/v1/events` no supera los 50KB con el límite por defecto de 20 registros.

---

## Triggers de revisión

- Si alguna tabla supera los 5,000,000 de filas y se observa degradación a pesar de los índices (evaluar particionado de tablas en PostgreSQL).
- Si la paginación offset genera problemas de consistencia perceptibles para los usuarios (migrar a cursor-based pagination).
- Si se detectan índices no utilizados en el análisis de queries de producción (eliminarlos para reducir overhead de escritura).

**Fecha sugerida de revisión:** 2026-12-01
