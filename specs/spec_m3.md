# Especificación Módulo 3: Acreditación

## Índice
1. [Descripción](#descripción)
2. [Arquitectura](#arquitectura)
3. [Módulos y Responsabilidades](#módulos-y-responsabilidades)
4. [Flujo de Datos](#flujo-de-datos)
5. [Endpoints y Contratos](#endpoints-y-contratos)
6. [Modelos de Datos](#modelos-de-datos)
7. [Reglas de Negocio](#reglas-de-negocio)

---

## Descripción

El Módulo 3 implementa el sistema de **Acreditación** que permite registrar la asistencia de participantes a eventos. Este módulo extiende la relación pivot `event_attendees` para incluir información de check-in.

### Objetivos
- Registrar asistencia (check-in) de participantes a eventos
- Verificar elegibilidad para check-in
- Consultar estado de acreditación
- Listar asistentes con estado de acreditación

---

## Arquitectura

```
┌─────────────────────────────────────────────┐
│           Cliente (Frontend)                │
└─────────────────┬───────────────────────────┘
                  │ HTTP/REST
┌─────────────────▼───────────────────────────┐
│         Capa de Presentación (API)          │
│  ┌──────────────────┐  ┌────────────────┐ │
│  │AccreditationRoutes│→│AccreditationCtrl│ │
│  └──────────────────┘  └────────────────┘ │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Lógica de Negocio           │
│         Accreditation Service               │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Acceso a Datos               │
│    EventAttendee Model (pivot actualizado)  │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Base de Datos (PostgreSQL)          │
└─────────────────────────────────────────────┘
```

---

## Módulos y Responsabilidades

### 1. Accreditation Controller (`src/controllers/accreditation-controller.js`)
**Métodos:**
- `checkIn(req, res)` - Registrar asistencia a evento
- `getCheckInStatus(req, res)` - Ver estado de check-in
- `getEventAccreditation(req, res)` - Listar acreditaciones de un evento

### 2. Accreditation Service (`src/services/accreditation-service.js`)
**Métodos:**
- `checkIn(eventId, userId)` - Registrar check-in
- `getCheckInStatus(eventId, userId)` - Obtener estado
- `getEventAccreditation(eventId)` - Obtener todas las acreditaciones

---

## Flujo de Datos

### Caso: Registrar Check-In

```
[1] Participante → POST /api/v1/events/:id/check-in + Token
                ↓
[2] Auth Middleware → Verifica JWT → extrae userId
                ↓
[3] Accreditation Controller → checkIn(req, res)
                ↓
[4] Accreditation Service → checkIn(eventId, userId)
                ↓
[5] Validaciones:
    - ¿Evento existe?
    - ¿Usuario registrado al evento?
    - ¿Evento está en progreso o finalizado?
    - ¿No hizo check-in previamente?
                ↓
[6] Update → EventAttendee.update({ attended: true, checkInAt: now })
                ↓
[7] Respuesta → 200 OK con datos de acreditación
```

---

## Endpoints y Contratos

### 3.1 Registrar Check-In
**POST** `/api/v1/events/:id/check-in`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "eventId": "uuid",
    "userId": "uuid",
    "attended": true,
    "checkInAt": "2026-06-15T10:30:00.000Z"
  },
  "message": "Asistencia registrada exitosamente"
}
```

**Response 400:**
```json
{
  "success": false,
  "error": {
    "code": "ALREADY_CHECKED_IN",
    "message": "Ya se registró asistencia para este evento"
  }
}
```

**Response 404:**
```json
{
  "success": false,
  "error": {
    "code": "NOT_REGISTERED",
    "message": "El usuario no está registrado en este evento"
  }
}
```

---

### 3.2 Ver Estado de Check-In
**GET** `/api/v1/events/:id/check-in/status`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "eventId": "uuid",
    "userId": "uuid",
    "attended": true,
    "checkInAt": "2026-06-15T10:30:00.000Z"
  },
  "message": "Estado de acreditación obtenido"
}
```

**Response 404:**
```json
{
  "success": false,
  "error": {
    "code": "NOT_REGISTERED",
    "message": "El usuario no está registrado en este evento"
  }
}
```

---

### 3.3 Listar Acreditaciones del Evento
**GET** `/api/v1/events/:id/accreditation`

**Headers:**
```
Authorization: Bearer <token>
```
**Rol requerido:** Organizador

**Response 200:**
```json
{
  "success": true,
  "data": [
    {
      "userId": "uuid",
      "user": {
        "firstName": "Juan",
        "lastName": "Pérez",
        "email": "juan@example.com"
      },
      "attended": true,
      "checkInAt": "2026-06-15T10:30:00.000Z"
    }
  ],
  "message": "Acreditaciones obtenidas exitosamente"
}
```

---

## Modelos de Datos

### EventAttendee (Pivot actualizado)

```javascript
const EventAttendee = sequelize.define('EventAttendee', {
  eventId: { type: DataTypes.UUID, primaryKey: true },
  userId: { type: DataTypes.UUID, primaryKey: true },
  attended: { type: DataTypes.BOOLEAN, defaultValue: false },
  checkInAt: { type: DataTypes.DATE, allowNull: true }
}, {
  tableName: 'event_attendees',
  timestamps: true
});
```

---

## Reglas de Negocio

1. **RN301**: Solo usuarios registrados al evento pueden hacer check-in
2. **RN302**: Un usuario solo puede hacer check-in una vez por evento
3. **RN303**: El check-in se registra con la fecha/hora actual del sistema
4. **RN304**: Solo el organizador del evento puede ver la lista de acreditaciones
5. **RN305**: El evento debe estar en estado 'published' para permitir check-in
6. **RN306**: El campo `attended` se setea a `true` al momento del check-in
7. **RN307**: El campo `checkInAt` registra el timestamp exacto del check-in
