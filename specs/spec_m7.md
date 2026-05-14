# Especificación Módulo 7: Notificaciones y Recordatorios


**Estado:** Finalizado

## Índice
1. [Descripción](#descripción)
2. [Objetivos y Contexto](#objetivos-y-contexto)
3. [Arquitectura](#arquitectura)
4. [Módulos y Responsabilidades](#módulos-y-responsabilidades)
5. [Flujo de Datos](#flujo-de-datos)
6. [Historias de Usuario y Criterios de Aceptación](#historias-de-usuario-y-criterios-de-aceptación)
7. [Requisitos Funcionales y Reglas de Negocio](#requisitos-funcionales-y-reglas-de-negocio)
8. [Restricciones Técnicas](#restricciones-técnicas)
9. [Endpoints y Contratos](#endpoints-y-contratos)
10. [Dependencias y Capas](#dependencias-y-capas)
11. [Modelo de Datos](#modelo-de-datos)
12. [Plan de Tareas](#plan-de-tareas)
13. [Estrategia de Verificación](#estrategia-de-verificación)

---

## Descripción

El Módulo 7 implementa el sistema de **Notificaciones y Recordatorios** del sistema de gestión de eventos. Permite notificar a los usuarios sobre eventos relevantes del sistema (registros, cancelaciones, recordatorios de eventos próximos, certificados disponibles) tanto mediante notificaciones in-app como por correo electrónico, utilizando Nodemailer como librería de envío.

---

## Objetivos y Contexto

- Notificar a los participantes al registrarse a un evento
- Enviar recordatorio automático 24 horas antes del inicio del evento
- Notificar a los inscritos cuando un evento es cancelado o modificado
- Notificar al participante cuando su certificado está disponible
- Notificar al organizador cuando recibe nuevo feedback
- Proveer un sistema de notificaciones in-app con estado leído/no leído
- Permitir al usuario gestionar (leer, eliminar) sus notificaciones
- Las notificaciones in-app y los correos se generan de forma desacoplada mediante un servicio centralizado

---

## Arquitectura

```
┌─────────────────────────────────────────────┐
│           Cliente (Frontend)                │
└─────────────────┬───────────────────────────┘
                  │ HTTP/REST
┌─────────────────▼───────────────────────────┐
│         Capa de Presentación (API)          │
│  ┌────────────────────┐  ┌───────────────┐ │
│  │ Notification Routes │→│Notification   │ │
│  └────────────────────┘  │Controller     │ │
│                           └───────────────┘ │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Lógica de Negocio           │
│         Notification Service               │
│   ┌────────────────┐  ┌─────────────────┐  │
│   │ In-App Service │  │  Email Service  │  │
│   └────────────────┘  └─────────────────┘  │
│              (Nodemailer)                   │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Acceso a Datos               │
│   Notification Model + User Model           │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Base de Datos (PostgreSQL)          │
└─────────────────────────────────────────────┘
```

---

## Módulos y Responsabilidades

### 1. Notification Controller (`src/controllers/notification-controller.js`)
**Responsabilidades:**
- Manejar requests HTTP para operaciones de notificaciones
- Verificar que el usuario accede solo a sus propias notificaciones
- Delegar lógica a Notification Service
- Formatear respuestas

**Métodos:**
- `getNotifications(req, res)` - Listar notificaciones del usuario autenticado
- `getUnreadCount(req, res)` - Obtener cantidad de notificaciones no leídas
- `markAsRead(req, res)` - Marcar una notificación como leída
- `markAllAsRead(req, res)` - Marcar todas las notificaciones del usuario como leídas
- `deleteNotification(req, res)` - Eliminar una notificación

---

### 2. Notification Service (`src/services/notification-service.js`)
**Responsabilidades:**
- Lógica de negocio de notificaciones in-app y por email
- Crear registros en la tabla `notifications`
- Enviar correos con Nodemailer
- Ser invocado por otros módulos (eventos, acreditación, certificados, feedback)

**Métodos:**
- `notify(userId, type, metadata)` - Crear notificación in-app y enviar email si corresponde
- `getByUser(userId, filters)` - Obtener notificaciones paginadas de un usuario
- `getUnreadCount(userId)` - Contar notificaciones no leídas
- `markAsRead(notificationId, userId)` - Marcar como leída
- `markAllAsRead(userId)` - Marcar todas como leídas
- `delete(notificationId, userId)` - Eliminar notificación
- `sendEmail(to, subject, body)` - Enviar email vía Nodemailer

---

### 3. Notification Model (`src/models/Notification.js`)
**Campos:**
- `id`: UUID
- `userId`: UUID (FK a User — destinatario)
- `type`: enum (ver tipos abajo)
- `title`: string (max 255)
- `message`: text
- `isRead`: boolean (default: false)
- `readAt`: DATE (nullable)
- `metadata`: JSON (datos extra: `eventId`, `certificateId`, etc.)
- `createdAt`: DATE
- `updatedAt`: DATE

**Tipos de notificación (`type`):**
| Valor | Descripción |
|-------|-------------|
| `event_registration` | Confirmación de registro a evento |
| `event_reminder` | Recordatorio 24h antes del evento |
| `event_cancelled` | Evento cancelado |
| `event_updated` | Datos del evento modificados |
| `checkin_confirmed` | Asistencia (check-in) confirmada |
| `certificate_available` | Certificado disponible para descarga |
| `feedback_received` | (organizador) Nuevo feedback recibido en su evento |

---

### 4. Notification Routes (`src/routes/notification-routes.js`)
**Endpoints:**
- `GET /api/v1/notifications` - Listar notificaciones propias
- `GET /api/v1/notifications/unread-count` - Contar no leídas
- `PUT /api/v1/notifications/:id/read` - Marcar una como leída
- `PUT /api/v1/notifications/read-all` - Marcar todas como leídas
- `DELETE /api/v1/notifications/:id` - Eliminar notificación

---

## Flujo de Datos

### Caso 1: Registro a Evento dispara notificación

```
[1] Participante → POST /api/v1/events/:id/register + Token
                ↓
[2] Event Service → registerUser(eventId, userId) [Módulo 2]
                ↓
[3] Event Service → notificationService.notify(userId, 'event_registration', { eventId })
                ↓
[4] Notification Service:
    a) Notification.create({ userId, type, title, message, metadata: { eventId } })
    b) User.findByPk(userId) → obtener email
    c) sendEmail(user.email, "Registro confirmado", "...")
                ↓
[5] PostgreSQL → INSERT INTO notifications
                ↓
[6] SMTP (Nodemailer) → Email enviado al usuario
```

---

### Caso 2: Usuario consulta sus notificaciones

```
[1] Usuario → GET /api/v1/notifications?page=1&unreadOnly=true + Token
                ↓
[2] Auth Middleware → Verifica JWT → extrae userId
                ↓
[3] Notification Controller → getNotifications(req, res)
                ↓
[4] Notification Service → getByUser(userId, { unreadOnly: true, page: 1 })
                ↓
[5] Notification.findAll({
      where: { userId, isRead: false },
      order: [['createdAt', 'DESC']],
      limit, offset
    })
                ↓
[6] Respuesta → 200 OK + lista paginada de notificaciones
```

---

### Caso 3: Recordatorio automático 24h antes del evento

```
[1] Cron Job → ejecuta cada hora
                ↓
[2] Notification Service → checkUpcomingEvents()
                ↓
[3] Event.findAll donde date BETWEEN (now + 23h) AND (now + 25h)
                ↓
[4] Para cada evento encontrado:
    EventAttendee.findAll({ where: { eventId } })
                ↓
[5] Para cada asistente registrado:
    notify(userId, 'event_reminder', { eventId })
                ↓
[6] Notification.create + sendEmail por cada usuario
```

---

### Caso 4: Cancelación de evento notifica a todos los inscritos

```
[1] Organizador → PUT /api/v1/events/:id con status: 'cancelled'
                ↓
[2] Event Service → update(id, { status: 'cancelled' }) [Módulo 2]
                ↓
[3] Event Service → notificationService.notifyEventCancelled(eventId)
                ↓
[4] Notification Service:
    - Obtiene todos los inscritos del evento
    - Por cada uno: notify(userId, 'event_cancelled', { eventId })
                ↓
[5] INSERT INTO notifications (N registros) + N emails enviados
```

---

## Historias de Usuario y Criterios de Aceptación

### HU701: Recibir notificación al registrarse a un evento
**Como** participante que se registra a un evento  
**Quiero** recibir una confirmación por email y una notificación in-app  
**Para** tener constancia de mi registro  

**Criterios de Aceptación:**
- Al completar el registro, se crea una notificación de tipo `event_registration`
- Se envía un email de confirmación al correo del usuario
- La notificación aparece en `/api/v1/notifications` como no leída
- El email incluye título del evento, fecha, ubicación y nombre del usuario

---

### HU702: Recibir recordatorio antes del evento
**Como** participante registrado a un evento  
**Quiero** recibir un recordatorio 24 horas antes del inicio  
**Para** no olvidarme de asistir  

**Criterios de Aceptación:**
- El recordatorio se envía automáticamente 24h (±1h) antes del evento
- Solo se envía a usuarios con `attended = false` (que aún no hicieron check-in)
- El recordatorio indica título, fecha, hora y ubicación del evento
- No se envía recordatorio si el evento fue cancelado

---

### HU703: Gestionar notificaciones in-app
**Como** usuario autenticado  
**Quiero** ver, marcar como leídas y eliminar mis notificaciones  
**Para** mantener mi bandeja de entrada organizada  

**Criterios de Aceptación:**
- Puedo listar mis notificaciones ordenadas por fecha descendente
- Puedo filtrar por no leídas (`unreadOnly=true`)
- Puedo marcar una o todas como leídas
- Puedo eliminar una notificación propia
- No puedo acceder ni modificar notificaciones de otros usuarios

---

### HU704: Ser notificado cuando un evento es cancelado
**Como** participante registrado a un evento  
**Quiero** recibir una notificación cuando el evento sea cancelado  
**Para** poder reorganizar mis planes  

**Criterios de Aceptación:**
- Al cancelar un evento, todos los inscritos reciben notificación in-app y email
- La notificación indica el nombre del evento y la fecha original
- El proceso no falla si algún email rebota (manejo de errores individual)

---

### HU705: Notificación de certificado disponible
**Como** participante que asistió a un evento  
**Quiero** recibir una notificación cuando mi certificado esté disponible  
**Para** saber cuándo puedo descargarlo  

**Criterios de Aceptación:**
- Al finalizar el evento (date < ahora), los usuarios con `attended = true` reciben notificación `certificate_available`
- La notificación incluye un link/referencia al endpoint de descarga
- La notificación se genera una única vez por usuario/evento

---

## Requisitos Funcionales y Reglas de Negocio

1. **RN701**: Un usuario solo puede ver, leer y eliminar sus propias notificaciones
2. **RN702**: Las notificaciones se generan automáticamente por acciones de otros módulos; no se crean manualmente vía API pública
3. **RN703**: El envío de emails es no bloqueante; si el email falla, la notificación in-app se persiste de todas formas
4. **RN704**: El recordatorio de 24h solo se envía si el evento no fue cancelado
5. **RN705**: El recordatorio de 24h solo se envía a participantes que aún no hicieron check-in (para no spamear a quienes ya asistieron en eventos de múltiples días)
6. **RN706**: La notificación `certificate_available` se genera una única vez por par (userId, eventId)
7. **RN707**: Las notificaciones in-app persisten en base de datos; los emails no se almacenan
8. **RN708**: El listado de notificaciones soporta paginación (default: 20 por página)
9. **RN709**: `readAt` se registra con el timestamp exacto en que se marca como leída
10. **RN710**: Al eliminar una notificación, se elimina físicamente del sistema (no soft delete)

---

## Restricciones Técnicas

- **Nodemailer**: librería para envío de emails (SMTP configurable vía `.env`)
- **node-cron**: librería para tareas programadas (recordatorios)
- **Variables de entorno requeridas**:
  - `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`
  - `SMTP_FROM` (email remitente, ej: `noreply@eventos.com`)
- El cron job de recordatorios se ejecuta cada hora
- En entorno de desarrollo/test, el envío de emails puede desactivarse mediante `EMAIL_ENABLED=false`

---

## Endpoints y Contratos

### 7.1 Listar Notificaciones del Usuario
**GET** `/api/v1/notifications`

**Headers:**
```
Authorization: Bearer <token>
```

**Query Params:**
```
page: number (default: 1)
limit: number (default: 20)
unreadOnly: boolean (default: false)
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "notifications": [
      {
        "id": "uuid",
        "type": "event_registration",
        "title": "Registro confirmado",
        "message": "Te registraste exitosamente al evento 'Conferencia de Tecnología'.",
        "isRead": false,
        "readAt": null,
        "metadata": {
          "eventId": "uuid",
          "eventTitle": "Conferencia de Tecnología"
        },
        "createdAt": "2026-06-14T15:00:00.000Z"
      }
    ],
    "pagination": {
      "total": 15,
      "page": 1,
      "limit": 20,
      "totalPages": 1
    }
  },
  "message": "Notificaciones obtenidas exitosamente"
}
```

---

### 7.2 Contar Notificaciones No Leídas
**GET** `/api/v1/notifications/unread-count`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "unreadCount": 3
  },
  "message": "Conteo de notificaciones no leídas obtenido"
}
```

---

### 7.3 Marcar Notificación como Leída
**PUT** `/api/v1/notifications/:id/read`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "isRead": true,
    "readAt": "2026-06-14T16:00:00.000Z"
  },
  "message": "Notificación marcada como leída"
}
```

**Response 404:**
```json
{
  "success": false,
  "error": {
    "code": "NOTIFICATION_NOT_FOUND",
    "message": "Notificación no encontrada"
  }
}
```

**Response 403:**
```json
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "No tiene permisos para modificar esta notificación"
  }
}
```

---

### 7.4 Marcar Todas las Notificaciones como Leídas
**PUT** `/api/v1/notifications/read-all`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "updatedCount": 5
  },
  "message": "Todas las notificaciones marcadas como leídas"
}
```

---

### 7.5 Eliminar Notificación
**DELETE** `/api/v1/notifications/:id`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "success": true,
  "data": null,
  "message": "Notificación eliminada exitosamente"
}
```

**Response 404:**
```json
{
  "success": false,
  "error": {
    "code": "NOTIFICATION_NOT_FOUND",
    "message": "Notificación no encontrada"
  }
}
```

---

## Dependencias y Capas

```
Módulo 7: Notificaciones y Recordatorios
├── Dependencias Internas:
│   ├── User Model (email y nombre del destinatario)
│   ├── Event Model (datos del evento en notificaciones)
│   ├── EventAttendee Model (lista de inscritos para notificaciones masivas)
│   └── Auth Middleware
│
├── Módulos que invocan Notification Service:
│   ├── Módulo 2: event_registration, event_cancelled, event_updated
│   ├── Módulo 3: checkin_confirmed
│   ├── Módulo 4: certificate_available
│   └── Módulo 5: feedback_received
│
├── Dependencias Externas:
│   ├── nodemailer (envío de emails SMTP)
│   ├── node-cron (tareas programadas para recordatorios)
│   ├── express (routing)
│   ├── sequelize (ORM)
│   └── jsonwebtoken (autenticación)
│
└── Base de Datos:
    └── PostgreSQL
        └── Tabla: notifications
```

---

## Modelo de Datos

### Notification Model

```javascript
// src/models/Notification.js
const { DataTypes } = require('sequelize');
const { sequelize } = require('../config/database');

const Notification = sequelize.define('Notification', {
  id: {
    type: DataTypes.UUID,
    defaultValue: DataTypes.UUIDV4,
    primaryKey: true
  },
  userId: {
    type: DataTypes.UUID,
    allowNull: false
  },
  type: {
    type: DataTypes.ENUM(
      'event_registration',
      'event_reminder',
      'event_cancelled',
      'event_updated',
      'checkin_confirmed',
      'certificate_available',
      'feedback_received'
    ),
    allowNull: false
  },
  title: {
    type: DataTypes.STRING(255),
    allowNull: false
  },
  message: {
    type: DataTypes.TEXT,
    allowNull: false
  },
  isRead: {
    type: DataTypes.BOOLEAN,
    defaultValue: false
  },
  readAt: {
    type: DataTypes.DATE,
    allowNull: true
  },
  metadata: {
    type: DataTypes.JSON,
    allowNull: true,
    defaultValue: {}
  }
}, {
  tableName: 'notifications',
  timestamps: true,
  indexes: [
    { fields: ['userId'] },
    { fields: ['userId', 'isRead'] },
    { fields: ['createdAt'] }
  ]
});

// Relaciones
Notification.belongsTo(User, { foreignKey: 'userId', as: 'user' });

module.exports = Notification;
```

### Estructura de `metadata` por tipo

```javascript
// event_registration / event_reminder / event_cancelled / event_updated / checkin_confirmed
{ eventId: "uuid", eventTitle: "Conferencia de Tecnología" }

// certificate_available
{ eventId: "uuid", eventTitle: "Conferencia de Tecnología", downloadUrl: "/api/v1/certificates/{eventId}/download" }

// feedback_received
{ eventId: "uuid", eventTitle: "Conferencia de Tecnología", totalFeedbacks: 12, averageRating: 4.3 }
```

---

## Plan de Tareas

| # | Tarea | Descripción | Estimación |
|---|-------|-------------|------------|
| T701 | Setup Notification Model | Crear modelo, migración y relaciones | 1.5h |
| T702 | Notification Service base | Implementar `notify()`, `sendEmail()`, `getByUser()` | 4h |
| T703 | Configurar Nodemailer | Setup SMTP, templates básicos de email, env vars | 2h |
| T704 | Notification Controller | Implementar CRUD de notificaciones del usuario | 2.5h |
| T705 | Notification Routes | Configurar rutas con middlewares de auth | 0.5h |
| T706 | Integración con Módulo 2 | Disparar notificaciones en registro y cambio de estado de evento | 2h |
| T707 | Integración con Módulo 3 | Disparar notificación en check-in | 1h |
| T708 | Integración con Módulo 4 | Disparar notificación de certificado disponible | 1h |
| T709 | Integración con Módulo 5 | Disparar notificación de feedback al organizador | 1h |
| T710 | Cron Job recordatorios | Implementar tarea programada con node-cron | 3h |
| T711 | Tests unitarios | Tests de service: notify, getByUser, markAsRead, cron logic | 4h |
| T712 | Tests integración | Tests de endpoints con base de datos | 3h |

**Total estimado: 25.5 horas**

---

## Estrategia de Verificación

### Tests Unitarios
- **Notification Service**:
  - `notify()`: verificar que crea registro en DB y llama a `sendEmail`
  - `notify()`: verificar que si `sendEmail` falla, la notificación in-app se persiste igual
  - `getByUser()`: verificar paginación y filtro `unreadOnly`
  - `markAsRead()`: verificar que actualiza `isRead=true` y `readAt`
  - `markAllAsRead()`: verificar que solo afecta notificaciones del usuario solicitante
  - `delete()`: verificar que lanza error si la notificación no pertenece al usuario
  - Cron lógica: verificar que `checkUpcomingEvents()` filtra eventos en el rango correcto

### Tests de Integración
- `GET /api/v1/notifications`: verificar lista paginada con filtros
- `GET /api/v1/notifications/unread-count`: verificar conteo correcto
- `PUT /api/v1/notifications/:id/read`: verificar actualización de estado
- `PUT /api/v1/notifications/read-all`: verificar actualización masiva
- `DELETE /api/v1/notifications/:id`: verificar eliminación y control de propiedad
- Verificar que usuario B no puede modificar notificaciones de usuario A

### Casos de Verificación Manual

| ID | Caso | Resultado Esperado |
|----|------|-------------------|
| V701 | Participante se registra a evento | Notificación in-app + email creados |
| V702 | Evento cancelado con 10 inscritos | 10 notificaciones in-app + 10 emails |
| V703 | Cron ejecuta con evento a 24h | Notificaciones de recordatorio generadas |
| V704 | Cron ejecuta con evento a 48h | Sin notificaciones generadas |
| V705 | Usuario marca notificación ajena como leída | 403 FORBIDDEN |
| V706 | Usuario lista notificaciones con `unreadOnly=true` | Solo devuelve no leídas |
| V707 | `markAllAsRead` con 5 no leídas | `updatedCount: 5`, todas con `isRead: true` |
| V708 | Email SMTP falla en envío | Notificación in-app persistida, error logueado, no rompe flujo |
| V709 | Participante con `attended=true` al momento del cron | No recibe recordatorio |
| V710 | Certificado disponible tras finalizar evento | Notificación `certificate_available` creada |

---

## Conclusión

El Módulo 7 (Notificaciones y Recordatorios) añade comunicación activa al sistema, cerrando el ciclo de interacción con el usuario más allá de las respuestas HTTP inmediatas. Al funcionar como un servicio transversal invocado por otros módulos, mantiene el bajo acoplamiento del sistema y centraliza toda la lógica de notificaciones en un único punto de entrada.

**Integración con módulos anteriores:**
- Módulo 2: dispara notificaciones en registro, cancelación y modificación de eventos
- Módulo 3: dispara notificación al confirmar check-in
- Módulo 4: dispara notificación cuando el certificado queda disponible
- Módulo 5: notifica al organizador al recibir nuevo feedback
