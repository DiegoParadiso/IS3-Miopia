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

---

## Módulo 3: Acreditación

### 3.1 Registrar Check-In
**POST** `/events/:id/check-in`

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

---

### 3.2 Ver Estado de Check-In
**GET** `/events/:id/check-in/status`

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

---

### 3.3 Listar Acreditaciones del Evento
**GET** `/events/:id/accreditation`

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

## Módulo 4: Certificados

### 4.1 Descargar Certificado
**GET** `/certificates/:eventId/download`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="certificado-{eventId}.pdf"

[Binary PDF content]
```

**Response 403:**
```json
{
  "success": false,
  "error": {
    "code": "NOT_ELIGIBLE",
    "message": "No es elegible para obtener certificado. Debe haber asistido al evento y el evento debe estar finalizado."
  }
}
```

---

### 4.2 Listar Certificados Disponibles
**GET** `/certificates`

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
      "eventId": "uuid",
      "eventTitle": "Conferencia de Tecnología",
      "eventDate": "2026-06-15T10:00:00.000Z",
      "location": "Auditorio Principal",
      "checkInAt": "2026-06-15T10:30:00.000Z",
      "certificateAvailable": true
    }
  ],
  "message": "Certificados disponibles obtenidos"
}
```

---

## Módulo 5: Encuestas y Comentarios

### 5.1 Enviar Feedback
**POST** `/events/:id/feedback`

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "rating": 5,
  "comment": "Excelente evento, muy buena organización"
}
```

**Validaciones:**
- `rating`: required, integer, min: 1, max: 5
- `comment`: optional, string, max: 1000 chars

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "eventId": "uuid",
    "userId": "uuid",
    "rating": 5,
    "comment": "Excelente evento, muy buena organización",
    "createdAt": "2026-06-16T14:00:00.000Z"
  },
  "message": "Feedback enviado exitosamente"
}
```

**Response 403:**
```json
{
  "success": false,
  "error": {
    "code": "NOT_ATTENDEE",
    "message": "Solo los asistentes al evento pueden enviar feedback"
  }
}
```

---

### 5.2 Listar Feedbacks del Evento
**GET** `/events/:id/feedback`

**Headers:**
```
Authorization: Bearer <token>
```

**Query Params (opcionales):**
- `page`: número de página (default: 1)
- `limit`: items por página (default: 20, max: 100)
- `rating`: filtrar por calificación (1-5)

**Rol requerido:** Organizador

**Response 200:**
```json
{
  "success": true,
  "data": {
    "feedbacks": [
      {
        "id": "uuid",
        "rating": 5,
        "comment": "Muy bueno",
        "user": {
          "firstName": "Juan",
          "lastName": "Pérez"
        },
        "createdAt": "2026-06-16T14:00:00.000Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 45
    }
  },
  "message": "Feedbacks obtenidos exitosamente"
}
```

---

### 5.3 Resumen de Métricas
**GET** `/events/:id/feedback/summary`

**Headers:**
```
Authorization: Bearer <token>
```

**Rol requerido:** Organizador

**Response 200:**
```json
{
  "success": true,
  "data": {
    "eventId": "uuid",
    "eventTitle": "Conferencia de Tecnología",
    "averageRating": 4.3,
    "totalFeedbacks": 45,
    "ratingDistribution": {
      "1": 2,
      "2": 3,
      "3": 5,
      "4": 15,
      "5": 20
    },
    "recentComments": [
      {
        "rating": 5,
        "comment": "Excelente organización",
        "createdAt": "2026-06-16T14:00:00.000Z"
      }
    ],
    "surveyResponseRate": "60%"
  },
  "message": "Resumen obtenido exitosamente"
}
```

---

### 5.4 Crear Encuesta
**POST** `/events/:id/surveys`

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "title": "Encuesta de Satisfacción - Conferencia 2026",
  "questions": [
    {
      "questionText": "¿Cómo calificarías la organización del evento?",
      "questionType": "rating",
      "required": true,
      "order": 1
    },
    {
      "questionText": "¿Qué tema te resultó más interesante?",
      "questionType": "multiple_choice",
      "options": ["IA", "Web Development", "DevOps", "Security"],
      "required": true,
      "order": 2
    },
    {
      "questionText": "¿Algún comentario adicional?",
      "questionType": "text",
      "required": false,
      "order": 3
    }
  ]
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "eventId": "uuid",
    "title": "Encuesta de Satisfacción - Conferencia 2026",
    "isActive": true,
    "questions": [
      {
        "id": "uuid",
        "questionText": "¿Cómo calificarías la organización del evento?",
        "questionType": "rating",
        "required": true,
        "order": 1
      }
    ],
    "createdAt": "2026-06-16T10:00:00.000Z"
  },
  "message": "Encuesta creada exitosamente"
}
```

---

### 5.5 Obtener Encuesta Activa
**GET** `/events/:id/surveys`

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
    "title": "Encuesta de Satisfacción - Conferencia 2026",
    "isActive": true,
    "questions": [
      {
        "id": "uuid",
        "questionText": "¿Cómo calificarías la organización del evento?",
        "questionType": "rating",
        "required": true,
        "order": 1
      }
    ]
  },
  "message": "Encuesta obtenida exitosamente"
}
```

---

### 5.6 Enviar Respuesta de Encuesta
**POST** `/events/:id/surveys/response`

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "answers": [
    {
      "questionId": "uuid",
      "answer": 5
    },
    {
      "questionId": "uuid",
      "answer": "IA"
    },
    {
      "questionId": "uuid",
      "answer": "Muy buena organización, seguiría asistiendo"
    }
  ]
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "surveyId": "uuid",
    "userId": "uuid",
    "eventId": "uuid",
    "submittedAt": "2026-06-16T15:00:00.000Z"
  },
  "message": "Respuesta de encuesta enviada exitosamente"
}
```

---

### 5.7 Resultados de Encuesta
**GET** `/events/:id/surveys/results`

**Headers:**
```
Authorization: Bearer <token>
```

**Rol requerido:** Organizador

**Response 200:**
```json
{
  "success": true,
  "data": {
    "surveyId": "uuid",
    "title": "Encuesta de Satisfacción - Conferencia 2026",
    "totalResponses": 27,
    "questions": [
      {
        "questionId": "uuid",
        "questionText": "¿Cómo calificarías la organización del evento?",
        "questionType": "rating",
        "results": {
          "average": 4.5,
          "distribution": { "1": 0, "2": 1, "3": 3, "4": 8, "5": 15 }
        }
      },
      {
        "questionId": "uuid",
        "questionText": "¿Qué tema te resultó más interesante?",
        "questionType": "multiple_choice",
        "results": {
          "IA": 12,
          "Web Development": 8,
          "DevOps": 4,
          "Security": 3
        }
      }
    ]
  },
  "message": "Resultados de encuesta obtenidos exitosamente"
}
```

---

## Módulo 6: Informes y Agenda

### 6.1 Descargar Agenda del Evento
**GET** `/events/:id/agenda/download`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="agenda-{eventId}.pdf"

[Binary PDF content]
```

**Response 400:**
```json
{
  "success": false,
  "error": {
    "code": "EVENT_NOT_PUBLISHED",
    "message": "La agenda solo está disponible para eventos publicados"
  }
}
```

---

### 6.2 Informe de Asistencia
**GET** `/events/:id/reports/attendance`

**Headers:**
```
Authorization: Bearer <token>
```

**Rol requerido:** Organizador

**Response 200:**
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="asistencia-{eventId}.pdf"

[Binary PDF content]
```

**Response 403:**
```json
{
  "success": false,
  "error": {
    "code": "NOT_ORGANIZER",
    "message": "Solo el organizador del evento puede generar este informe"
  }
}
```

---

### 6.3 Informe Estadístico
**GET** `/events/:id/reports/statistics`

**Headers:**
```
Authorization: Bearer <token>
```

**Rol requerido:** Organizador

**Response 200:**
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="estadisticas-{eventId}.pdf"

[Binary PDF content]
```

---

### 6.4 Listar Informes Disponibles
**GET** `/events/:id/reports`

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
      "id": "attendance",
      "name": "Informe de Asistencia",
      "description": "Lista completa de registrados y asistentes al evento",
      "endpoint": "/api/v1/events/:id/reports/attendance",
      "available": true,
      "requirements": "Evento finalizado"
    },
    {
      "id": "statistics",
      "name": "Informe Estadístico",
      "description": "Métricas consolidadas del evento incluyendo asistencia, satisfacción y encuestas",
      "endpoint": "/api/v1/events/:id/reports/statistics",
      "available": true,
      "requirements": "Evento finalizado"
    },
    {
      "id": "agenda",
      "name": "Agenda del Evento",
      "description": "Programa de charlas con horarios y disertantes",
      "endpoint": "/api/v1/events/:id/agenda/download",
      "available": true,
      "requirements": "Evento publicado"
    }
  ],
  "message": "Informes disponibles obtenidos exitosamente"
}
```
