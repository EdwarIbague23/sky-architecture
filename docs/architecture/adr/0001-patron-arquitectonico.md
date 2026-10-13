# 0001 - Patrón Arquitectónico

**Estado**: Aceptado

## Contexto

Sky Fussion Analytics es un proyecto de análisis geospatial y procesamiento de datos satelitales que requiere combinar múltiples tecnologías y tipos de procesamiento:

- **Frontend interactivo** en React con visualización de mapas y resultados
- **APIs REST** para consumo de servicios por parte de frontend y externos
- **Pipeline de Machine Learning** en Python para inferencia y procesamiento de imágenes
- **Orquestación de agentes** para tareas distribuidas y en segundo plano
- **Base de datos de grafo** para representar relaciones espaciales y conexiones entre entidades

El proyecto original surgió como una necesidad de integrar herramientas dispares (satélites, modelos ML, visualización de datos) en una plataforma cohesiva, lo que naturalmente llevó a una arquitectura que refleja estas diferentes preocupaciones.

## Decisión

Se adoptó una arquitectura **Modular Monolith con elementos de microservicios por dominio** (domain-driven modular monolith). Los detalles clave son:

### Estructura de Capas

1. **Capa de Presentación** (`src/frontend/`): Aplicación React con Tailwind CSS. Maneja la UI, consumo de APIs y visualización de resultados geospaciales.

2. **Capa de APIs** (`backend-node/` y `src/backend/`):
   - **Node.js**: Manejo de endpoints de API, integración con LLM (Large Language Models), operaciones de Neo4j y autenticación básica
   - **.NET Core**: Lógica de negocio central, procesamiento de solicitudes, seguridad y validación

3. **Capa de Datos y ML** (`src/python/`): Pipeline completo para ingestión, preprocesamiento, inferencia de modelos y generación de resultados satelitales.

4. **Capa de Orchestration** (`agents/`): Framework para tarea distribuidas (prediction-agent, reporting-agent, analysis-agent) que ejecutan procesos en segundo plano.

### Módulos Acoplados Internamente, Aislados Externamente

- Cada módulo tiene su propia base de datos o almacenamiento apropiado (ver ADR-0002)
- Las interfaces entre módulos se definen a través de APIs y mensajes de agente
- El monolito permite compartir código y configuración común mientras mantiene la independencia de despliegue de componentes críticos

## Consecuencias

### Positivas

- **Flexibilidad tecnológica**: Cada equipo puede usar la mejor herramienta para su dominio (React para UI, Node.js para APIs de LLM, .NET para lógica de negocio, Python para ML)
- **Despliegue incremental**: Componentes individuales pueden actualizarse sin requerir un despliegue completo del sistema
- **Curva de aprendizaje gradual**: Los nuevos desarrolladores pueden comenzar con su área de experiencia y expandirse gradualmente
- **Testing aislado**: Los módulos pueden probarse de forma independiente usando sus APIs de integración

### Desafiantes

- **Complejidad de orquestación**: El coordination entre Node.js, .NET y Python requiere atención cuidadosa a patrones de comunicación (mensajería, APIs REST)
- **Consistencia de desarrollo**: Es necesario establecer y hacer cumplir estándares de API y formatos de datos entre módulos
- **Despliegue multi-contenedor**: Requiere Docker y docker-compose para orquestar los diferentes servicios (ya contemplado en la configuración existente)
- **Gestión de dependencias**: Cada lenguaje tiene su propio ecosistema de dependencias que debe mantenerse

## Alternativas Consideradas

| Patrón | Razón de no selección |
|---|---|
| Microservicios puros | Overhead operativo excesivo para el tamaño y madurez actual del equipo; complexity innecesaria sin necesidad de independencia de despliegue a nivel de servicio |
| Monolito tradicional | Acoplamiento demasiado alto entre concerns diferentes (UI, ML, DB); dificulta el uso de tecnologías diversas por segmento |
| Serverless | Dificultad para mantener estado y sesiones largas en procesamiento de imágenes satelitales; costos impredecibles para cargas de trabajo por lotes |

## Relación con otros ADRs

Esta decisión de patrón arquitectónico establece el marco dentro del cual operan las decisiones de persistencia (ADR-0002) y seguridad (ADR-0003). La arquitectura modular justifica el uso de múltiples estrategias de almacenamiento y el diseño de seguridad por capas.