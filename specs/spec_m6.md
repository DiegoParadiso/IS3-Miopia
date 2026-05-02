# Especificación Módulo 6: Generación de Informes y Agenda

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

El Módulo 6 implementa la **Generación de Informes y Agenda** del sistema, permitiendo a los organizadores generar reportes PDF sobre sus eventos (asistencia, estadísticas, certificados emitidos) y a cualquier usuario visualizar y descargar la agenda del evento con las charlas programadas, horarios y disertantes.

---

## Objetivos y Contexto

- Generar informes PDF de asistencia por evento
- Generar la agenda del evento en formato PDF (charlas, horarios, disertantes)
- Generar informe estadístico del evento (inscritos vs asistentes, ratings, etc.)
- Permitir descarga de informes solo al organizador del evento
- Permitir que cualquier usuario autenticado descargue la agenda
- Los informes se generan on-demand (no se almacenan en el servidor)

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
│  │  Report Routes    │→│ Report Control │ │
│  └──────────────────┘  └────────────────┘ │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Lógica de Negocio           │
│         Report Service                      │
│         (PDF Generation con PDFKit)         │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Acceso a Datos               │
│  Event, Talk, EventAttendee, Feedback       │
│  Models (consulta y agregación de datos)     │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Base de Datos (PostgreSQL)          │
└─────────────────────────────────────────────┘
```

---

## Módulos y Responsabilidades

### 1. Report Controller (`src/controllers/report-controller.js`)
**Responsabilidades:**
- Manejar requests HTTP para generación de informes y agendas
- Verificar permisos de acceso
- Delegar lógica a Report Service
- Retornar PDFs como streams en la respuesta

**Métodos:**
- `generateAttendanceReport(req, res)` - Generar PDF de asistencia
- `generateEventAgenda(req, res)` - Generar PDF de agenda del evento
- `generateEventStatistics(req, res)` - Generar PDF de estadísticas
- `getAvailableReports(req, res)` - Listar informes disponibles para un evento

---

### 2. Report Service (`src/services/report-service.js`)
**Responsabilidades:**
- Consultar y agregar datos de múltiples modelos
- Generar PDFs con PDFKit
- Formatear datos para presentación en informes
- Validar elegibilidad para generar informes

**Métodos:**
- `generateAttendancePDF(eventId)` - Generar PDF de lista de asistencia
- `generateAgendaPDF(eventId)` - Generar PDF de agenda
- `generateStatisticsPDF(eventId)` - Generar PDF de estadísticas
- `getEventReportData(eventId)` - Obtener datos consolidados del evento
- `validateReportAccess(eventId, userId)` - Validar acceso a informes

---

## Flujo de Datos

### Caso 1: Generar Informe de Asistencia

```
[1] Organizador → GET /api/v1/events/:id/reports/attendance + Token
                ↓
[2] Auth Middleware → Verifica JWT → extrae userId
                ↓
[3] Report Controller → generateAttendanceReport(req, res)
                ↓
[4] Report Service → validateReportAccess(eventId, userId)
                ↓
[5] Validación: ¿Usuario es organizador del evento?
                ↓ (si sí)
[6] Consulta de datos:
    - Event.findByPk(eventId)
    - EventAttendees.findAll({ where: { eventId }, include: [User] })
    - Contar: total registrados, total asistentes (attended=true)
                ↓
[7] Generación PDF con PDFKit:
    - Título: "Informe de Asistencia"
    - Datos del evento
    - Tabla: Nombre, Email, Rol, Asistió (Sí/No), Check-in
    - Resumen al final
                ↓
[8] Response → PDF stream, Content-Type: application/pdf
               Content-Disposition: attachment; filename="asistencia-{eventId}.pdf"
```

---

### Caso 2: Generar Agenda del Evento

```
[1] Usuario → GET /api/v1/events/:id/agenda/download + Token
                ↓
[2] Auth Middleware → Verifica JWT
                ↓
[3] Report Controller → generateEventAgenda(req, res)
                ↓
[4] Report Service → generateAgendaPDF(eventId)
                ↓
[5] Consulta de datos:
    - Event.findByPk(eventId, { include: ['organizer'] })
    - Talk.findAll({ where: { eventId }, include: ['speaker'], order: [['startTime', 'ASC']] })
                ↓
[6] Generación PDF con PDFKit:
    - Header: Logo/título del evento
    - Info del evento (título, fecha, ubicación, organizador)
    - Tabla cronológica de charlas:
      | Horario | Título | Disertante | Duración |
    - Notas al pie
                ↓
[7] Response → PDF stream, Content-Type: application/pdf
               Content-Disposition: attachment; filename="agenda-{eventId}.pdf"
```

---

### Caso 3: Generar Informe Estadístico

```
[1] Organizador → GET /api/v1/events/:id/reports/statistics + Token
                ↓
[2] Auth + Role Auth → Verifica JWT y rol organizador
                ↓
[3] Report Controller → generateEventStatistics(req, res)
                ↓
[4] Report Service → generateStatisticsPDF(eventId)
                ↓
[5] Consulta y agregación de datos:
    - Total registrados
    - Total asistentes (attended=true)
    - Tasa de asistencia (%)
    - Total de charlas
    - Promedio de rating (del módulo 5)
    - Distribución de ratings
    - Total de respuestas de encuesta
    - Lista de disertantes
                ↓
[6] Generación PDF con PDFKit:
    - Título: "Informe Estadístico del Evento"
    - Sección 1: Métricas generales (cards numéricas)
    - Sección 2: Gráfico de distribución de ratings (texto/barras)
    - Sección 3: Resumen de charlas
    - Sección 4: Resumen de encuestas (si existen)
                ↓
[7] Response → PDF stream, Content-Type: application/pdf
               Content-Disposition: attachment; filename="estadisticas-{eventId}.pdf"
```

---

## Historias de Usuario y Criterios de Aceptación

### HU601: Descargar Agenda del Evento
**Como** usuario autenticado  
**Quiero** descargar la agenda del evento en formato PDF  
**Para** conocer las charlas, horarios y disertantes del evento  

**Criterios de Aceptación:**
- Dado que el evento existe y tiene charlas programadas, puedo descargar la agenda
- La agenda incluye título del evento, fecha, ubicación y organizador
- Las charlas se muestran ordenadas por horario de inicio
- Cada charla muestra: título, disertante, horario de inicio y fin, duración
- Dado que el evento no tiene charlas, la agenda muestra solo info del evento
- Cualquier usuario autenticado puede descargar la agenda

---

### HU602: Generar Informe de Asistencia
**Como** organizador de un evento  
**Quiero** generar un PDF con la lista de asistencia  
**Para** tener un registro formal de quiénes asistieron  

**Criterios de Aceptación:**
- Dado que soy organizador del evento, puedo generar el informe
- El informe lista todos los registrados con su estado de asistencia
- Incluye: nombre, email, rol, si asistió (sí/no), fecha de check-in
- Incluye un resumen con totales al final
- Dado que no soy organizador del evento, no puedo generar este informe

---

### HU603: Generar Informe Estadístico
**Como** organizador de un evento  
**Quiero** generar un informe con estadísticas del evento  
**Para** evaluar el rendimiento del evento y tomar decisiones para futuros eventos  

**Criterios de Aceptación:**
- Dado que soy organizador del evento, puedo generar el informe estadístico
- El informe incluye: total registrados, total asistentes, tasa de asistencia
- Incluye métricas de satisfacción (si hay feedback)
- Incluye resumen de encuestas (si existen)
- Incluye cantidad de charlas y disertantes

---

### HU604: Ver Informes Disponibles
**Como** organizador de un evento  
**Quiero** ver qué informes puedo generar para un evento  
**Para** seleccionar el informe que necesito  

**Criterios de Aceptación:**
- Dado que soy organizador, puedo ver la lista de informes disponibles
- Cada informe tiene nombre, descripción y endpoint asociado
- Muestra si el informe requiere datos adicionales (ej: necesita charlas o feedback)

---

## Requisitos Funcionales y Reglas de Negocio

### Requisitos Funcionales
- **RF601**: El sistema debe generar la agenda del evento en formato PDF
- **RF602**: El sistema debe generar informe de asistencia en formato PDF
- **RF603**: El sistema debe generar informe estadístico en formato PDF
- **RF604**: La agenda debe ser accesible para cualquier usuario autenticado
- **RF605**: Los informes de asistencia y estadísticas solo para el organizador
- **RF606**: Los PDFs se generan on-demand, no se almacenan
- **RF607**: Los informes deben incluir datos consolidados de todos los módulos relevantes

---

### Reglas de Negocio

| Código | Regla |
|--------|-------|
| **RN601** | Solo el organizador del evento puede generar informes de asistencia y estadísticas |
| **RN602** | Cualquier usuario autenticado puede descargar la agenda del evento |
| **RN603** | Los PDFs se generan on-demand y no se almacenan en el servidor |
| **RN604** | La agenda solo se puede generar para eventos publicados |
| **RN605** | El informe de asistencia muestra todos los registrados (asistieron o no) |
| **RN606** | La tasa de asistencia se calcula como: (asistentes / registrados) * 100 |
| **RN607** | El informe estadístico incluye datos de feedback solo si existen calificaciones |
| **RN608** | El informe estadístico incluye datos de encuestas solo si existen respuestas |
| **RN609** | Las charlas en la agenda se ordenan cronológicamente por startTime |
| **RN610** | El nombre del archivo PDF debe ser descriptivo y contener el eventId |
| **RN611** | Los informes de asistencia y estadísticas requieren que el evento haya finalizado |
| **RN612** | Si no hay datos para un informe, el PDF se genera igual indicando ausencia de datos |

---

## Restricciones Técnicas

- Seguir arquitectura en capas (Routes → Controller → Service → Model)
- Usar Sequelize ORM para consultas de datos
- Usar **PDFKit** para generación de PDFs
- Los PDFs se envían como stream directamente en la respuesta HTTP
- No almacenar PDFs en disco ni en base de datos
- Usar transacciones solo cuando se requieran múltiples lecturas consistentes
- Las consultas de agregación deben usar métodos eficientes de Sequelize
- Seguir formato de nombre de archivo: `{tipo}-{eventId}.pdf`
- Los PDFs deben ser visualmente legibles y profesionalmente formateados
- Content-Type: `application/pdf` en todas las respuestas de PDF

---

## Endpoints y Contratos

Referencia completa en [contracts.md](../contracts.md#modulo-6-informes-y-agenda)

### Resumen de Endpoints

| Método | Endpoint | Descripción | Rol Requerido |
|--------|----------|-------------|---------------|
| GET | `/api/v1/events/:id/agenda/download` | Descargar agenda PDF | Todos (auth) |
| GET | `/api/v1/events/:id/reports/attendance` | Informe de asistencia PDF | Organizador |
| GET | `/api/v1/events/:id/reports/statistics` | Informe estadístico PDF | Organizador |
| GET | `/api/v1/events/:id/reports` | Listar informes disponibles | Organizador |

---

### 6.1 Descargar Agenda del Evento

**GET** `/api/v1/events/:id/agenda/download`

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

**Contenido del PDF de Agenda:**
1. **Encabezado**:
   - Título del evento
   - Fecha y ubicación
   - Nombre del organizador

2. **Cuerpo - Tabla de Charlas**:

   | Horario | Título | Disertante | Duración |
   |---------|--------|------------|----------|
   | 10:00-11:00 | Introducción a IA | Dr. García | 60 min |
   | 11:30-12:30 | Web Apps Modernas | Ing. López | 60 min |

3. **Pie**:
   - "Agenda sujeta a cambios"
   - Fecha de generación del documento

**Response 404:**
```json
{
  "success": false,
  "error": {
    "code": "EVENT_NOT_FOUND",
    "message": "Evento no encontrado"
  }
}
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

**GET** `/api/v1/events/:id/reports/attendance`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="asistencia-{eventId}.pdf"

[Binary PDF content]
```

**Contenido del PDF de Asistencia:**
1. **Encabezado**: "INFORME DE ASISTENCIA"
2. **Info del evento**: título, fecha, ubicación, organizador
3. **Tabla de registrados**:

   | # | Nombre | Email | Rol | Asistió | Check-in |
   |---|--------|-------|-----|---------|----------|
   | 1 | Juan Pérez | juan@... | Participante | Sí | 2026-06-15 10:30 |
   | 2 | María Gómez | maria@... | Participante | No | - |

4. **Resumen**:
   - Total registrados: 50
   - Total asistentes: 42
   - Tasa de asistencia: 84%

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

**Response 400:**
```json
{
  "success": false,
  "error": {
    "code": "EVENT_NOT_FINISHED",
    "message": "El informe de asistencia solo puede generarse después de finalizado el evento"
  }
}
```

---

### 6.3 Informe Estadístico

**GET** `/api/v1/events/:id/reports/statistics`

**Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="estadisticas-{eventId}.pdf"

[Binary PDF content]
```

**Contenido del PDF de Estadísticas:**
1. **Encabezado**: "INFORME ESTADÍSTICO DEL EVENTO"
2. **Sección 1 - Métricas Generales**:
   - Total registrados: 50
   - Total asistentes: 42
   - Tasa de asistencia: 84%
   - Total de charlas: 8
   - Total de disertantes: 5

3. **Sección 2 - Satisfacción** (si hay feedback):
   - Calificación promedio: 4.3/5.0
   - Distribución de ratings:
     - ★★★★★: 20 (44%)
     - ★★★★: 15 (33%)
     - ★★★: 5 (11%)
     - ★★: 3 (7%)
     - ★: 2 (4%)
   - Total de comentarios: 30

4. **Sección 3 - Encuestas** (si hay respuestas):
   - Total de respuestas: 27
   - Tasa de respuesta: 54%
   - Resultados por pregunta (resumen)

5. **Sección 4 - Disertantes**:
   - Lista de disertantes con cantidad de charlas

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

### 6.4 Listar Informes Disponibles

**GET** `/api/v1/events/:id/reports`

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

---

## Dependencias y Capas

```
Módulo 6: Informes y Agenda
├── Dependencias Internas:
│   ├── User Model (organizador, asistentes, disertantes)
│   ├── Event Model (datos del evento)
│   ├── Talk Model (charlas para agenda)
│   ├── EventAttendee Model (asistencia)
│   ├── Auth Middleware
│   └── Role Auth Middleware
│
├── Dependencias de Módulos Anteriores:
│   ├── Módulo 1: Role checking (organizador genera informes)
│   ├── Módulo 2: Event y Talk data
│   ├── Módulo 3: Accreditation data (attended, checkInAt)
│   └── Módulo 5: Feedback y Survey data (para estadísticas)
│
├── Dependencias Externas:
│   ├── express (routing)
│   ├── sequelize (ORM + agregaciones)
│   ├── pdfkit (generación de PDFs)
│   └── jsonwebtoken (autenticación)
│
└── Base de Datos:
    └── PostgreSQL
        ├── Tabla: events (lectura)
        ├── Tabla: talks (lectura)
        ├── Tabla: event_attendees (lectura)
        ├── Tabla: feedback (lectura, opcional)
        ├── Tabla: surveys (lectura, opcional)
        └── Tabla: survey_responses (lectura, opcional)
```

---

## Modelo de Datos

> **Nota**: Este módulo no crea nuevos modelos. Utiliza los modelos existentes de los módulos anteriores para consultas y generación de informes. Los datos se leen y agregan sin modificar la base de datos.

### Modelos Utilizados

| Modelo | Origen | Uso |
|--------|--------|-----|
| `Event` | Módulo 2 | Datos del evento (título, fecha, ubicación, estado) |
| `Talk` | Módulo 2 | Charlas para la agenda (título, horarios, speaker) |
| `EventAttendee` | Módulo 2 + 3 | Lista de registrados y estado de asistencia |
| `User` | Project.md | Datos de usuarios (nombre, email, rol) |
| `Feedback` | Módulo 5 | Calificaciones y comentarios para estadísticas |
| `Survey` | Módulo 5 | Encuestas para estadísticas |
| `SurveyResponse` | Módulo 5 | Respuestas de encuestas para estadísticas |

### Consultas Clave

```javascript
// Obtener todos los asistentes con info del usuario
EventAttendee.findAll({
  where: { eventId },
  include: [{
    model: User,
    as: 'user',
    attributes: ['id', 'firstName', 'lastName', 'email']
  }],
  order: [['createdAt', 'ASC']]
});

// Obtener charlas con disertante ordenadas por horario
Talk.findAll({
  where: { eventId },
  include: [{
    model: User,
    as: 'speaker',
    attributes: ['id', 'firstName', 'lastName']
  }],
  order: [['startTime', 'ASC']]
});

// Calcular promedio de rating
Feedback.findOne({
  where: { eventId },
  attributes: [
    [sequelize.fn('AVG', sequelize.col('rating')), 'averageRating'],
    [sequelize.fn('COUNT', sequelize.col('id')), 'totalFeedbacks']
  ]
});

// Distribución de ratings
Feedback.findAll({
  where: { eventId },
  attributes: [
    'rating',
    [sequelize.fn('COUNT', sequelize.col('id')), 'count']
  ],
  group: ['rating']
});
```

---

## Formato de los PDFs Generados

### Agenda del Evento
```
┌────────────────────────────────────────────┐
│           AGENDA DEL EVENTO                │
│                                            │
│   Título: Conferencia de Tecnología 2026   │
│   Fecha: 15 de Junio, 2026                │
│   Ubicación: Auditorio Principal           │
│   Organizador: Dr. Martín Rodríguez        │
│                                            │
│   ┌──────────┬──────────────┬────────────┐ │
│   │ Horario  │    Charla    │ Disertante │ │
│   ├──────────┼──────────────┼────────────┤ │
│   │ 10:00-11 │ Intro a IA   │ Dr.García  │ │
│   │ 11:30-12 │ Web Apps     │ Ing.López  │ │
│   │ 14:00-15 │ DevOps Today │ Sra.Torres │ │
│   └──────────┴──────────────┴────────────┘ │
│                                            │
│   * Agenda sujeta a cambios                │
│   Generado: 10/06/2026                     │
└────────────────────────────────────────────┘
```

### Informe de Asistencia
```
┌────────────────────────────────────────────┐
│       INFORME DE ASISTENCIA                │
│                                            │
│   Evento: Conferencia de Tecnología 2026   │
│   Fecha: 15 de Junio, 2026                │
│                                            │
│   ┌─┬─────────────┬───────────┬───┬───┬────┐│
│   │#│    Nombre   │   Email   │Rol│As │Check││
│   ├─┼─────────────┼───────────┼───┼───┼────┤│
│   │1│Juan Pérez   │juan@..    │Par│Sí │10:30││
│   │2|María Gómez  │maria@..   │Par│No │  -  ││
│   └─┴─────────────┴───────────┴───┴───┴────┘│
│                                            │
│   Resumen:                                 │
│   Total registrados: 50                    │
│   Total asistentes: 42                     │
│   Tasa de asistencia: 84%                  │
└────────────────────────────────────────────┘
```

### Informe Estadístico
```
┌────────────────────────────────────────────┐
│     INFORME ESTADÍSTICO DEL EVENTO         │
│                                            │
│   MÉTRICAS GENERALES:                      │
│   Registrados: 50    | Asistentes: 42      │
│   Asistencia: 84%    | Charlas: 8          │
│   Disertantes: 5                           │
│                                            │
│   SATISFACCIÓN:                            │
│   Promedio: 4.3/5.0                        │
│   ★★★★★: 20 (44%)  ★★★★: 15 (33%)         │
│   ★★★: 5 (11%)    ★★: 3 (7%)  ★: 2 (4%)  │
│   Comentarios: 30                           │
│                                            │
│   ENCUESTAS:                               │
│   Respuestas: 27 (54% de asistentes)       │
│                                            │
│   DISERTANTES:                             │
│   Dr.García: 2 charlas                     │
│   Ing.López: 3 charlas                     │
│   Sra.Torres: 1 charla                     │
│   ...                                      │
└────────────────────────────────────────────┘
```

---

## Plan de Tareas

| # | Tarea | Descripción | Estimación |
|---|-------|-------------|------------|
| T601 | Instalar PDFKit | Agregar pdfkit como dependencia | 0.5h |
| T602 | Report Service base | Crear service con métodos de validación y consulta | 3h |
| T603 | Generar Agenda PDF | Implementar generateAgendaPDF con PDFKit | 4h |
| T604 | Generar Asistencia PDF | Implementar generateAttendancePDF con PDFKit | 4h |
| T605 | Generar Estadísticas PDF | Implementar generateStatisticsPDF con agregaciones | 5h |
| T606 | Report Controller | Implementar controller con endpoints de informes | 3h |
| T607 | Report Routes | Configurar rutas con middlewares de auth | 1h |
| T608 | Endpoint informes disponibles | Implementar listado de informes disponibles | 2h |
| T609 | Formateo de PDFs | Refinar formato visual de los 3 tipos de PDF | 3h |
| T610 | Tests unitarios | Tests de service y generación de datos | 3h |
| T611 | Tests integración | Tests de endpoints y validación de PDFs | 3h |

**Total estimado: 31.5 horas**

---

## Estrategia de Verificación

### Tests Unitarios
- **Report Service**:
  - `validateReportAccess`: verificar que solo organizador acceda
  - `generateAgendaPDF`: verificar que se generen datos correctamente
  - `generateAttendancePDF`: verificar conteo de asistentes
  - `generateStatisticsPDF`: verificar cálculo de tasa de asistencia
  - `generateStatisticsPDF`: verificar que incluya feedback si existe
  - `generateStatisticsPDF`: verificar que excluya feedback si no existe

### Tests de Integración
- GET `/api/v1/events/:id/agenda/download`: verificar generación de PDF válido
- GET `/api/v1/events/:id/reports/attendance`: verificar acceso solo para organizador
- GET `/api/v1/events/:id/reports/statistics`: verificar datos estadísticos correctos
- GET `/api/v1/events/:id/reports`: verificar listado de informes disponibles
- Verificar que PDFs tengan Content-Type correcto
- Verificar nombres de archivo en Content-Disposition

### Casos de Verificación Manual
| ID | Caso | Resultado Esperado |
|----|------|-------------------|
| V601 | Usuario descarga agenda de evento publicado | PDF descargado correctamente |
| V602 | Usuario descarga agenda de evento draft | 400 EVENT_NOT_PUBLISHED |
| V603 | Organizador descarga informe de asistencia de evento finalizado | PDF con lista completa |
| V604 | No-organizador intenta descargar informe de asistencia | 403 NOT_ORGANIZER |
| V605 | Organizador descarga informe de evento no finalizado | 400 EVENT_NOT_FINISHED |
| V606 | Organizador descarga estadísticas con feedback | PDF incluye sección de satisfacción |
| V607 | Organizador descarga estadísticas sin feedback | PDF sin sección de satisfacción |
| V608 | Evento sin charlas - descargar agenda | PDF con info del evento, sin tabla de charlas |
| V609 | Verificar nombre de archivo del PDF | "asistencia-{uuid}.pdf" |
| V610 | Verificar Content-Type de respuesta | application/pdf |

---

## Conclusión

El Módulo 6 (Generación de Informes y Agenda) cierra el ciclo de valor del sistema, proporcionando a los organizadores herramientas concretas para evaluar sus eventos y a los participantes información clara sobre la programación. La generación on-demand de PDFs elimina la necesidad de almacenamiento adicional mientras garantiza que los informes siempre reflejen los datos más actualizados.

**Integración con módulos anteriores:**
- Módulo 1: Control de acceso por rol para informes restringidos
- Módulo 2: Datos de eventos, charlas y registros como fuente principal
- Módulo 3: Datos de acreditación para informes de asistencia
- Módulo 5: Datos de feedback y encuestas para informes estadísticos

**Dependencia externa clave:** PDFKit para generación de documentos PDF.
