# Especificación Módulo 8: Gestión de Perfil de Usuario


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

El Módulo 8 implementa la **Gestión de Perfil de Usuario**, que permite a cada usuario autenticado visualizar y actualizar sus datos personales, cambiar su contraseña, consultar su historial de actividad en el sistema (eventos registrados, charlas dictadas, feedback enviado) y, opcionalmente, eliminar su cuenta. Este módulo opera exclusivamente sobre los datos del usuario autenticado, sin exponer información de otros usuarios.

---

## Objetivos y Contexto

- Permitir al usuario ver y editar su información personal (nombre, email)
- Permitir al usuario cambiar su contraseña de forma segura
- Proveer al participante un historial de sus eventos registrados y asistidos
- Proveer al disertante un listado de sus charlas programadas
- Proveer al organizador un resumen de los eventos que creó
- Consolidar la vista del historial de feedback enviado por el usuario
- Permitir la eliminación de cuenta con validaciones de integridad
- Centralizar toda la lógica de auto-gestión del usuario en un único módulo

---

## Arquitectura

```
┌─────────────────────────────────────────────┐
│           Cliente (Frontend)                │
└─────────────────┬───────────────────────────┘
                  │ HTTP/REST
┌─────────────────▼───────────────────────────┐
│         Capa de Presentación (API)          │
│  ┌──────────────────┐  ┌─────────────────┐ │
│  │  Profile Routes  │→ │Profile Controller│ │
│  └──────────────────┘  └─────────────────┘ │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Lógica de Negocio           │
│         Profile Service                     │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Acceso a Datos               │
│  User + Event + Talk + EventAttendee +      │
│  Feedback + Notification Models             │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Base de Datos (PostgreSQL)          │
└─────────────────────────────────────────────┘
```

---

## Módulos y Responsabilidades

### 1. Profile Controller (`src/controllers/profile-controller.js`)
**Responsabilidades:**
- Manejar requests HTTP para operaciones de perfil
- Asegurar que todas las operaciones actúen sobre el usuario autenticado (`req.user.id`)
- Delegar lógica a Profile Service
- Formatear respuestas

**Métodos:**
- `getProfile(req, res)` - Obtener datos del perfil del usuario autenticado
- `updateProfile(req, res)` - Actualizar datos personales (nombre, email)
- `changePassword(req, res)` - Cambiar contraseña
- `getMyEvents(req, res)` - Historial de eventos según rol del usuario
- `getMyTalks(req, res)` - Charlas del disertante autenticado
- `getMyFeedback(req, res)` - Feedback enviado por el usuario
- `deleteAccount(req, res)` - Eliminar cuenta del usuario autenticado

---

### 2. Profile Service (`src/services/profile-service.js`)
**Responsabilidades:**
- Lógica de negocio de gestión de perfil
- Validar contraseña actual antes de permitir cambio
- Verificar unicidad de email al actualizar
- Consultar historial de actividad del usuario
- Gestionar eliminación de cuenta con verificación de integridad

**Métodos:**
- `getProfile(userId)` - Obtener perfil completo con rol
- `updateProfile(userId, data)` - Actualizar firstName, lastName, email
- `changePassword(userId, currentPassword, newPassword)` - Cambiar contraseña
- `getMyEvents(userId, role)` - Obtener eventos según rol (organizer/participant/speaker)
- `getMyTalks(userId)` - Obtener charlas del disertante
- `getMyFeedback(userId)` - Obtener feedback enviado
- `deleteAccount(userId)` - Eliminar cuenta con validaciones

---

## Flujo de Datos

### Caso 1: Ver Perfil

```
[1] Usuario → GET /api/v1/profile + Token
                ↓
[2] Auth Middleware → Verifica JWT → extrae userId
                ↓
[3] Profile Controller → getProfile(req, res)
                ↓
[4] Profile Service → getProfile(userId)
                ↓
[5] User.findByPk(userId, { include: [{ model: Role, as: 'role' }] })
                ↓
[6] Respuesta → 200 OK + datos del perfil (sin password hash)
```

---

### Caso 2: Cambiar Contraseña

```
[1] Usuario → PUT /api/v1/profile/password + Token + Body { currentPassword, newPassword }
                ↓
[2] Auth Middleware → Verifica JWT → extrae userId
                ↓
[3] Profile Controller → changePassword(req, res)
                ↓
[4] Profile Service → changePassword(userId, currentPassword, newPassword)
                ↓
[5] Validaciones:
    - User.findByPk(userId) → obtener hash actual
    - bcrypt.compare(currentPassword, user.password) → ¿contraseña actual correcta?
    - ¿newPassword cumple requisitos mínimos? (min 8 chars)
    - ¿newPassword !== currentPassword?
                ↓
[6] bcrypt.hash(newPassword, 10)
                ↓
[7] user.update({ password: newHash })
                ↓
[8] Respuesta → 200 OK "Contraseña actualizada exitosamente"
```

---

### Caso 3: Historial de eventos del participante

```
[1] Participante → GET /api/v1/profile/events + Token
                ↓
[2] Auth Middleware → Verifica JWT → extrae userId, rol: 'participante'
                ↓
[3] Profile Controller → getMyEvents(req, res)
                ↓
[4] Profile Service → getMyEvents(userId, 'participante')
                ↓
[5] EventAttendee.findAll({
      where: { userId },
      include: [{ model: Event, as: 'event', include: ['organizer'] }],
      order: [['createdAt', 'DESC']]
    })
                ↓
[6] Respuesta → 200 OK + lista de eventos con estado de asistencia
```

---

### Caso 4: Eliminar Cuenta

```
[1] Usuario → DELETE /api/v1/profile + Token + Body { password }
                ↓
[2] Auth Middleware → Verifica JWT → extrae userId
                ↓
[3] Profile Controller → deleteAccount(req, res)
                ↓
[4] Profile Service → deleteAccount(userId)
                ↓
[5] Validaciones:
    - bcrypt.compare(password, user.password) → confirmar identidad
    - ¿Usuario es organizador de eventos futuros activos? → bloquear
                ↓
[6] Transacción:
    - Eliminar notificaciones del usuario
    - Eliminar feedback del usuario
    - Eliminar survey responses del usuario
    - Eliminar registros en event_attendees
    - User.destroy({ where: { id: userId } })
                ↓
[7] Respuesta → 200 OK "Cuenta eliminada"
```

---

## Historias de Usuario y Criterios de Aceptación

### HU801: Ver y editar mi perfil
**Como** usuario autenticado  
**Quiero** ver y actualizar mis datos personales  
**Para** mantener mi información actualizada en el sistema  

**Criterios de Aceptación:**
- Puedo ver mi nombre, apellido, email y rol actual
- Puedo actualizar mi nombre y apellido
- Puedo actualizar mi email si no está registrado por otro usuario
- Los cambios se reflejan inmediatamente en el token al próximo login
- El hash de contraseña nunca se devuelve en la respuesta

---

### HU802: Cambiar contraseña
**Como** usuario autenticado  
**Quiero** poder cambiar mi contraseña  
**Para** mantener la seguridad de mi cuenta  

**Criterios de Aceptación:**
- Debo proveer la contraseña actual para poder cambiarla
- La nueva contraseña debe tener al menos 8 caracteres
- La nueva contraseña no puede ser igual a la actual
- Si la contraseña actual es incorrecta, recibo error 400
- Tras cambiar la contraseña, los tokens anteriores siguen válidos hasta su expiración (comportamiento JWT stateless)

---

### HU803: Ver mi historial de eventos
**Como** participante  
**Quiero** ver todos los eventos a los que me registré  
**Para** llevar un registro de mi participación  

**Criterios de Aceptación:**
- Veo la lista de eventos con fecha, título, ubicación y estado de asistencia (Asistió/No asistió)
- Los eventos se ordenan por fecha descendente (más recientes primero)
- Se indica si el certificado está disponible para eventos finalizados con asistencia confirmada

---

### HU804: Ver mis charlas (disertante)
**Como** disertante  
**Quiero** ver las charlas que tengo programadas  
**Para** gestionar mi agenda de presentaciones  

**Criterios de Aceptación:**
- Veo todas mis charlas con título, evento, fecha, hora de inicio y fin
- Las charlas se ordenan por fecha de inicio ascendente (próximas primero)
- Puedo filtrar entre charlas pasadas y futuras

---

### HU805: Eliminar mi cuenta
**Como** usuario autenticado  
**Quiero** poder eliminar mi cuenta  
**Para** ejercer mi derecho a retirar mis datos del sistema  

**Criterios de Aceptación:**
- Debo confirmar con mi contraseña antes de que la cuenta sea eliminada
- Si soy organizador de eventos futuros activos, no puedo eliminar la cuenta sin antes cancelar esos eventos
- Al eliminarse la cuenta, se eliminan mis datos personales, feedback, notificaciones y registros a eventos
- Los eventos pasados que organicé permanecen en el sistema (integridad histórica)

---

## Requisitos Funcionales y Reglas de Negocio

1. **RN801**: Todas las operaciones de perfil actúan exclusivamente sobre el usuario autenticado (userId del JWT)
2. **RN802**: El email actualizado debe ser único en el sistema; si ya existe, se retorna error 400
3. **RN803**: Para cambiar la contraseña se requiere la contraseña actual correcta
4. **RN804**: La nueva contraseña debe tener un mínimo de 8 caracteres
5. **RN805**: La nueva contraseña no puede ser idéntica a la actual
6. **RN806**: El hash de contraseña nunca debe incluirse en ninguna respuesta de perfil
7. **RN807**: `getMyEvents` devuelve información diferente según el rol:
   - **Participante**: eventos donde está en `event_attendees`
   - **Organizador**: eventos donde `organizerId = userId`
   - **Disertante**: eventos donde tiene charlas asignadas
8. **RN808**: Un organizador no puede eliminar su cuenta si tiene eventos futuros en estado `published`
9. **RN809**: La eliminación de cuenta se realiza dentro de una transacción; si algún paso falla, se revierte todo
10. **RN810**: Los eventos y charlas históricas no se eliminan al borrar una cuenta (integridad referencial; el campo `organizerId`/`speakerId` puede quedar como referencia huérfana o setearse a null según política del sistema)

---

## Restricciones Técnicas

- No se crean modelos nuevos; este módulo reutiliza los modelos existentes: `User`, `Role`, `Event`, `Talk`, `EventAttendee`, `Feedback`, `SurveyResponse`, `Notification`
- El cambio de contraseña usa `bcrypt` (consistente con el resto del sistema)
- La eliminación de cuenta usa transacciones Sequelize para garantizar atomicidad
- El endpoint `DELETE /api/v1/profile` requiere el body con `{ password }` como confirmación explícita

---

## Endpoints y Contratos

### 8.1 Obtener Perfil
**GET** `/api/v1/profile`

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
    "firstName": "Juan",
    "lastName": "Pérez",
    "email": "juan@example.com",
    "role": {
      "id": "uuid",
      "name": "participante",
      "description": "Asiste a eventos"
    },
    "createdAt": "2026-01-10T10:00:00.000Z",
    "updatedAt": "2026-06-01T12:00:00.000Z"
  },
  "message": "Perfil obtenido exitosamente"
}
```

---

### 8.2 Actualizar Perfil
**PUT** `/api/v1/profile`

**Headers:**
```
Authorization: Bearer <token>
```

**Body:**
```json
{
  "firstName": "Juan Carlos",
  "lastName": "Pérez",
  "email": "juancarlos@example.com"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "firstName": "Juan Carlos",
    "lastName": "Pérez",
    "email": "juancarlos@example.com",
    "role": { "id": "uuid", "name": "participante" },
    "updatedAt": "2026-06-15T14:30:00.000Z"
  },
  "message": "Perfil actualizado exitosamente"
}
```

**Response 400 (email duplicado):**
```json
{
  "success": false,
  "error": {
    "code": "EMAIL_ALREADY_IN_USE",
    "message": "El email ya está registrado por otro usuario"
  }
}
```

---

### 8.3 Cambiar Contraseña
**PUT** `/api/v1/profile/password`

**Headers:**
```
Authorization: Bearer <token>
```

**Body:**
```json
{
  "currentPassword": "password123",
  "newPassword": "newSecurePass456"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": null,
  "message": "Contraseña actualizada exitosamente"
}
```

**Response 400 (contraseña actual incorrecta):**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_CURRENT_PASSWORD",
    "message": "La contraseña actual es incorrecta"
  }
}
```

**Response 400 (nueva igual a actual):**
```json
{
  "success": false,
  "error": {
    "code": "SAME_PASSWORD",
    "message": "La nueva contraseña no puede ser igual a la actual"
  }
}
```

---

### 8.4 Historial de Eventos del Usuario
**GET** `/api/v1/profile/events`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200 (participante):**
```json
{
  "success": true,
  "data": [
    {
      "eventId": "uuid",
      "title": "Conferencia de Tecnología",
      "date": "2026-06-15T10:00:00.000Z",
      "location": "Auditorio Principal",
      "status": "published",
      "attended": true,
      "checkInAt": "2026-06-15T10:30:00.000Z",
      "certificateAvailable": true
    }
  ],
  "message": "Historial de eventos obtenido"
}
```

**Response 200 (organizador):**
```json
{
  "success": true,
  "data": [
    {
      "eventId": "uuid",
      "title": "Conferencia de Tecnología",
      "date": "2026-06-15T10:00:00.000Z",
      "location": "Auditorio Principal",
      "status": "published",
      "totalRegistered": 48,
      "totalAttended": 42
    }
  ],
  "message": "Historial de eventos obtenido"
}
```

---

### 8.5 Charlas del Disertante
**GET** `/api/v1/profile/talks`

**Headers:**
```
Authorization: Bearer <token>
```
**Rol requerido:** Disertante

**Query Params:**
```
upcoming: boolean (default: false — devuelve todas)
```

**Response 200:**
```json
{
  "success": true,
  "data": [
    {
      "talkId": "uuid",
      "title": "Introducción a la IA",
      "description": "Un repaso por los fundamentos...",
      "startTime": "2026-06-15T10:00:00.000Z",
      "endTime": "2026-06-15T11:00:00.000Z",
      "event": {
        "id": "uuid",
        "title": "Conferencia de Tecnología",
        "date": "2026-06-15T10:00:00.000Z",
        "location": "Auditorio Principal",
        "status": "published"
      }
    }
  ],
  "message": "Charlas obtenidas exitosamente"
}
```

**Response 403 (no es disertante):**
```json
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "Este recurso es solo para disertantes"
  }
}
```

---

### 8.6 Feedback Enviado por el Usuario
**GET** `/api/v1/profile/feedback`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "success": true,
  "data": [
    {
      "feedbackId": "uuid",
      "rating": 5,
      "comment": "Excelente evento, muy bien organizado.",
      "createdAt": "2026-06-16T09:00:00.000Z",
      "event": {
        "id": "uuid",
        "title": "Conferencia de Tecnología",
        "date": "2026-06-15T10:00:00.000Z"
      }
    }
  ],
  "message": "Historial de feedback obtenido"
}
```

---

### 8.7 Eliminar Cuenta
**DELETE** `/api/v1/profile`

**Headers:**
```
Authorization: Bearer <token>
```

**Body:**
```json
{
  "password": "password123"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": null,
  "message": "Cuenta eliminada exitosamente"
}
```

**Response 400 (organizador con eventos activos):**
```json
{
  "success": false,
  "error": {
    "code": "HAS_ACTIVE_EVENTS",
    "message": "No puede eliminar su cuenta mientras tenga eventos futuros activos. Cancele o finalice sus eventos primero."
  }
}
```

**Response 400 (contraseña incorrecta):**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_PASSWORD",
    "message": "Contraseña incorrecta. No se puede eliminar la cuenta."
  }
}
```

---

## Dependencias y Capas

```
Módulo 8: Gestión de Perfil de Usuario
├── Dependencias Internas (modelos reutilizados):
│   ├── User Model (datos personales, contraseña)
│   ├── Role Model (rol del usuario)
│   ├── Event Model (historial organizador / participante)
│   ├── Talk Model (historial disertante)
│   ├── EventAttendee Model (historial participante + asistencia)
│   ├── Feedback Model (historial feedback enviado)
│   ├── SurveyResponse Model (eliminación en cascada)
│   └── Notification Model (eliminación en cascada)
│
├── Dependencias de Módulos Anteriores:
│   ├── Módulo 1: Role checking para `getMyTalks` (solo disertante)
│   ├── Módulo 2: Event data para historial
│   ├── Módulo 3: EventAttendee para estado de asistencia
│   ├── Módulo 4: Lógica de certificateAvailable
│   └── Módulo 5: Feedback para historial
│
├── Dependencias Externas:
│   ├── bcrypt (verificación y hash de contraseña)
│   ├── express (routing)
│   ├── sequelize (ORM, transacciones)
│   ├── joi (validación)
│   └── jsonwebtoken (autenticación)
│
└── Base de Datos:
    └── PostgreSQL
        └── (sin nuevas tablas — solo lecturas y modificaciones a tablas existentes)
```

---

## Modelo de Datos

> **Nota**: Este módulo no crea nuevos modelos. Extiende el uso del modelo `User` existente y consulta los modelos de los módulos anteriores.

### Campos del modelo User utilizados

```javascript
// src/models/User.js (existente — campos relevantes para este módulo)
{
  id: UUID,
  email: STRING (unique),
  password: STRING (bcrypt hash),
  firstName: STRING,
  lastName: STRING,
  roleId: UUID (FK a Role),
  createdAt: DATE,
  updatedAt: DATE
}
```

### DTOs de este módulo

```typescript
// ProfileUpdateDTO
interface ProfileUpdateDTO {
  firstName?: string; // 2-100 chars
  lastName?: string;  // 2-100 chars
  email?: string;     // email válido, único
}

// ChangePasswordDTO
interface ChangePasswordDTO {
  currentPassword: string;
  newPassword: string; // min 8 chars
}

// DeleteAccountDTO
interface DeleteAccountDTO {
  password: string; // confirmación explícita
}

// ProfileResponseDTO
interface ProfileResponseDTO {
  id: string;
  firstName: string;
  lastName: string;
  email: string;
  role: {
    id: string;
    name: 'organizador' | 'participante' | 'disertante';
    description: string;
  };
  createdAt: Date;
  updatedAt: Date;
}
```

---

## Plan de Tareas

| # | Tarea | Descripción | Estimación |
|---|-------|-------------|------------|
| T801 | Profile Service - CRUD básico | Implementar `getProfile`, `updateProfile` | 2h |
| T802 | Profile Service - changePassword | Implementar cambio de contraseña con bcrypt | 2h |
| T803 | Profile Service - getMyEvents | Lógica diferenciada por rol (participante/organizador/disertante) | 3h |
| T804 | Profile Service - getMyTalks | Obtener charlas del disertante con filtro upcoming | 1.5h |
| T805 | Profile Service - getMyFeedback | Historial de feedback con datos del evento | 1h |
| T806 | Profile Service - deleteAccount | Lógica de eliminación con transacción y validaciones | 3h |
| T807 | Profile Controller | Implementar todos los métodos del controller | 2.5h |
| T808 | Profile Routes | Configurar rutas y middlewares de auth/rol | 0.5h |
| T809 | Validaciones Joi | Schemas para update, changePassword, deleteAccount | 1.5h |
| T810 | Tests unitarios | Tests de service: cambio de contraseña, getMyEvents por rol, deleteAccount | 4h |
| T811 | Tests integración | Tests de endpoints con distintos roles | 3h |

**Total estimado: 24 horas**

---

## Estrategia de Verificación

### Tests Unitarios
- **Profile Service**:
  - `updateProfile()`: verificar que email duplicado lanza error 400
  - `updateProfile()`: verificar que solo se actualizan los campos enviados
  - `changePassword()`: verificar que contraseña incorrecta lanza error 400
  - `changePassword()`: verificar que nueva contraseña igual a la actual lanza error 400
  - `changePassword()`: verificar que nueva contraseña se guarda como hash bcrypt válido
  - `getMyEvents()`: verificar que participante ve solo sus registros en `event_attendees`
  - `getMyEvents()`: verificar que organizador ve solo sus eventos creados
  - `getMyTalks()`: verificar que solo funciona para disertantes
  - `deleteAccount()`: verificar que organizador con eventos futuros activos no puede eliminar
  - `deleteAccount()`: verificar que la transacción revierte si algún paso falla

### Tests de Integración
- `GET /api/v1/profile`: verificar que no incluye campo `password`
- `PUT /api/v1/profile`: verificar actualización parcial
- `PUT /api/v1/profile/password`: flujo completo de cambio
- `GET /api/v1/profile/events`: verificar respuesta diferente por rol
- `GET /api/v1/profile/talks`: verificar 403 para no-disertantes
- `DELETE /api/v1/profile`: verificar eliminación en cascada con transacción

### Casos de Verificación Manual

| ID | Caso | Resultado Esperado |
|----|------|-------------------|
| V801 | Usuario obtiene su perfil | 200 sin campo `password` |
| V802 | Usuario actualiza email a uno ya existente | 400 EMAIL_ALREADY_IN_USE |
| V803 | Usuario cambia contraseña con actual incorrecta | 400 INVALID_CURRENT_PASSWORD |
| V804 | Usuario cambia contraseña con nueva igual a actual | 400 SAME_PASSWORD |
| V805 | Participante consulta su historial de eventos | 200 con estado attended y certificateAvailable |
| V806 | Disertante consulta sus charlas con `upcoming=true` | Solo charlas con startTime > ahora |
| V807 | Organizador consulta charlas de disertante | 403 FORBIDDEN |
| V808 | Organizador con eventos futuros intenta eliminar cuenta | 400 HAS_ACTIVE_EVENTS |
| V809 | Usuario elimina cuenta con contraseña incorrecta | 400 INVALID_PASSWORD |
| V810 | Usuario elimina cuenta exitosamente | 200, notificaciones y feedback eliminados, token expirado en próximo uso |

---

## Conclusión

El Módulo 8 (Gestión de Perfil de Usuario) completa el ciclo de autogestión del sistema, dando a cada usuario control total sobre su información personal e historial de actividad. Al no requerir nuevos modelos, este módulo es una capa de orquestación que agrega valor sin aumentar la complejidad del esquema de datos, reutilizando de forma coherente toda la infraestructura de los módulos anteriores.

**Integración con módulos anteriores:**
- Módulo 1: Control de acceso por rol para endpoints específicos (charlas solo para disertantes)
- Módulo 2: Fuente de datos de eventos para historial de organizadores y participantes
- Módulo 3: Estado de asistencia (attended, checkInAt) para historial del participante
- Módulo 4: Cálculo de `certificateAvailable` en el historial del participante
- Módulo 5: Historial de feedback enviado por el usuario
- Módulo 7: Eliminación de notificaciones en cascada al borrar la cuenta
