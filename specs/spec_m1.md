# Especificación Módulo 1: Gestión de Roles

**Estado:** Finalizado

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

El Módulo 1 implementa la **Gestión de Roles** del sistema de eventos, permitiendo definir y controlar los tres tipos de usuarios:

- **Organizador**: Gestiona y administra eventos, tiene acceso total al sistema.
- **Participante**: Asiste a eventos, puede registrarse y visualizar eventos.
- **Disertante**: Expone charlas en eventos, puede crear y gestionar sus presentaciones.

### Objetivos
- Definir roles con permisos específicos
- Asignar roles a usuarios
- Controlar acceso basado en roles (RBAC)
- Mantener integridad en la asignación de permisos

---

## Arquitectura

El sistema utiliza una **Arquitectura en Capas (Layered Architecture)** con separación clara de responsabilidades:

```
┌─────────────────────────────────────────────┐
│           Cliente (Frontend)                │
└─────────────────┬───────────────────────────┘
                  │ HTTP/REST
┌─────────────────▼───────────────────────────┐
│         Capa de Presentación (API)          │
│  ┌──────────────┐    ┌──────────────┐     │
│  │   Routes     │───▶│ Controllers  │     │
│  └──────────────┘    └──────────────┘     │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Lógica de Negocio           │
│              (Services)                      │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Capa de Acceso a Datos               │
│            (Models/ORM)                      │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Base de Datos (PostgreSQL)          │
└─────────────────────────────────────────────┘
```

### Capas implementadas

1. **Capa de Presentación**: Rutas y controladores que manejan requests HTTP
2. **Capa de Lógica de Negocio**: Servicios con reglas de negocio
3. **Capa de Acceso a Datos**: Modelos Sequelize (ORM)
4. **Middlewares**: Autenticación y autorización

---

## Módulos y Responsabilidades

### 1. Role Controller (`src/controllers/role-controller.js`)
**Responsabilidades:**
- Manejar requests HTTP para operaciones de roles
- Validar datos de entrada
- Delegar lógica de negocio a Role Service
- Formatear respuestas HTTP

**Métodos:**
- `getAllRoles(req, res)` - Listar todos los roles
- `getRoleById(req, res)` - Obtener rol por ID
- `createRole(req, res)` - Crear nuevo rol
- `updateRole(req, res)` - Actualizar rol existente
- `deleteRole(req, res)` - Eliminar rol
- `assignRoleToUser(req, res)` - Asignar rol a usuario

---

### 2. Role Service (`src/services/role-service.js`)
**Responsabilidades:**
- Implementar lógica de negocio de roles
- Validar reglas de negocio (ej: no eliminar roles en uso)
- Interactuar con el modelo Role
- Manejar transacciones si es necesario

**Métodos:**
- `findAll()` - Obtener todos los roles
- `findById(id)` - Buscar rol por ID
- `create(roleData)` - Crear rol con validaciones
- `update(id, roleData)` - Actualizar rol
- `delete(id)` - Eliminar rol (verificando que no tenga usuarios)
- `assignToUser(userId, roleId)` - Asignar rol a usuario

---

### 3. Role Model (`src/models/Role.js`)
**Responsabilidades:**
- Definir esquema de datos del rol
- Configurar validaciones a nivel de base de datos
- Definir relaciones con otros modelos

**Campos:**
- `id`: UUID primario
- `name`: enum ['organizador', 'participante', 'disertante']
- `description`: texto descriptivo
- `permissions`: array de permisos (JSON)

---

### 4. Role Routes (`src/routes/role-routes.js`)
**Responsabilidades:**
- Definir endpoints de la API para roles
- Aplicar middlewares de autenticación y autorización

**Endpoints:**
- `GET /api/v1/roles` - Listar roles
- `GET /api/v1/roles/:id` - Obtener rol
- `POST /api/v1/roles` - Crear rol (solo organizador)
- `PUT /api/v1/roles/:id` - Actualizar rol (solo organizador)
- `DELETE /api/v1/roles/:id` - Eliminar rol (solo organizador)
- `POST /api/v1/roles/assign` - Asignar rol (solo organizador)

---

### 5. Middlewares

#### Auth Middleware (`src/middlewares/auth.js`)
- Verificar token JWT
- Decodificar y adjuntar usuario al request

#### Role Auth Middleware (`src/middlewares/role-auth.js`)
- Verificar que el usuario tenga el rol necesario
- Permitir acceso basado en permisos

---

## Flujo de Datos

### Caso 1: Crear un nuevo rol (Solo Organizador)

```
[1] Cliente → POST /api/v1/roles + Token + Body
                ↓
[2] Auth Middleware → Verifica JWT → Extrae usuario
                ↓
[3] Role Auth Middleware → Verifica que sea "organizador"
                ↓
[4] Role Routes → role-routes.js
                ↓
[5] Role Controller → createRole(req, res)
                ↓
[6] Validación Joi → Valida body (name, description, permissions)
                ↓
[7] Role Service → create(roleData)
                ↓
[8] Role Model → Role.create(roleData)
                ↓
[9] PostgreSQL → INSERT INTO roles
                ↓
[10] Respuesta → 201 Created + Role creado
```

---

### Caso 2: Asignar rol a usuario

```
[1] Cliente → POST /api/v1/roles/assign + Token + Body {userId, roleId}
                ↓
[2] Auth Middleware → Verifica JWT
                ↓
[3] Role Auth Middleware → Verifica "organizador"
                ↓
[4] Role Controller → assignRoleToUser(req, res)
                ↓
[5] Role Service → assignToUser(userId, roleId)
                ↓
[6] Validaciones:
    - ¿Existe userId? → User.findByPk(userId)
    - ¿Existe roleId? → Role.findByPk(roleId)
                ↓
[7] User Model → user.update({ roleId })
                ↓
[8] PostgreSQL → UPDATE users SET roleId = ?
                ↓
[9] Respuesta → 200 OK + Usuario con rol asignado
```

---

### Caso 3: Eliminar rol (con validación de integridad)

```
[1] Cliente → DELETE /api/v1/roles/:id + Token
                ↓
[2] Auth + Role Auth → Verifica permisos
                ↓
[3] Role Controller → deleteRole(req, res)
                ↓
[4] Role Service → delete(id)
                ↓
[5] Verificar usuarios asignados:
    - User.count({ where: { roleId: id } })
                ↓
[6] ¿Hay usuarios? → 400 Bad Request "Rol en uso"
                ↓
[7] ¿No hay usuarios? → Role.destroy({ where: { id } })
                ↓
[8] PostgreSQL → DELETE FROM roles WHERE id = ?
                ↓
[9] Respuesta → 200 OK "Rol eliminado"
```

---

## Endpoints y Contratos

Referencia completa en [contracts.md](../contracts.md#modulo-1-gestión-de-roles)

### Resumen de Endpoints

| Método | Endpoint | Descripción | Rol Requerido |
|--------|----------|-------------|---------------|
| GET | `/api/v1/roles` | Listar todos los roles | Organizador |
| GET | `/api/v1/roles/:id` | Obtener rol por ID | Organizador |
| POST | `/api/v1/roles` | Crear nuevo rol | Organizador |
| PUT | `/api/v1/roles/:id` | Actualizar rol | Organizador |
| DELETE | `/api/v1/roles/:id` | Eliminar rol | Organizador |
| POST | `/api/v1/roles/assign` | Asignar rol a usuario | Organizador |

---

## Dependencias y Capas

### Dependencias del Módulo 1

```
Módulo 1: Role Management
├── Dependencias Internas:
│   ├── User Model (para asignación de roles)
│   ├── Auth Middleware (verificación de token)
│   └── Role Auth Middleware (verificación de permisos)
│
├── Dependencias Externas:
│   ├── express (routing)
│   ├── jsonwebtoken (autenticación)
│   ├── bcrypt (hasheo de passwords)
│   ├── joi (validación)
│   └── sequelize (ORM)
│
└── Base de Datos:
    └── PostgreSQL
        ├── Tabla: roles
        └── Tabla: users (FK: roleId)
```

### Diagrama de Capas

```
┌─────────────────────────────────────────┐
│           Dependencias Externas          │
│   (Express, JWT, Joi, Sequelize, PG)    │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│           Capa de Modelos                │
│   Role.js ←── depende de ──→ User.js   │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│           Capa de Servicios              │
│   role-service.js                        │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│        Capa de Controladores             │
│   role-controller.js                     │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│         Capa de Rutas/Middlewares        │
│   role-routes.js + auth middlewares     │
└─────────────────────────────────────────┘
```

---

## Modelos de Datos

### Role Model

```javascript
// src/models/Role.js
const { DataTypes } = require('sequelize');
const { sequelize } = require('../config/database');

const Role = sequelize.define('Role', {
  id: {
    type: DataTypes.UUID,
    defaultValue: DataTypes.UUIDV4,
    primaryKey: true
  },
  name: {
    type: DataTypes.ENUM('organizador', 'participante', 'disertante'),
    allowNull: false,
    unique: true,
    validate: {
      isIn: [['organizador', 'participante', 'disertante']]
    }
  },
  description: {
    type: DataTypes.STRING(255),
    allowNull: false
  },
  permissions: {
    type: DataTypes.JSON,
    allowNull: false,
    defaultValue: []
  }
}, {
  tableName: 'roles',
  timestamps: true
});

// Relación: Un rol tiene muchos usuarios
Role.hasMany(User, { foreignKey: 'roleId', as: 'users' });

module.exports = Role;
```

### Permisos por Rol

| Rol | Permisos |
|-----|----------|
| **organizador** | `event:create`, `event:update`, `event:delete`, `user:manage`, `role:manage` |
| **participante** | `event:view`, `event:register` |
| **disertante** | `event:view`, `talk:create`, `talk:update` |

---

## Especificación Formal

### Contratos de Datos

#### RoleCreateDTO
```typescript
interface RoleCreateDTO {
  name: 'organizador' | 'participante' | 'disertante';
  description: string; // max 255 chars
  permissions: string[];
}
```

#### RoleUpdateDTO
```typescript
interface RoleUpdateDTO {
  description?: string;
  permissions?: string[];
}
```

#### RoleAssignDTO
```typescript
interface RoleAssignDTO {
  userId: string; // UUID
  roleId: string; // UUID
}
```

#### RoleResponseDTO
```typescript
interface RoleResponseDTO {
  id: string;
  name: string;
  description: string;
  permissions: string[];
  createdAt: Date;
  updatedAt: Date;
}
```

---

### Reglas de Negocio

1. **RN001**: Solo usuarios con rol "organizador" pueden gestionar roles
2. **RN002**: No se pueden eliminar roles que tengan usuarios asignados
3. **RN003**: Los nombres de roles deben ser únicos
4. **RN004**: Los permisos deben ser validados contra un catálogo de permisos válidos
5. **RN005**: Al crear un usuario, se debe asignar un rol válido
6. **RN006**: Un usuario solo puede tener un rol a la vez
7. **RN007** — Seguridad (OWASP API2): El sistema debe validar la firma, expiración e integridad del JWT en cada request y verificar que el estado y los permisos del usuario almacenados en base de datos no hayan sido modificados desde la emisión del token. Si el token es inválido, expiró, o los permisos del usuario fueron cambiados (ej: cambio de rol), la operación debe rechazarse con HTTP 401 y el token debe considerarse no válido para futuros requests.

---

### Casos de Uso

#### CU1: Crear Rol
**Actor**: Organizador  
**Precondición**: Usuario autenticado con rol organizador  
**Flujo**:
1. Organizador envía datos del rol (name, description, permissions)
2. Sistema valida datos
3. Sistema verifica que el nombre no exista
4. Sistema crea el rol
5. Sistema retorna rol creado

**Excepciones**:
- E1: Nombre duplicado → Error 400
- E2: Datos inválidos → Error 400 con detalles

---

#### CU2: Asignar Rol a Usuario
**Actor**: Organizador  
**Precondición**: Usuario autenticado con rol organizador y JWT válido vigente  
**Flujo**:
1. Organizador envía userId y roleId
2. Sistema valida firma, expiración e integridad del JWT y verifica que los permisos del organizador en base de datos sigan vigentes
3. Sistema verifica que usuario exista
4. Sistema verifica que rol exista
5. Sistema asigna rol al usuario
6. Sistema invalida el JWT del usuario cuyo rol fue modificado para forzar un nuevo login
7. Sistema retorna usuario con rol actualizado

**Excepciones**:
- E1: Usuario no encontrado → Error 404
- E2: Rol no encontrado → Error 404
- E3: JWT inválido, expirado, o permisos del organizador modificados desde la emisión del token → Error 401

---

#### CU3: Eliminar Rol
**Actor**: Organizador  
**Precondición**: Usuario autenticado con rol organizador y JWT válido vigente  
**Flujo**:
1. Organizador solicita eliminar rol por ID
2. Sistema valida firma, expiración e integridad del JWT y verifica que los permisos del organizador en base de datos sigan vigentes
3. Sistema verifica que rol exista
4. Sistema verifica que no tenga usuarios asignados
5. Sistema elimina rol
6. Sistema confirma eliminación

**Excepciones**:
- E1: Rol no encontrado → Error 404
- E2: Rol en uso → Error 400
- E3: JWT inválido, expirado, o permisos del organizador modificados desde la emisión del token → Error 401

---

### Matriz de Permisos

| Acción / Rol | Organizador | Participante | Disertante |
|--------------|-------------|--------------|------------|
| Ver eventos | ✓ | ✓ | ✓ |
| Crear evento | ✓ | ✗ | ✗ |
| Editar evento | ✓ | ✗ | ✗ |
| Eliminar evento | ✓ | ✗ | ✗ |
| Registrarse a evento | ✗ | ✓ | ✗ |
| Crear charla | ✗ | ✗ | ✓ |
| Editar charla propia | ✗ | ✗ | ✓ |
| Gestionar usuarios | ✓ | ✗ | ✗ |
| Gestionar roles | ✓ | ✗ | ✗ |

---

## Conclusión

El Módulo 1 (Gestión de Roles) proporciona la base para el control de acceso del sistema, definiendo tres roles claros con permisos específicos. La arquitectura en capas asegura mantenibilidad y testeabilidad, mientras que el uso de JWT y middleware de autorización garantiza seguridad en el acceso a los recursos.

**Próximos pasos:**
- Implementar Módulo 2: Gestión de Eventos
- Integrar con sistema de autenticación
- Desarrollar tests unitarios y de integración
