# ADR-001 — Adopción de Sequelize como ORM para acceso a datos

**Estado:** Aceptado  
**Fecha:** 2026-06-16  
**Decisores:** Amarilla Aldana Abigail Itatí, González Diego Agustín, Ibañez Adrian Jose María, Machado Victor Leandro  
**Relacionado:** Issue #13 · Project.md · spec_m1.md — spec_m8.md · src/models/

---

## Contexto

Con Node.js + Express y PostgreSQL, el equipo debía elegir cómo gestionar el acceso a la base de datos. El sistema cuenta con 9 modelos (`User`, `Role`, `Event`, `Talk`, `EventAttendee`, `Feedback`, `Survey`, `SurveyQuestion`, `SurveyResponse`) más el modelo `Notification` incorporado en M7, con múltiples relaciones entre ellos.

**Restricciones que aplicaban:**

- El equipo tiene experiencia previa con ORMs en contextos académicos.
- Se requiere gestión de migraciones para evolucionar el esquema sin pérdida de datos.
- Las asociaciones entre modelos son complejas: relaciones many-to-many (`Event` ↔ `User` a través de `event_attendees`), one-to-many (`Survey` → `SurveyQuestion` → `SurveyResponse`).
- Las operaciones críticas (eliminación de cuenta en M8, creación de encuestas en M5) requieren soporte de transacciones.
- El equipo no cuenta con un DBA dedicado; el ORM debe facilitar la definición del esquema desde código.

**Datos del proyecto que sustentan la decisión:**

- 10 modelos con ~15 asociaciones Sequelize definidas en `src/models/`.
- Consultas con múltiples JOINs e `include` anidados (ej: evento con organizador, charlas con disertantes, asistentes con estado de check-in).
- El módulo 6 requiere uso de `sequelize.fn('AVG', ...)`, `sequelize.fn('COUNT', ...)` y `GROUP BY` nativos.
- La eliminación de cuenta (M8) requiere una transacción que afecta 6 tablas.

---

## Decisión

Se adopta **Sequelize v6** como ORM para la gestión de modelos, relaciones, migraciones y consultas a la base de datos PostgreSQL.

**Alcance:**
- Cubre la definición de todos los modelos del sistema y sus asociaciones.
- Cubre migraciones de esquema con `sequelize-cli`.
- Cubre consultas simples y complejas (incluyendo aggregations con `sequelize.fn`).
- Cubre transacciones mediante `sequelize.transaction()`.
- No cubre procedimientos almacenados ni vistas SQL (se prefieren consultas Sequelize).
- Las consultas extremadamente complejas del M6 pueden usar `sequelize.query()` con SQL raw como escape hatch.

---

## Alternativas consideradas

**Opción A: Sequelize v6**
- ✅ ORM maduro con amplia documentación y comunidad en el ecosistema Node.js.
- ✅ Soporte completo de PostgreSQL, incluyendo tipos UUID, JSON y arrays.
- ✅ Sistema de migraciones integrado vía `sequelize-cli`.
- ✅ Definición de modelos en JavaScript puro, consistente con el stack.
- ✅ Soporte de asociaciones complejas (`belongsToMany`, `hasMany`, `include` anidado).
- ❌ Verboso en queries complejas comparado con alternativas modernas.
- ❌ Documentación de casos avanzados (hooks, eager loading profundo) puede ser confusa.

**Opción B: Prisma**
- ✅ TypeScript-first con schema declarativo y autocompletado superior.
- ✅ Migraciones automáticas basadas en schema diff.
- ❌ Requiere TypeScript o adaptación significativa para JavaScript puro.
- ❌ Modelo mental diferente al equipo (schema.prisma vs modelos JS).
- ❌ Menor adopción en el ecosistema académico local al momento de la decisión.

**Opción C: Knex.js (query builder sin ORM)**
- ✅ Máximo control sobre las queries SQL generadas.
- ✅ Menor abstracción, más predecible en performance.
- ❌ Sin definición de modelos ni asociaciones: requiere gestionar relaciones manualmente.
- ❌ Mayor cantidad de código boilerplate para las 10+ entidades del sistema.
- ❌ Sin soporte de migraciones nativo; requiere librería adicional.

---

## Consecuencias

**Beneficios esperados:**
- Los modelos en `src/models/` sirven como única fuente de verdad del esquema de datos.
- Las asociaciones Sequelize (`belongsTo`, `hasMany`, `belongsToMany`) generan los JOINs automáticamente en las consultas con `include`.
- Las migraciones versionadas garantizan reproducibilidad del esquema en cualquier entorno.
- Las transacciones con `sequelize.transaction()` garantizan atomicidad en M5 (creación de encuestas) y M8 (eliminación de cuenta).

**Costos o riesgos que se aceptan:**
- Sequelize puede generar queries subóptimas en casos complejos; requiere monitoreo de performance (logging de queries en desarrollo).
- Los cambios de esquema deben gestionarse vía migraciones y nunca con `sync({ force: true })` en producción.
- Actualizaciones de Sequelize (v6 → v7) pueden introducir breaking changes que requieran refactorización.

**Impacto en operación y equipo:**
- `sequelize-cli` debe instalarse globalmente o como dev-dependency para gestionar migraciones.
- Todos los modelos deben definirse en `src/models/` siguiendo la convención PascalCase.
- Un archivo `src/models/index.js` debe centralizar las asociaciones y exportar todos los modelos.

---

## Plan de implementación

1. Instalar `sequelize ^6.35.0`, `pg ^8.11.0`, `pg-hstore`.
2. Instalar `sequelize-cli` como dev-dependency.
3. Inicializar estructura de Sequelize (`npx sequelize-cli init`) para generar directorios `migrations/` y `seeders/`.
4. Implementar los 10 modelos en `src/models/` con tipos, validaciones y asociaciones.
5. Crear migraciones iniciales para cada tabla.
6. Configurar `config/database.js` con las credenciales de PostgreSQL desde `.env`.

**Dependencias:** Node.js 18+, PostgreSQL 14+, `sequelize ^6.35.0`, `pg ^8.11.0`.

**Métrica de éxito:**
- `sequelize.authenticate()` pasa sin errores en todos los entornos.
- `npx sequelize-cli db:migrate` crea todas las tablas con FKs correctas.
- Las consultas con `include` anidado (ej: evento + charlas + disertante) se ejecutan sin N+1 queries.

---

## Triggers de revisión

- Si se detectan problemas de N+1 queries en endpoints de alta frecuencia (alternativa: DataLoader o queries SQL raw).
- Si el equipo migra a TypeScript (evaluar Prisma como alternativa más adecuada para ese contexto).
- Si el volumen de datos requiere query optimization avanzada no alcanzable con la API de Sequelize.

**Fecha sugerida de revisión:** 2027-06-01
