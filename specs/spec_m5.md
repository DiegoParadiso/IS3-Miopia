# Especificación Módulo 5: Encuestas y Comentarios Post-Evento

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

El Módulo 5 implementa el sistema de **Encuestas y Comentarios Post-Evento** que permite a los participantes que asistieron a un evento dejar valoraciones, comentarios y responder encuestas de satisfacción. Los organizadores pueden ver los resultados consolidados y métricas de satisfacción.

---

## Objetivos y Contexto

- Permitir a los asistentes dejar feedback sobre el evento
- Implementar un sistema de calificación numérica (1-5 estrellas)
- Permitir comentarios textuales opcionales
- Permitir a los organizadores crear encuestas con preguntas específicas
- Generar métricas de satisfacción para el organizador
- Solo usuarios que hicieron check-in pueden dejar feedback

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
│  │  Feedback Routes  │→│Feedback Control│ │
│  └──────────────────┘  └────────────────┘ │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Lógica de Negocio           │
│         Feedback Service                    │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Acceso a Datos               │
│  Survey, SurveyQuestion, Feedback Models    │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Base de Datos (PostgreSQL)          │
└─────────────────────────────────────────────┘
```

---

## Módulos y Responsabilidades

### 1. Feedback Controller (`src/controllers/feedback-controller.js`)
**Responsabilidades:**
- Manejar requests HTTP para operaciones de feedback y encuestas
- Validar datos de entrada
- Delegar lógica a Feedback Service
- Formatear respuestas

**Métodos:**
- `submitFeedback(req, res)` - Enviar feedback/calificación a un evento
- `getEventFeedback(req, res)` - Obtener todos los feedbacks de un evento (organizador)
- `getEventFeedbackSummary(req, res)` - Obtener resumen/métricas de un evento (organizador)
- `getSurveyByEvent(req, res)` - Obtener encuesta activa de un evento
- `submitSurveyResponse(req, res)` - Enviar respuestas de encuesta
- `createSurvey(req, res)` - Crear encuesta para un evento (organizador)

---

### 2. Feedback Service (`src/services/feedback-service.js`)
**Responsabilidades:**
- Lógica de negocio de feedback y encuestas
- Validar elegibilidad (solo asistentes con check-in)
- Calcular métricas de satisfacción (promedio, distribución)
- Gestionar ciclo de vida de encuestas

**Métodos:**
- `submitFeedback(eventId, userId, rating, comment)` - Registrar feedback
- `getEventFeedback(eventId, organizerId)` - Obtener feedbacks del evento
- `getFeedbackSummary(eventId)` - Calcular métricas
- `createSurvey(organizerId, eventId, surveyData)` - Crear encuesta
- `getActiveSurvey(eventId)` - Obtener encuesta activa
- `submitSurveyResponse(eventId, userId, answers)` - Registrar respuestas

---

### 3. Feedback Model (`src/models/Feedback.js`)
**Campos:**
- `id`: UUID
- `eventId`: UUID (FK a Event)
- `userId`: UUID (FK a User)
- `rating`: integer (1-5)
- `comment`: text (opcional, max 1000 chars)
- `createdAt`: DATE
- `updatedAt`: DATE

---

### 4. Survey Model (`src/models/Survey.js`)
**Campos:**
- `id`: UUID
- `eventId`: UUID (FK a Event, unique)
- `title`: string
- `isActive`: boolean (default: true)
- `createdAt`: DATE
- `updatedAt`: DATE

---

### 5. SurveyQuestion Model (`src/models/SurveyQuestion.js`)
**Campos:**
- `id`: UUID
- `surveyId`: UUID (FK a Survey)
- `questionText`: text
- `questionType`: enum ['rating', 'text', 'multiple_choice']
- `options`: JSON array (solo para multiple_choice)
- `order`: integer
- `required`: boolean (default: false)

---

### 6. SurveyResponse Model (`src/models/SurveyResponse.js`)
**Campos:**
- `id`: UUID
- `surveyId`: UUID (FK a Survey)
- `userId`: UUID (FK a User)
- `eventId`: UUID (FK a Event)
- `answers`: JSON array [{questionId, answer}]
- `createdAt`: DATE

---

## Flujo de Datos

### Caso 1: Enviar Feedback/Calificación

```
[1] Participante → POST /api/v1/events/:id/feedback + Token + Body {rating, comment}
                ↓
[2] Auth Middleware → Verifica JWT → extrae userId
                ↓
[3] Feedback Controller → submitFeedback(req, res)
                ↓
[4] Feedback Service → submitFeedback(eventId, userId, rating, comment)
                ↓
[5] Validaciones:
    - ¿Evento existe y está finalizado?
    - ¿Usuario registrado al evento?
    - ¿Usuario hizo check-in (attended = true)?
    - ¿Usuario ya envió feedback para este evento?
    - ¿Rating entre 1 y 5?
                ↓
[6] Feedback Model → Feedback.create({ eventId, userId, rating, comment })
                ↓
[7] PostgreSQL → INSERT INTO feedback
                ↓
[8] Respuesta → 201 Created + Feedback creado
```

---

### Caso 2: Crear Encuesta (Organizador)

```
[1] Organizador → POST /api/v1/events/:id/surveys + Token + Body
                ↓
[2] Auth + Role Auth → Verifica JWT y rol organizador
                ↓
[3] Feedback Controller → createSurvey(req, res)
                ↓
[4] Feedback Service → createSurvey(organizerId, eventId, surveyData)
                ↓
[5] Validaciones:
    - ¿Organizador es dueño del evento?
    - ¿Evento existe?
    - ¿Ya existe encuesta para este evento?
                ↓
[6] Survey Model + SurveyQuestion Model → Transacción para crear encuesta + preguntas
                ↓
[7] PostgreSQL → INSERT INTO surveys + INSERT INTO survey_questions
                ↓
[8] Respuesta → 201 Created + Encuesta creada
```

---

### Caso 3: Ver Resumen de Feedback (Organizador)

```
[1] Organizador → GET /api/v1/events/:id/feedback/summary + Token
                ↓
[2] Auth + Role Auth → Verifica JWT y permisos
                ↓
[3] Feedback Controller → getEventFeedbackSummary(req, res)
                ↓
[4] Feedback Service → getFeedbackSummary(eventId)
                ↓
[5] Cálculos:
    - Promedio general de rating
    - Distribución por estrellas (1★, 2★, 3★, 4★, 5★)
    - Total de feedbacks recibidos
    - Lista de comentarios más recientes
    - Resultados de encuestas (si existen)
                ↓
[6] Respuesta → 200 OK + Resumen con métricas
```

---

## Historias de Usuario y Criterios de Aceptación

### HU501: Calificar un Evento Finalizado
**Como** participante que asistió a un evento  
**Quiero** poder dejar una calificación y comentario  
**Para** expresar mi opinión sobre el evento  

**Criterios de Aceptación:**
- Dado que asistí al evento (check-in realizado), puedo enviar un rating de 1 a 5
- Dado que ya envié feedback para ese evento, no puedo enviar otro
- Dado que no asistí al evento, no puedo enviar feedback
- El campo comentario es opcional
- El rating es obligatorio

---

### HU502: Crear Encuesta de Satisfacción
**Como** organizador de un evento  
**Quiero** crear una encuesta con preguntas personalizadas  
**Para** obtener feedback detallado de los asistentes  

**Criterios de Aceptación:**
- Dado que soy organizador del evento, puedo crear una encuesta
- La encuesta puede tener preguntas de tipo rating, texto y opción múltiple
- Cada pregunta puede ser marcada como requerida u opcional
- Solo puedo crear una encuesta por evento
- Las preguntas tienen un orden definido

---

### HU503: Responder Encuesta
**Como** participante que asistió a un evento  
**Quiero** responder la encuesta del evento  
**Para** dar feedback detallado al organizador  

**Criterios de Aceptación:**
- Dado que asistí al evento, puedo ver y responder la encuesta
- Debo responder todas las preguntas marcadas como requeridas
- Solo puedo enviar una respuesta por encuesta
- Dado que la encuesta no está activa, no puedo responderla

---

### HU504: Ver Resumen de Feedback
**Como** organizador de un evento  
**Quiero** ver las métricas de satisfacción del evento  
**Para** evaluar la calidad del evento y mejorar futuros eventos  

**Criterios de Aceptación:**
- Dado que soy organizador del evento, puedo ver el promedio de calificaciones
- Puedo ver la distribución de calificaciones por estrellas
- Puedo ver la lista de comentarios de los participantes
- Puedo ver los resultados de la encuesta (si existe)
- Solo el organizador puede acceder a este resumen

---

## Requisitos Funcionales y Reglas de Negocio

### Requisitos Funcionales
- **RF501**: El sistema debe permitir enviar calificaciones de 1 a 5 estrellas
- **RF502**: El sistema debe permitir agregar un comentario textual opcional (max 1000 caracteres)
- **RF503**: Solo usuarios que hicieron check-in al evento pueden enviar feedback
- **RF504**: Un usuario solo puede enviar un feedback por evento
- **RF505**: Los organizadores pueden crear encuestas con preguntas de distintos tipos
- **RF506**: Los participantes pueden responder encuestas una sola vez
- **RF507**: Los organizadores pueden ver métricas de satisfacción consolidadas

---

### Reglas de Negocio

| Código | Regla |
|--------|-------|
| **RN501** | Solo usuarios con `attended = true` en `event_attendees` pueden enviar feedback |
| **RN502** | El rating debe ser un entero entre 1 y 5 (inclusive) |
| **RN503** | Un usuario solo puede enviar un feedback por evento (rating único) |
| **RN504** | El evento debe estar finalizado (date < fecha actual) para recibir feedback |
| **RN505** | Solo el organizador del evento puede crear encuestas |
| **RN506** | Solo puede existir una encuesta activa por evento |
| **RN507** | Un usuario solo puede responder una encuesta una vez |
| **RN508** | Las preguntas requeridas deben tener respuesta para enviar la encuesta |
| **RN509** | Las preguntas de tipo `multiple_choice` deben tener al menos 2 opciones |
| **RN510** | Solo el organizador del evento puede ver el resumen de feedback |
| **RN511** | El campo `comment` del feedback es opcional, máximo 1000 caracteres |
| **RN512** | Una encuesta puede ser desactivada por el organizador (isActive = false) |

---

## Restricciones Técnicas

- Seguir arquitectura en capas (Routes → Controller → Service → Model)
- Usar Sequelize ORM para acceso a datos
- Validar datos de entrada con Joi
- Las operaciones de creación de encuesta + preguntas deben usar transacciones
- Las respuestas de encuesta se almacenan como JSON para flexibilidad
- Los comentarios de feedback no requieren sanitización especial (se almacenan como texto plano)
- Calcular métricas de forma eficiente usando agregaciones de base de datos (AVG, COUNT, GROUP BY)
- Seguir formato de respuesta definido en Project.md
- Usar autenticación JWT en todos los endpoints

---

## Endpoints y Contratos

Referencia completa en [contracts.md](../contracts.md#modulo-5-encuestas-y-comentarios)

### Resumen de Endpoints

| Método | Endpoint | Descripción | Rol Requerido |
|--------|----------|-------------|---------------|
| POST | `/api/v1/events/:id/feedback` | Enviar calificación y comentario | Participante/Disertante (asistente) |
| GET | `/api/v1/events/:id/feedback` | Listar feedbacks del evento | Organizador |
| GET | `/api/v1/events/:id/feedback/summary` | Resumen de métricas | Organizador |
| POST | `/api/v1/events/:id/surveys` | Crear encuesta | Organizador |
| GET | `/api/v1/events/:id/surveys` | Obtener encuesta activa | Todos (autenticados) |
| POST | `/api/v1/events/:id/surveys/response` | Enviar respuesta de encuesta | Participante/Disertante (asistente) |
| GET | `/api/v1/events/:id/surveys/results` | Resultados de encuesta | Organizador |

---

### 5.1 Enviar Feedback

**POST** `/api/v1/events/:id/feedback`

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

**Response 400:**
```json
{
  "success": false,
  "error": {
    "code": "FEEDBACK_ALREADY_SUBMITTED",
    "message": "Ya enviaste feedback para este evento"
  }
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

**GET** `/api/v1/events/:id/feedback`

**Headers:**
```
Authorization: Bearer <token>
```

**Query Params (opcionales):**
- `page`: número de página (default: 1)
- `limit`: items por página (default: 20, max: 100)
- `rating`: filtrar por calificación (1-5)

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

**GET** `/api/v1/events/:id/feedback/summary`

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

**POST** `/api/v1/events/:id/surveys`

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

**Validaciones:**
- `title`: required, string, max: 255 chars
- `questions`: required, array, min: 1, max: 20
- `questions[].questionText`: required, string, max: 500 chars
- `questions[].questionType`: required, enum ['rating', 'text', 'multiple_choice']
- `questions[].options`: required si questionType es 'multiple_choice', min: 2 opciones
- `questions[].order`: required, integer, único dentro de la encuesta

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

**Response 400:**
```json
{
  "success": false,
  "error": {
    "code": "SURVEY_ALREADY_EXISTS",
    "message": "Ya existe una encuesta para este evento"
  }
}
```

---

### 5.5 Obtener Encuesta Activa

**GET** `/api/v1/events/:id/surveys`

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

**Response 404:**
```json
{
  "success": false,
  "error": {
    "code": "SURVEY_NOT_FOUND",
    "message": "No existe encuesta activa para este evento"
  }
}
```

---

### 5.6 Enviar Respuesta de Encuesta

**POST** `/api/v1/events/:id/surveys/response`

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

**Validaciones:**
- `answers`: required, array
- Cada answer debe incluir `questionId` y `answer`
- Todas las preguntas requeridas deben tener respuesta
- El tipo de respuesta debe coincidir con el tipo de pregunta

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

**Response 400:**
```json
{
  "success": false,
  "error": {
    "code": "SURVEY_ALREADY_ANSWERED",
    "message": "Ya respondiste esta encuesta"
  }
}
```

---

### 5.7 Resultados de Encuesta (Organizador)

**GET** `/api/v1/events/:id/surveys/results`

**Headers:**
```
Authorization: Bearer <token>
```

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

## Dependencias y Capas

```
Módulo 5: Encuestas y Comentarios
├── Dependencias Internas:
│   ├── User Model (autor del feedback)
│   ├── Event Model (evento evaluado)
│   ├── EventAttendee Model (verificación de asistencia)
│   └── Auth Middleware
│
├── Dependencias de Módulos Anteriores:
│   ├── Módulo 1: Role checking (organizador gestiona encuestas)
│   ├── Módulo 2: Event validation
│   └── Módulo 3: Accreditation check (attended = true)
│
├── Dependencias Externas:
│   ├── express (routing)
│   ├── sequelize (ORM)
│   ├── joi (validación)
│   └── jsonwebtoken (autenticación)
│
└── Base de Datos:
    └── PostgreSQL
        ├── Tabla: feedback
        ├── Tabla: surveys
        ├── Tabla: survey_questions
        └── Tabla: survey_responses
```

---

## Modelo de Datos

### Feedback Model
```javascript
// src/models/Feedback.js
const { DataTypes } = require('sequelize');
const { sequelize } = require('../config/database');

const Feedback = sequelize.define('Feedback', {
  id: {
    type: DataTypes.UUID,
    defaultValue: DataTypes.UUIDV4,
    primaryKey: true
  },
  eventId: {
    type: DataTypes.UUID,
    allowNull: false
  },
  userId: {
    type: DataTypes.UUID,
    allowNull: false
  },
  rating: {
    type: DataTypes.INTEGER,
    allowNull: false,
    validate: {
      min: 1,
      max: 5
    }
  },
  comment: {
    type: DataTypes.TEXT,
    allowNull: true,
    validate: {
      len: [0, 1000]
    }
  }
}, {
  tableName: 'feedback',
  timestamps: true,
  indexes: [
    { unique: true, fields: ['eventId', 'userId'] }
  ]
});

// Relaciones
Feedback.belongsTo(Event, { foreignKey: 'eventId', as: 'event' });
Feedback.belongsTo(User, { foreignKey: 'userId', as: 'user' });

module.exports = Feedback;
```

### Survey Model
```javascript
// src/models/Survey.js
const Survey = sequelize.define('Survey', {
  id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
  eventId: {
    type: DataTypes.UUID,
    allowNull: false,
    unique: true
  },
  title: { type: DataTypes.STRING(255), allowNull: false },
  isActive: { type: DataTypes.BOOLEAN, defaultValue: true }
}, {
  tableName: 'surveys',
  timestamps: true
});

// Relaciones
Survey.belongsTo(Event, { foreignKey: 'eventId', as: 'event' });
Survey.hasMany(SurveyQuestion, { foreignKey: 'surveyId', as: 'questions' });
Survey.hasMany(SurveyResponse, { foreignKey: 'surveyId', as: 'responses' });

module.exports = Survey;
```

### SurveyQuestion Model
```javascript
// src/models/SurveyQuestion.js
const SurveyQuestion = sequelize.define('SurveyQuestion', {
  id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
  surveyId: { type: DataTypes.UUID, allowNull: false },
  questionText: { type: DataTypes.TEXT, allowNull: false },
  questionType: {
    type: DataTypes.ENUM('rating', 'text', 'multiple_choice'),
    allowNull: false
  },
  options: {
    type: DataTypes.JSON,
    allowNull: true
  },
  order: { type: DataTypes.INTEGER, allowNull: false },
  required: { type: DataTypes.BOOLEAN, defaultValue: false }
}, {
  tableName: 'survey_questions',
  timestamps: true
});

SurveyQuestion.belongsTo(Survey, { foreignKey: 'surveyId', as: 'survey' });

module.exports = SurveyQuestion;
```

### SurveyResponse Model
```javascript
// src/models/SurveyResponse.js
const SurveyResponse = sequelize.define('SurveyResponse', {
  id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
  surveyId: { type: DataTypes.UUID, allowNull: false },
  userId: { type: DataTypes.UUID, allowNull: false },
  eventId: { type: DataTypes.UUID, allowNull: false },
  answers: { type: DataTypes.JSON, allowNull: false }
}, {
  tableName: 'survey_responses',
  timestamps: true,
  indexes: [
    { unique: true, fields: ['surveyId', 'userId'] }
  ]
});

SurveyResponse.belongsTo(Survey, { foreignKey: 'surveyId', as: 'survey' });
SurveyResponse.belongsTo(User, { foreignKey: 'userId', as: 'user' });

module.exports = SurveyResponse;
```

---

## Plan de Tareas

| # | Tarea | Descripción | Estimación |
|---|-------|-------------|------------|
| T501 | Setup modelos | Crear Feedback, Survey, SurveyQuestion, SurveyResponse models | 2h |
| T502 | Configurar relaciones | Definir associations entre modelos nuevos y existentes | 1h |
| T503 | Feedback Service | Implementar submitFeedback, getEventFeedback, getFeedbackSummary | 4h |
| T504 | Feedback Controller | Implementar endpoints de feedback con validaciones | 3h |
| T505 | Feedback Routes | Configurar rutas con middlewares de auth | 1h |
| T506 | Survey Service | Implementar createSurvey, getActiveSurvey, submitSurveyResponse | 4h |
| T507 | Survey Controller | Implementar endpoints de encuestas | 3h |
| T508 | Survey Routes | Configurar rutas de encuestas | 1h |
| T509 | Validaciones Joi | Crear schemas de validación para todos los endpoints | 2h |
| T510 | Tests unitarios | Tests de servicios y validaciones | 4h |
| T511 | Tests integración | Tests de endpoints con base de datos | 3h |

**Total estimado: 28 horas**

---

## Estrategia de Verificación

### Tests Unitarios
- **Feedback Service**:
  - `submitFeedback`: validar que solo asistentes puedan enviar feedback
  - `submitFeedback`: validar que no se pueda enviar feedback duplicado
  - `submitFeedback`: validar rating entre 1 y 5
  - `getFeedbackSummary`: verificar cálculo correcto de promedio y distribución

- **Survey Service**:
  - `createSurvey`: validar que solo organizador del evento pueda crear
  - `createSurvey`: validar que no se puedan crear dos encuestas para el mismo evento
  - `submitSurveyResponse`: validar que todas las preguntas requeridas tengan respuesta
  - `submitSurveyResponse`: validar que no se pueda responder dos veces

### Tests de Integración
- POST `/api/v1/events/:id/feedback`: verificar flujo completo de envío
- GET `/api/v1/events/:id/feedback/summary`: verificar cálculo de métricas con datos reales
- POST `/api/v1/events/:id/surveys`: verificar creación con transacción
- POST `/api/v1/events/:id/surveys/response`: verificar envío de respuestas
- Verificar que usuarios sin check-in reciban 403
- Verificar paginación en listado de feedbacks

### Casos de Verificación Manual
| ID | Caso | Resultado Esperado |
|----|------|-------------------|
| V501 | Asistente envía feedback válido | 201 Created |
| V502 | Asistente envía feedback duplicado | 400 FEEDBACK_ALREADY_SUBMITTED |
| V503 | No-asistente intenta enviar feedback | 403 NOT_ATTENDEE |
| V504 | Rating fuera de rango (0 o 6) | 400 VALIDATION_ERROR |
| V505 | Organizador crea encuesta | 201 Created |
| V506 | Organizador intenta crear segunda encuesta | 400 SURVEY_ALREADY_EXISTS |
| V507 | Participante responde encuesta con preguntas requeridas sin contestar | 400 VALIDATION_ERROR |
| V508 | Organizador ve resumen de métricas | 200 con datos correctos |

---

## Conclusión

El Módulo 5 (Encuestas y Comentarios Post-Evento) cierra el ciclo de feedback del sistema, permitiendo a los asistentes evaluar los eventos y a los organizadores obtener métricas valiosas. La integración con el módulo de acreditación asegura que solo quienes realmente asistieron puedan opinar, manteniendo la integridad de los datos de satisfacción.

**Integración con módulos anteriores:**
- Módulo 1: Control de roles para gestión de encuestas
- Módulo 2: Validación de eventos y organización
- Módulo 3: Verificación de asistencia (attended = true) como requisito para feedback
