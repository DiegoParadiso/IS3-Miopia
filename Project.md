# Gestión de Eventos - IS3 Miopia

## Descripción del Proyecto

Sistema de gestión de eventos que permite la creación, organización y administración de eventos. El sistema cuenta con gestión de roles para diferenciar permisos entre organizadores, participantes y disertantes.

## Stack Tecnológico

- **Runtime**: Node.js 18+
- **Framework**: Express.js
- **Base de datos**: PostgreSQL
- **ORM**: Sequelize
- **Autenticación**: JWT (JSON Web Tokens)
- **Validación**: Joi
- **Lenguaje**: JavaScript (ES6+)

## Convenciones de Código

- **Estilo**: ESLint + Prettier
- **Naming**:
  - Archivos: kebab-case (ej: user-controller.js)
  - Variables/Funciones: camelCase
  - Clases/Modelos: PascalCase
  - Constantes: UPPER_SNAKE_CASE
- **Commits**: Conventional Commits (feat:, fix:, docs:, etc.)

## Estructura de Directorios

```
IS3-Miopia/
├── Project.md              # Documento principal del proyecto
├── README.md               # Información de la cátedra
├── contracts.md             # Contratos de API
├── package.json            # Dependencias Node.js
├── config/
│   └── database.js         # Configuración de base de datos
├── src/
│   ├── models/             # Modelos de datos
│   │   ├── User.js
│   │   ├── Role.js
│   │   ├── Event.js
│   │   ├── Talk.js
│   │   ├── EventAttendee.js
│   │   ├── Feedback.js
│   │   ├── Survey.js
│   │   ├── SurveyQuestion.js
│   │   └── SurveyResponse.js
│   ├── controllers/        # Controladores
│   │   ├── auth-controller.js
│   │   ├── role-controller.js
│   │   ├── event-controller.js
│   │   ├── accreditation-controller.js
│   │   ├── certificate-controller.js
│   │   ├── feedback-controller.js
│   │   └── report-controller.js
│   ├── routes/             # Rutas de API
│   │   ├── auth-routes.js
│   │   ├── role-routes.js
│   │   ├── event-routes.js
│   │   ├── accreditation-routes.js
│   │   ├── certificate-routes.js
│   │   ├── feedback-routes.js
│   │   └── report-routes.js
│   ├── middlewares/        # Middlewares
│   │   ├── auth.js
│   │   └── role-auth.js
│   ├── services/          # Lógica de negocio
│   │   ├── auth-service.js
│   │   ├── role-service.js
│   │   ├── event-service.js
│   │   ├── accreditation-service.js
│   │   ├── certificate-service.js
│   │   ├── feedback-service.js
│   │   └── report-service.js
│   └── utils/              # Utilidades
│       └── validator.js
├── specs/                  # Especificaciones por módulo
│   ├── spec_m1.md          # Gestión de Roles
│   ├── spec_m2.md          # Gestión de Eventos
│   ├── spec_m3.md          # Acreditación
│   ├── spec_m4.md          # Certificados
│   ├── spec_m5.md          # Encuestas y Comentarios
│   └── spec_m6.md          # Informes y Agenda
└── tests/                  # Tests
    ├── unit/
    └── integration/
```

## Modelos

### User (Compartido)
```javascript
{
  id: UUID,
  email: STRING,
  password: STRING (hash),
  firstName: STRING,
  lastName: STRING,
  roleId: UUID (FK),
  createdAt: DATE,
  updatedAt: DATE
}
```

### Role
```javascript
{
  id: UUID,
  name: ENUM('organizador', 'participante', 'disertante'),
  description: STRING,
  permissions: JSON,
  createdAt: DATE,
  updatedAt: DATE
}
```

## Dependencias

### Producción
- express: ^4.18.2
- sequelize: ^6.35.0
- pg: ^8.11.0
- jsonwebtoken: ^9.0.2
- bcrypt: ^5.1.1
- joi: ^17.11.0
- dotenv: ^16.3.1
- pdfkit: ^0.14.0

### Desarrollo
- jest: ^29.7.0
- eslint: ^8.56.0
- prettier: ^3.1.0
- nodemon: ^3.0.2

## Formato de Respuestas API

### Respuesta Exitosa
```json
{
  "success": true,
  "data": { ... },
  "message": "Operación exitosa"
}
```

### Respuesta de Error
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Descripción del error"
  }
}
```

### Códigos de Estado HTTP
- 200: OK
- 201: Created
- 400: Bad Request
- 401: Unauthorized
- 403: Forbidden
- 404: Not Found
- 500: Internal Server Error

## Módulos del Sistema

- **Módulo 1**: Gestión de Roles (organizador/participante/disertante) → [specs/spec_m1.md](specs/spec_m1.md)
- **Módulo 2**: Gestión de Eventos → [specs/spec_m2.md](specs/spec_m2.md)
- **Módulo 3**: Acreditación de participantes → [specs/spec_m3.md](specs/spec_m3.md)
- **Módulo 4**: Generación de Certificados → [specs/spec_m4.md](specs/spec_m4.md)
- **Módulo 5**: Encuestas y Comentarios Post-Evento → [specs/spec_m5.md](specs/spec_m5.md)
- **Módulo 6**: Generación de Informes y Agenda → [specs/spec_m6.md](specs/spec_m6.md)

## Estado

En desarrollo.
