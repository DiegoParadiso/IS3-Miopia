# Especificación Módulo 4: Certificados

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

El Módulo 4 implementa la **Generación de Certificados** de asistencia a eventos. Solo los usuarios que asistieron efectivamente a eventos finalizados pueden descargar su certificado en formato PDF.

### Objetivos
- Generar certificados PDF de asistencia
- Validar elegibilidad para obtener certificado
- Descargar certificado en formato PDF
- Listar certificados disponibles para un usuario

---

## Arquitectura

```
┌─────────────────────────────────────────────┐
│           Cliente (Frontend)                │
└─────────────────┬───────────────────────────┘
                  │ HTTP/REST
┌─────────────────▼───────────────────────────┐
│         Capa de Presentación (API)          │
│  ┌────────────────┐  ┌──────────────────┐ │
│  │CertificateRoutes│→│CertificateController│
│  └────────────────┘  └──────────────────┘ │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Lógica de Negocio           │
│         Certificate Service                 │
│         (PDF Generation con PDFKit)         │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Acceso a Datos               │
│    Event + EventAttendee + User Models      │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Base de Datos (PostgreSQL)          │
└─────────────────────────────────────────────┘
```

---

## Módulos y Responsabilidades

### 1. Certificate Controller (`src/controllers/certificate-controller.js`)
**Métodos:**
- `generateCertificate(req, res)` - Generar y descargar PDF
- `getAvailableCertificates(req, res)` - Listar certificados disponibles

### 2. Certificate Service (`src/services/certificate-service.js`)
**Métodos:**
- `generatePDF(userId, eventId)` - Generar PDF del certificado
- `getAvailableCertificates(userId)` - Obtener eventos elegibles para certificado
- `validateEligibility(userId, eventId)` - Validar si puede obtener certificado

---

## Flujo de Datos

### Caso: Generar Certificado

```
[1] Usuario → GET /api/v1/certificates/:eventId/download + Token
                ↓
[2] Auth Middleware → Verifica JWT → extrae userId
                ↓
[3] Certificate Controller → generateCertificate(req, res)
                ↓
[4] Certificate Service → validateEligibility(userId, eventId)
                ↓
[5] Validaciones:
    - ¿Usuario registrado al evento?
    - ¿Usuario tiene attended = true?
    - ¿Evento está finalizado (date < ahora)?
                ↓
[6] Generación PDF con PDFKit:
    - Título: "Certificado de Asistencia"
    - Nombre del participante
    - Nombre del evento
    - Fecha y ubicación del evento
    - Fecha de generación
                ↓
[7] Response → PDF stream con Content-Type: application/pdf
```

---

## Endpoints y Contratos

### 4.1 Descargar Certificado
**GET** `/api/v1/certificates/:eventId/download`

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

---

### 4.2 Listar Certificados Disponibles
**GET** `/api/v1/certificates`

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

## Formato del Certificado PDF

El certificado generado debe incluir:

1. **Encabezado**: "CERTIFICADO DE ASISTENCIA"
2. **Cuerpo**:
   - "Se certifica que"
   - Nombre completo del participante (firstName + lastName)
   - "asistió al evento"
   - Título del evento
   - "realizado el" + fecha del evento
   - "en" + ubicación del evento
3. **Pie**:
   - "Fecha de emisión: " + fecha de generación
   - Línea decorativa inferior

---

## Reglas de Negocio

1. **RN401**: Solo usuarios con `attended = true` pueden obtener certificado
2. **RN402**: El evento debe estar finalizado (date < fecha actual) para generar certificado
3. **RN403**: El certificado se genera en formato PDF usando PDFKit
4. **RN404**: El PDF se devuelve como stream directamente en la respuesta HTTP
5. **RN405**: El nombre del archivo debe ser `certificado-{eventId}.pdf`
6. **RN406**: No se almacenan los PDFs en el servidor, se generan on-demand
7. **RN407**: Solo el propio usuario puede descargar sus certificados
8. **RN408**: La lista de certificados disponibles solo muestra eventos finalizados con asistencia confirmada
