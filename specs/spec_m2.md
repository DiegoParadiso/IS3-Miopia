# Especificación Módulo 2: Gestión de Eventos

**Estado:** En proceso

## Índice
1. [Descripción](#descripción)
2. [Arquitectura](#arquitectura)
3. [Módulos y Responsabilidades](#módulos-y-responsabilidades)
4. [Flujo de Datos](#flujo-de-datos)
5. [Endpoints y Contratos](#endpoints-y-contratos)
6. [Dependencias y Capas](#dependencias-y-capas)
7. [Modelos de Datos](#modelos-de-datos)
8. [Especificación Formal](#especificación-formal)

---

## Descripción

El Módulo 2 implementa la **Gestión de Eventos** del sistema, permitiendo a los organizadores crear y administrar eventos, a los participantes registrarse y asistir, y a los disertantes gestionar sus charlas.

### Objetivos
- CRUD completo de eventos
- Registro de participantes a eventos
- Gestión de charlas dentro de eventos
- Control de capacidad y disponibilidad
- Filtrado y búsqueda de eventos

---

## Arquitectura

El módulo sigue la misma **Arquitectura en Capas** que el Módulo 1:

```
┌─────────────────────────────────────────────┐
│           Cliente (Frontend)                │
└─────────────────┬───────────────────────────┘
                  │ HTTP/REST
┌─────────────────▼───────────────────────────┐
│         Capa de Presentación (API)          │
│  ┌──────────────┐    ┌──────────────┐     │
│  │ Event Routes │───▶│Event Controll│     │
│  └──────────────┘    └──────────────┘     │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Lógica de Negocio           │
│           Event Service                      │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Acceso a Datos               │
│        Event Model + Talk Model              │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Base de Datos (PostgreSQL)          │
└─────────────────────────────────────────────┘
```

---

## Módulos y Responsabilidades

### 1. Event Controller (`src/controllers/event-controller.js`)
**Responsabilidades:**
- Manejar requests HTTP para operaciones de eventos
- Validar datos de entrada
- Delegar lógica a Event Service
- Formatear respuestas

**Métodos:**
- `createEvent(req, res)` - Crear evento (organizador)
- `getAllEvents(req, res)` - Listar eventos (todos)
- `getEventById(req, res)` - Obtener evento (todos)
- `updateEvent(req, res)` - Actualizar evento (organizador)
- `deleteEvent(req, res)` - Eliminar evento (organizador)
- `registerToEvent(req, res)` - Registrar participante (participante)
- `getEventAttendees(req, res)` - Ver asistentes (organizador)

---

### 2. Event Service (`src/services/event-service.js`)
**Responsabilidades:**
- Lógica de negocio de eventos
- Validar capacidad antes de registrar
- Verificar conflictos de horarios
- Gestionar relación eventos-charlas

**Métodos:**
- `create(eventData, organizerId)` - Crear evento
- `findAll(filters)` - Buscar con filtros
- `findById(id)` - Buscar por ID con relaciones
- `update(id, eventData)` - Actualizar evento
- `delete(id)` - Eliminar evento
- `registerUser(eventId, userId)` - Registrar usuario
- `getAttendees(eventId)` - Obtener asistentes

---

### 3. Event Model (`src/models/Event.js`)
**Campos:**
- `id`: UUID
- `title`: string
- `description`: text
- `date`: datetime
- `location`: string
- `capacity`: integer
- `organizerId`: UUID (FK a User)
- `status`: enum ['draft', 'published', 'cancelled']

---

### 4. Talk Model (`src/models/Talk.js`)
**Campos:**
- `id`: UUID
- `title`: string
- `description`: text
- `speakerId`: UUID (FK a User - disertante)
- `eventId`: UUID (FK a Event)
- `startTime`: datetime
- `endTime`: datetime

---

## Flujo de Datos

### Caso: Registrar participante a evento

```
[1] Participante → POST /api/v1/events/:id/register + Token
                ↓
[2] Auth Middleware → Verifica JWT → Rol: participante
                ↓
[3] Event Controller → registerToEvent(req, res)
                ↓
[4] Event Service → registerUser(eventId, userId)
                ↓
[5] Validaciones:
    - ¿Evento existe? → Event.findByPk(eventId, { include: 'attendees' })
    - ¿Capacidad disponible? → attendees.length < capacity
    - ¿No está ya registrado? → !attendees.includes(userId)
                ↓
[6] Registro → EventAttendee.create({ eventId, userId })
                ↓
[7] PostgreSQL → INSERT INTO event_attendees
                ↓
[8] Respuesta → 200 OK "Registro exitoso"
```

---

## Endpoints y Contratos

Referencia completa en [contracts.md](../contracts.md#modulo-2-gestión-de-eventos)

### Resumen de Endpoints

| Método | Endpoint | Descripción | Rol Requerido |
|--------|----------|-------------|---------------|
| POST | `/api/v1/events` | Crear evento | Organizador |
| GET | `/api/v1/events` | Listar eventos | Todos (auth) |
| GET | `/api/v1/events/:id` | Obtener evento | Todos (auth) |
| PUT | `/api/v1/events/:id` | Actualizar evento | Organizador |
| DELETE | `/api/v1/events/:id` | Eliminar evento | Organizador |
| POST | `/api/v1/events/:id/register` | Registrarse | Participante |
| GET | `/api/v1/events/:id/attendees` | Ver asistentes | Organizador |
| POST | `/api/v1/events/:id/talks` | Crear charla | Disertante |
| PUT | `/api/v1/talks/:id` | Editar charla | Disertante |

---

## Dependencias y Capas

```
Módulo 2: Event Management
├── Dependencias Internas:
│   ├── User Model (organizador, asistentes)
│   ├── Role Model (verificación de permisos)
│   ├── Talk Model (charlas del evento)
│   └── Auth Middleware
│
├── Dependencias del Módulo 1:
│   └── Role checking (solo organizador puede crear eventos)
│
└── Base de Datos:
    └── PostgreSQL
        ├── Tabla: events
        ├── Tabla: talks
        ├── Tabla: event_attendees (pivot)
        └── Tabla: users (FKs)
```

---

## Modelos de Datos

### Event Model
```javascript
const Event = sequelize.define('Event', {
  id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
  title: { type: DataTypes.STRING(255), allowNull: false },
  description: { type: DataTypes.TEXT },
  date: { type: DataTypes.DATE, allowNull: false },
  location: { type: DataTypes.STRING(255), allowNull: false },
  capacity: { type: DataTypes.INTEGER, allowNull: false, validate: { min: 1 } },
  organizerId: { type: DataTypes.UUID, allowNull: false },
  status: { 
    type: DataTypes.ENUM('draft', 'published', 'cancelled'),
    defaultValue: 'draft'
  }
});

// Relaciones
Event.belongsTo(User, { foreignKey: 'organizerId', as: 'organizer' });
Event.belongsToMany(User, { through: 'event_attendees', as: 'attendees' });
Event.hasMany(Talk, { foreignKey: 'eventId', as: 'talks' });
```

### Talk Model
```javascript
const Talk = sequelize.define('Talk', {
  id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
  title: { type: DataTypes.STRING(255), allowNull: false },
  description: { type: DataTypes.TEXT },
  speakerId: { type: DataTypes.UUID, allowNull: false },
  eventId: { type: DataTypes.UUID, allowNull: false },
  startTime: { type: DataTypes.DATE, allowNull: false },
  endTime: { type: DataTypes.DATE, allowNull: false }
});

// Relaciones
Talk.belongsTo(User, { foreignKey: 'speakerId', as: 'speaker' });
Talk.belongsTo(Event, { foreignKey: 'eventId', as: 'event' });
```

---

## Especificación Formal

### Contratos de Datos

#### EventCreateDTO
```typescript
interface EventCreateDTO {
  title: string; // 3-255 chars
  description?: string;
  date: Date; // debe ser futura
  location: string; // 3-255 chars
  capacity: number; // min: 1
}
```

#### EventUpdateDTO
```typescript
interface EventUpdateDTO {
  title?: string;
  description?: string;
  date?: Date;
  location?: string;
  capacity?: number;
  status?: 'draft' | 'published' | 'cancelled';
}
```

#### EventResponseDTO
```typescript
interface EventResponseDTO {
  id: string;
  title: string;
  description: string;
  date: Date;
  location: string;
  capacity: number;
  organizer: {
    id: string;
    firstName: string;
    lastName: string;
    email: string;
  };
  attendees: User[];
  talks: Talk[];
  status: string;
  createdAt: Date;
  updatedAt: Date;
}
```

---

### Reglas de Negocio

1. **RN101**: Solo organizadores pueden crear eventos
2. **RN102**: La fecha del evento debe ser futura
3. **RN103**: La capacidad debe ser mayor a 0
4. **RN104**: No se pueden registrar más usuarios que la capacidad
5. **RN105**: Un usuario no puede registrarse dos veces al mismo evento
6. **RN106**: Solo el organizador del evento puede editarlo/eliminarlo
7. **RN107**: Las charlas solo pueden ser creadas por disertantes
8. **RN108**: Los horarios de charlas no deben solaparse en el mismo evento

---

### Casos de Uso

#### CU101: Crear Evento
**Actor**: Organizador  
**Flujo**:
1. Organizador envía datos del evento
2. Sistema valida datos y fecha futura
3. Sistema crea evento con estado 'draft'
4. Sistema retorna evento creado

---

#### CU102: Registrarse a Evento
**Actor**: Participante  
**Flujo**:
1. Participante solicita registro a evento
2. Sistema verifica que evento exista y esté publicado
3. Sistema verifica capacidad disponible
4. Sistema verifica que no esté ya registrado
5. Sistema registra participante

**Excepciones**:
- E1: Evento lleno → Error 400
- E2: Ya registrado → Error 400

---

#### CU103: Crear Charla en Evento
**Actor**: Disertante  
**Flujo**:
1. Disertante envía datos de la charla
2. Sistema verifica que evento exista
3. Sistema verifica que horarios no se solapen
4. Sistema crea charla asociada al disertante

---

## Conclusión

El Módulo 2 complementa al Módulo 1 proporcionando la funcionalidad core del sistema: la gestión de eventos. La integración con el sistema de roles asegura que cada tipo de usuario pueda realizar solo las acciones correspondientes a su rol.

**Integración con Módulo 1:**
- Verificación de roles en cada endpoint
- Asignación de organizadores al crear eventos
- Registro de participantes y creación de charlas por disertantes
