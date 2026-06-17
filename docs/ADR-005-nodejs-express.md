# ADR-005: Elección de Entorno y Framework para el Backend (Node.js y Express)

## Estado
Aceptado

## Contexto
Previo a la definición de las especificaciones del sistema, era necesario establecer la tecnología base para el desarrollo del backend. Se requería una tecnología que permitiera un desarrollo rápido, buena integración con herramientas de frontend, y una amplia comunidad de soporte.

## Decisión
Se decidió utilizar **Node.js** como entorno de ejecución y **Express.js** como framework para la construcción de la API REST. 

## Justificación
- **Express.js** es minimalista, flexible y cuenta con un vasto ecosistema de middlewares que aceleran el desarrollo (como `cors` y soporte para validaciones con `joi`).
- La curva de aprendizaje del equipo era favorable para el entorno de JavaScript.
- Node.js es altamente eficiente para operaciones de I/O, lo cual es ideal para el manejo de múltiples consultas a la base de datos.

## Consecuencias
- **Positivas:** Rápida implementación de endpoints, facilidad para integrar herramientas de testing como Jest.
- **Negativas:** La falta de tipado estricto por defecto requiere un esfuerzo adicional en la validación de esquemas.
