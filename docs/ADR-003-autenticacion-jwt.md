# ADR-003 — Autenticación basada en JSON Web Tokens (JWT)
 
**Estado:** Aceptado  
**Fecha:** 2026-06-17  
**Decisores:** Amarilla Aldana Abigail Itatí, González Diego Agustín, Ibañez Adrian Jose María, Machado Victor Leandro  
**Relacionado:** Issue #14 · Project.md · spec_m1.md — spec_m8.md · src/middlewares/auth.js · src/routes/auth-routes.js
 
---
 
## Contexto
 
El sistema de gestión de eventos requiere un mecanismo de autenticación para diferenciar permisos entre organizadores, participantes y disertantes (M1). El sistema debe ser capaz de procesar múltiples peticiones simultáneas de forma rápida y segura, protegiendo los recursos privados de cada módulo.
 
**Restricciones que aplicaban:**
- El backend del sistema debe ser stateless (sin estado) para facilitar el escalado horizontal y reducir el uso de memoria en el servidor.
- La autenticación debe ser de bajo overhead y fácil consumo por clientes web, móviles o APIs externas.
- Debe integrarse de manera fluida con los middlewares de autorización de roles (`src/middlewares/role-auth.js`).
- Los datos de sesión (ID de usuario, rol) deben poder recuperarse sin realizar consultas adicionales a la base de datos en cada petición, optimizando la latencia general del sistema.
 
**Datos del proyecto que sustentan la decisión:**
- Todos los módulos privados (M1 al M8) requieren verificar la identidad y los permisos del usuario a través de la cabecera `Authorization: Bearer <token>`.
- El flujo de login (`POST /api/v1/auth/login`) y registro (`POST /api/v1/auth/register`) deben emitir credenciales válidas y seguras de forma directa.
- El módulo 8 (Gestión de Perfil) y el módulo 7 (Notificaciones) necesitan un acceso constante a `req.user.id` para filtrar los recursos pertenecientes al usuario autenticado.
 
---
 
## Decisión
 
Se adopta **JSON Web Tokens (JWT)** como estándar para la autenticación stateless y el intercambio seguro de claims entre el cliente y el servidor.
 
**Alcance:**
- Cubre la generación de tokens en el servicio de autenticación tras el inicio de sesión o registro de usuarios.
- Cubre la validación de tokens en la capa de middlewares en todas las rutas protegidas del sistema.
- El token incluirá en su payload únicamente información no sensible: `id` (UUID del usuario) y `role` (nombre del rol).
- El algoritmo utilizado para la firma del token será **HMAC con SHA-256 (HS256)**, utilizando una clave secreta (`JWT_SECRET`) almacenada en variables de entorno.
- El tiempo de expiración por defecto se fija en **24 horas**.
- No cubre la implementación de tokens de refresco (Refresh Tokens) en la fase actual.
 
---
 
## Alternativas consideradas
 
**Opción A: JSON Web Tokens (JWT) [Adoptada]**
- ✅ **Stateless:** El servidor no necesita almacenar datos de sesión en memoria ni consultar la base de datos para verificar el token en cada request.
- ✅ **Escalabilidad:** Simplifica el escalado horizontal, ya que cualquier instancia del servidor backend puede validar el token de forma independiente utilizando la clave secreta.
- ✅ **Compatibilidad:** Estándar ampliamente adoptado, fácil de integrar en el ecosistema Node.js y Express mediante la librería `jsonwebtoken`.
- ❌ **Complejidad de revocación:** Es difícil invalidar un token JWT antes de su expiración de forma stateless (se mitiga en M8 validando la existencia del usuario en base de datos para operaciones críticas).
 
**Opción B: Sesiones basadas en Base de Datos (Stateful)**
- ✅ **Revocación inmediata:** Permite invalidar sesiones de inmediato destruyendo el registro de sesión en la base de datos.
- ❌ **Overhead de I/O:** Requiere realizar una consulta a la base de datos en cada petición HTTP entrante para validar la sesión, incrementando la latencia del sistema.
- ❌ **Problemas de escalado:** Requiere replicar o centralizar la base de datos/almacén de sesiones (ej: Redis) a medida que el backend escala horizontalmente.
 
**Opción C: API Keys simples**
- ✅ Muy simple de implementar y de bajo overhead.
- ❌ Inadecuado para flujos interactivos de usuarios finales.
- ❌ Menos seguro que JWT para el manejo dinámico de roles y expiración de sesiones.
 
---
 
## Consecuencias
 
**Beneficios esperados:**
- El backend se mantiene totalmente stateless, lo que permite un escalado horizontal lineal y sin fricción.
- Reducción en los tiempos de respuesta al eliminar consultas repetitivas de sesión a PostgreSQL en cada petición a endpoints privados.
- El middleware de autorización puede leer el ID y el rol del usuario decodificando el JWT directamente desde el header de autorización.
 
**Costos o riesgos que se aceptan:**
- Si la clave secreta `JWT_SECRET` se ve comprometida, toda la seguridad del sistema se ve vulnerada. Se deben aplicar políticas estrictas para la gestión de variables de entorno en producción.
- La invalidación inmediata de tokens en caso de cierre de sesión no es posible de manera pura. Los tokens seguirán siendo válidos hasta que expiren.
 
**Impacto en operación y equipo:**
- Se requiere la librería `jsonwebtoken` como dependencia de producción en `package.json`.
- Cada entorno de despliegue debe contar obligatoriamente con las variables `JWT_SECRET` y `JWT_EXPIRES_IN` configuradas.
 
---
 
## Plan de implementación
 
1. Instalar la dependencia `jsonwebtoken ^9.0.2` en el proyecto.
2. Definir e implementar el middleware `auth.js` en `src/middlewares/` que:
   - Extraiga el token del header `Authorization: Bearer <token>`.
   - Valide la firma utilizando `jwt.verify` y la variable `JWT_SECRET`.
   - Adjunte el payload decodificado en `req.user`.
   - Retorne error 401 si el token es inválido o no está presente.
3. Actualizar `src/services/auth-service.js` para generar los tokens utilizando `jwt.sign` en los métodos de login y registro.
4. Documentar los contratos de login y registro en `contracts.md` especificando la devolución del token.
 
**Dependencias:** Node.js 18+, Express.js 4+, `jsonwebtoken ^9.0.2`.
 
**Métrica de éxito:**
- El middleware de autenticación valida correctamente los tokens y rechaza las peticiones no autenticadas con código HTTP 401.
- El tiempo empleado en verificar la validez del token en cada request es inferior a 2ms (verificado mediante logs de desarrollo).
 
---
 
## Triggers de revisión
 
- Si surge la necesidad crítica de implementar cierre de sesión instantáneo global o invalidación de tokens en caso de robos de sesión (evaluar uso de lista negra de tokens en memoria caché como Redis).
- Si el tamaño de los claims requeridos en el token crece significativamente, afectando el ancho de banda (evaluar pasar a un token de referencia o acotar claims).
- Si el ecosistema requiere de federación de identidades o soporte OAuth2/OIDC.
 
**Fecha sugerida de revisión:** 2026-12-15
