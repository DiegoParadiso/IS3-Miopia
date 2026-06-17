# ADR-006: Dockerización para Escalabilidad y Control de Entornos

## Estado
Aceptado

## Contexto
Ante la necesidad de que los módulos de la aplicación puedan ser escalados horizontalmente, así como la dificultad de mantener los mismos entornos en las máquinas de todos los desarrolladores y en producción, surgió la necesidad de implementar un sistema de contenedores.

## Decisión
Se decidió incorporar **Docker** y **Docker Compose** en el proyecto para orquestar la aplicación Node.js junto a sus dependencias (base de datos PostgreSQL, Redis para colas de mensajes).

## Justificación
- Permite aislar la aplicación, garantizando que funcione de forma idéntica en cualquier entorno.
- Facilita el escalado de módulos (por ejemplo, levantar múltiples instancias del contenedor de la API si el tráfico aumenta).
- Centraliza la infraestructura, permitiendo levantar todo el ecosistema con un simple `docker-compose up`.

## Consecuencias
- **Positivas:** Escalabilidad simplificada, eliminación del problema "funciona en mi máquina" y despliegue estandarizado.
- **Negativas:** Introduce una nueva capa de complejidad y requiere conocimiento de Docker por parte de los integrantes del equipo.
