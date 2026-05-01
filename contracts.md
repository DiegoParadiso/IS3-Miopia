# Contratos de API - IS3 Miopia

## Base URL
```
http://localhost:3000/api/v1
```

---

## Módulo 1: Gestión de Roles

### 1.1 Obtener todos los roles
**GET** `/roles`

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
      "id": "uuid",
      "name": "organizador",
      "description": "Gestiona y administra eventos",
      "permissions": ["event:create", "event:update", "event:delete", "user:manage"],
      "createdAt": "2026-04-30T00:00:00.000Z",
      "updatedAt": "2026-04-30T00:00:00.000Z"
    },
    {
      "id": "uuid",
      "name": "participante",
      "description": "Asiste a eventos",
      "permissions": ["event:view", "event:register"],
      "createdAt": "2026-04-30T00:00:00.000Z",
      "updatedAt": "2026-04-30T00:00:00.000Z"
    },
    {
      "id": "uuid",
      "name": "disertante",
      "description": "Expone charlas en eventos",
      "permissions": ["event:view", "talk:create", "talk:update"],
      "createdAt": "2026-04-30T00:00:00.000Z",
      "updatedAt": "2026-04-30T00:00:00.000Z"
    }
  ],
  "message": "Roles obtenidos exitosamente"
}
```

---

### 1.2 Obtener rol por ID
**GET** `/roles/:id`

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
    "name": "organizador",
    "description": "Gestiona y administra eventos",
    "permissions": ["event:create", "event:update", "event:delete", "user:manage"],
    "createdAt": "2026-04-30T00:00:00.000Z",
    "updatedAt": "2026-04-30T00:00:00.000Z"
  },
  "message": "Rol obtenido exitosamente"
}
```

**Response 404:**
```json
{
  "success": false,
  "error": {
    "code": "ROLE_NOT_FOUND",
    "message": "Rol no encontrado"
  }
}
```

---

### 1.3 Crear rol (Solo organizador)
**POST** `/roles`

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "name": "organizador",
  "description": "Gestiona y administra eventos",
  "permissions": ["event:create", "event:update", "event:delete", "user:manage"]
}
```

**Validaciones:**
- `name`: required, enum ['organizador', 'participante', 'disertante']
- `description`: required, string, max 255 chars
- `permissions`: required, array of strings

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "organizador",
    "description": "Gestiona y administra eventos",
    "permissions": ["event:create", "event:update", "event:delete", "user:manage"],
    "createdAt": "2026-04-30T00:00:00.000Z",
    "updatedAt": "2026-04-30T00:00:00.000Z"
  },
  "message": "Rol creado exitosamente"
}
```

**Response 400:**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Datos inválidos",
    "details": [
      {
        "field": "name",
        "message": "El nombre debe ser organizador, participante o disertante"
      }
    ]
  }
}
```

---

### 1.4 Actualizar rol (Solo organizador)
**PUT** `/roles/:id`

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "description": "Nueva descripción",
  "permissions": ["event:create", "event:update"]
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "organizador",
    "description": "Nueva descripción",
    "permissions": ["event:create", "event:update"],
    "createdAt": "2026-04-30T00:00:00.000Z",
    "updatedAt": "2026-04-30T00:00:00.000Z"
  },
  "message": "Rol actualizado exitosamente"
}
```

---

### 1.5 Eliminar rol (Solo organizador)
**DELETE** `/roles/:id`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "success": true,
  "message": "Rol eliminado exitosamente"
}
```

**Response 400:**
```json
{
  "success": false,
  "error": {
    "code": "ROLE_IN_USE",
    "message": "No se puede eliminar un rol que tiene usuarios asignados"
  }
}
```

---

### 1.6 Asignar rol a usuario (Solo organizador)
**POST** `/roles/assign`

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "userId": "uuid",
  "roleId": "uuid"
}
```

**Validaciones:**
- `userId`: required, UUID válido, debe existir
- `roleId`: required, UUID válido, debe existir

**Response 200:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "firstName": "Juan",
      "lastName": "Pérez",
      "role": {
        "id": "uuid",
        "name": "disertante",
        "description": "Expone charlas en eventos"
      }
    }
  },
  "message": "Rol asignado exitosamente"
}
```

**Response 404:**
```json
{
  "success": false,
  "error": {
    "code": "USER_OR_ROLE_NOT_FOUND",
    "message": "Usuario o rol no encontrado"
  }
}
```

---

## Módulo 2: Gestión de Eventos

### 2.1 Crear evento (Solo organizador)
**POST** `/events`

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "title": "Conferencia de Tecnología",
  "description": "Evento anual de tecnología",
  "date": "2026-06-15T10:00:00.000Z",
  "location": "Auditorio Principal",
  "capacity": 100
}
```

---

### 2.2 Listar eventos (Todos los roles autenticados)
**GET** `/events`

**Headers:**
```
Authorization: Bearer <token>
```

---

### 2.3 Obtener evento por ID (Todos los roles autenticados)
**GET** `/events/:id`

**Headers:**
```
Authorization: Bearer <token>
```

---

### 2.4 Actualizar evento (Solo organizador)
**PUT** `/events/:id`

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

---

### 2.5 Eliminar evento (Solo organizador)
**DELETE** `/events/:id`

**Headers:**
```
Authorization: Bearer <token>
```

---

### 2.6 Registrar participante a evento (Participante)
**POST** `/events/:id/register`

**Headers:**
```
Authorization: Bearer <token>
```

---

## Autenticación

### Login
**POST** `/auth/login`

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "firstName": "Juan",
      "lastName": "Pérez",
      "role": "organizador"
    }
  },
  "message": "Login exitoso"
}
```

### Registro
**POST** `/auth/register`

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "password123",
  "firstName": "Juan",
  "lastName": "Pérez",
  "roleId": "uuid"
}
```
