# ADR-007: Selección de PostgreSQL como Motor de Base de Datos Relacional

**Estado:** Aceptado  
**Fecha:** 2026-06-17  
**Decisores:** Amarilla Aldana Abigail Itatí, González Diego Agustín, Ibañez Adrian Jose María, Machado Victor Leandro]  
**Relacionado:** Issue #16

---

## Contexto

Al iniciar el diseño del sistema de gestión de eventos, el equipo se enfrentó a la primera decisión técnica fundamental: qué motor de base de datos relacional utilizar como capa de persistencia. El sistema planifica 10+ modelos (`User`, `Role`, `Event`, `Talk`, `EventAttendee`, `Feedback`, `Survey`, `SurveyQuestion`, `SurveyResponse`, `Notification`) con relaciones complejas entre sí, incluyendo asociaciones many-to-many (`Event` ↔ `User` mediante `event_attendees`) y one-to-many anidadas (`Survey` → `SurveyQuestion` → `SurveyResponse`).

**Restricciones que aplicaban en ese momento:**

- El sistema debía garantizar integridad referencial y consistencia transaccional, dado que operaciones como la creación de encuestas (M5) y la eliminación de cuentas (M8) afectan múltiples tablas de forma atómica.
- El stack del lado del servidor ya estaba definido como Node.js + Express, por lo que el motor de base de datos debía ser compatible con el ecosistema Node (drivers maduros y con soporte activo).
- El equipo no contaba con un DBA dedicado; se requería un motor con buena documentación, comunidad activa y facilidad de administración.
- El sistema debía soportar tipos de datos avanzados: UUIDs para claves primarias, JSON para permisos y metadatos, y ENUMs para campos como roles y tipos de notificación.
- El proyecto se ejecuta en un contexto académico con restricciones de presupuesto, por lo que la solución debía ser de código abierto y sin costos de licenciamiento.

**Datos del proyecto relevantes para la decisión:**

- 10+ tablas con múltiples foreign keys y restricciones de integridad referencial.
- Operaciones transaccionales que afectan hasta 6 tablas simultáneamente (M8).
- Consultas con agregaciones (`AVG`, `COUNT`, `GROUP BY`) para el módulo de informes (M6).
- Almacenamiento de campos semi-estructurados en formato JSON (permisos de roles, metadatos de notificaciones).
- Necesidad de migraciones de esquema versionadas para evolucionar la base de datos sin pérdida de datos.

---

## Decisión

Se selecciona **PostgreSQL 14+** como motor de base de datos relacional del sistema.

**Alcance:**

- Cubre la persistencia principal de todos los modelos del sistema.
- Cubre la ejecución de transacciones ACID para operaciones multi-tabla.
- Cubre consultas con agregaciones, JOINs complejos y subconsultas.
- Cubre el almacenamiento de tipos de datos avanzados (UUID, JSON, ENUM, array).
- No cubre caché de datos en memoria (se delega a Redis vía BullMQ, ADR-004).
- No cubre almacenamiento de archivos binarios (PDFs de certificados e informes se generan bajo demanda y se sirven en memoria).

---

## Alternativas consideradas

### Opción A: PostgreSQL 14+
- ✅ Cumplimiento ACID completo con soporte de transacciones serializables y aislamiento `READ COMMITTED` por defecto.
- ✅ Tipos de datos nativos para UUID, JSONB, ENUM y arrays — coinciden directamente con los modelos definidos en `Project.md`.
- ✅ Driver `pg` maduro y ampliamente adoptado en el ecosistema Node.js (8+ millones de descargas semanales).
- ✅ Soporte de índices avanzados (B-tree, Hash, GiST, GIN, BRIN) útiles para la estrategia de paginación (ADR-002).
- ✅ Comunidad activa, documentación excelente y gran cantidad de recursos educativos.
- ✅ Código abierto (licencia PostgreSQL) sin costos de licencia.
- ❌ Mayor consumo de memoria RAM comparado con motores embebidos como SQLite.
- ❌ Configuración inicial más compleja que alternativas como MySQL.

### Opción B: MySQL 8+
- ✅ Amplia adopción en la industria y gran cantidad de documentación.
- ✅ Cumplimiento ACID con motor InnoDB.
- ❌ Soporte limitado de tipos de datos avanzados (JSON existe pero menos maduro que PostgreSQL; UUID requiere funciones `UUID()`).
- ❌ Históricamente con rendimiento inferior en consultas con agregaciones complejas y CTEs (Common Table Expressions).
- ❌ Manejo de `ENUM` como tipo de dato es menos flexible (no permite agregar valores sin `ALTER TABLE`).

### Opción C: SQLite 3
- ✅ Cero configuración: base de datos embebida sin servidor.
- ✅ Ideal para prototipado rápido y pruebas unitarias.
- ❌ Sin soporte de concurrencia real (bloqueo a nivel de archivo en escritura).
- ❌ Sin tipos de datos nativos para UUID o JSON (requiere almacenamiento como texto).
- ❌ Rendimiento degradado en operaciones con alta concurrencia y múltiples JOINs.
- ❌ No apto para producción con múltiples instancias de aplicación escalando horizontalmente.

### Opción D: MariaDB 10+
- ✅ Fork de MySQL con mejor rendimiento en ciertos escenarios.
- ✅ Compatibilidad con MySQL, facilitando migración.
- ❌ Ecosistema Node.js con menor tracción que PostgreSQL (driver no oficial menos mantenido).
- ❌ Comunidad más pequeña, menos recursos educativos y ejemplos disponibles.

---

## Consecuencias

### Positivas

1. **Integridad transaccional garantizada:** Las operaciones críticas del sistema (creación de encuestas en M5, eliminación de cuentas en M8, registro de asistentes en M2) se ejecutan dentro de transacciones ACID, asegurando atomicidad, consistencia, aislamiento y durabilidad. Si una operación falla a mitad de camino, PostgreSQL revierte automáticamente todos los cambios, evitando datos huérfanos o inconsistentes.

2. **Soporte nativo de tipos de datos modernos:** PostgreSQL permite modelar fielmente los dominios del problema sin recurrir a capas de abstracción adicionales. Los UUIDs como claves primarias previenen colisiones en entornos distribuidos; JSONB almacena permisos de roles y metadatos de notificaciones con capacidad de consulta directa; los ENUMs garantizan valores válidos en campos como `type` de notificaciones y `name` de roles.

3. **Compatibilidad total con Sequelize y el ecosistema Node.js:** El driver `pg` es el más maduro y descargado para Node.js. Sequelize (ADR-001) tiene soporte de primera clase para PostgreSQL, incluyendo funciones de agregación nativas (`sequelize.fn`), transacciones, migraciones y tipos de datos específicos. Esta sinergia reduce la fricción técnica y acelera el desarrollo.

4. **Escalabilidad horizontal mediante replicación:** PostgreSQL soporta replicación nativa (streaming replication) que permitiría, en una etapa futura, configurar réplicas de solo lectura para distribuir la carga de consultas sin modificar la aplicación.

### Negativas

1. **Mayor consumo de recursos comparado con alternativas más livianas:** PostgreSQL requiere más memoria RAM y espacio en disco que motores como SQLite. En entornos de desarrollo con recursos limitados, esto puede traducirse en un inicio más lento del contenedor Docker y mayor consumo de recursos del equipo. Sin embargo, para un sistema con 10+ tablas y consultas complejas, este costo es aceptable frente a los beneficios de robustez y funcionalidad.

2. **Complejidad de configuración y tuning:** La configuración óptima de PostgreSQL (buffers compartidos, `work_mem`, `maintenance_work_mem`, planificador de consultas) requiere comprensión de sus parámetros internos. En un contexto académico sin DBA dedicado, existe el riesgo de una configuración subóptima que afecte el rendimiento. Se mitiga mediante el uso de valores por defecto sensibles y la revisión periódica de consultas lentas vía `EXPLAIN ANALYZE`.

3. **Operaciones de backup más complejas que en motores embebidos:** A diferencia de SQLite (un solo archivo), PostgreSQL requiere herramientas como `pg_dump` o `pg_basebackup` para realizar copias de seguridad consistentes. Esto añade un paso adicional en la estrategia de backup del sistema.

---

## Plan de implementación

1. Instalar PostgreSQL 14+ en los entornos de desarrollo (local o vía Docker según ADR-006).
2. Configurar las variables de entorno en `.env`:
   - `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`.
3. Crear la base de datos inicial:
   ```sql
   CREATE DATABASE is3_miopia;
   ```
4. Configurar `config/database.js` con Sequelize para usar PostgreSQL:
   ```javascript
   module.exports = {
     host: process.env.DB_HOST,
     port: process.env.DB_PORT,
     database: process.env.DB_NAME,
     username: process.env.DB_USER,
     password: process.env.DB_PASSWORD,
     dialect: 'postgres',
     dialectOptions: { ssl: process.env.DB_SSL === 'true' ? { rejectUnauthorized: false } : false },
     logging: process.env.NODE_ENV === 'development' ? console.log : false,
     pool: { max: 10, min: 0, acquire: 30000, idle: 10000 }
   };
   ```
5. Verificar la conexión con `sequelize.authenticate()` en el arranque de la aplicación.
6. Ejecutar migraciones iniciales con `npx sequelize-cli db:migrate`.

**Dependencias:** Node.js 18+, PostgreSQL 14+, `pg ^8.11.0`, `pg-hstore`.

**Métrica de éxito:**
- `sequelize.authenticate()` retorna exitosamente en los entornos de desarrollo, staging y producción.
- Las migraciones de Sequelize se ejecutan sin errores.
- Las transacciones rollbackean correctamente ante fallos simulados.

---

## Triggers de revisión

- Si el volumen de datos supera los 10 GB y las consultas comienzan a degradarse (evaluar particionamiento de tablas o migración a un clúster administrado).
- Si el equipo decide migrar a una arquitectura de microservicios (evaluar si cada servicio requiere su propia instancia de PostgreSQL o si conviene usar bases de datos por servicio).
- Si los costos de infraestructura se vuelven prohibitivos (evaluar alternativas administradas como Amazon RDS, Google Cloud SQL o Supabase).

**Fecha sugerida de revisión:** 2027-06-01
