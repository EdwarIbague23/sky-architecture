# Sky Fussion Analytics - Arquitectura del Sistema

## Visión Global

Este documento sirve como índice y punto de entrada al ecosistema de documentación viva de la arquitectura de **Sky Fussion Analytics**. El proyecto combina múltiples tecnologías (Node.js, .NET, Python, React) en una arquitectura modular orientada al procesamiento y análisis de datos geospaciales satelitales.

## Trazabilidad

La arquitectura está documentada a través de los siguientes Architecture Decision Records (ADRs) y diagramas C4:

| Componente | Archivo | Descripción |
|---|---|---|
| Patrón Arquitectónico | [./adr/0001-patron-arquitectonico.md](./adr/0001-patron-arquitectonico.md) | Define el patrón modular monolith con elementos de microservicios |
| Sistema de Persistencia | [./adr/0002-sistema-de-persistencia.md](./adr/0002-sistema-de-persistencia.md) | Estrategia de almacenamiento multi-base de datos (Neo4j, SQL, JSON) |
| Estrategia de Seguridad | [./adr/0003-estrategia-seguridad.md](./adr/0003-estrategia-seguridad.md) | Autenticación, autorización y control de acceso propuesto |

### Diagramas C4

| Diagrama | Ubicación | Descripción |
|---|---|---|
| Contexto del Sistema (Nivel 1) | [diagrams/c4-context.puml](./diagrams/c4-context.puml) | Actores y componentes del sistema en contexto |
| Secuencia de Flujo Clave | [diagrams/secuencia-proceso-clave.puml](./diagrams/secuencia-proceso-clave.puml) | Flujo transaccional de análisis satelital |

## Justificación de la Arquitectura

### Mitigación de Deuda Técnica

1. **Arquitectura Modular por Dominio**: Al separar funcionalidades en Frontend (React), Backend Node.js, Backend .NET Core y Python ML Pipeline, cada componente puede evolucionar de forma independiente sin afectar los demás. Esto evita la clásica "bolita de lodo" en una sola base de código.

2. **Persistencia Especializada**: El uso de Neo4j para datos de grafo espacial, SQL para transacciones y JSON para metadatos ligeros evita el uso inapropiado de una sola tecnología para todos los casos de uso, reduciendo la complejidad y los cuellos de botella de rendimiento.

3. **Orchestration basada en Agentes**: El framework de agentes permite encapsular lógica compleja de procesamiento por lotes y ML, manteniendo los endpoints de API ligeros y focuseados en consumo de servicios.

### Facilitación del Onboarding

#### Para Desarrolladores Humanos

- **Documentación declarativa**: Los ADRs registran el "por qué" de cada decisión, no solo el "qué". Los nuevos miembros del equipo pueden comprender el razonamiento arquitectónico en minutos leyendo los records.
- **Diagramas C4 vivos**: Los diagramas de contexto y secuencia proporcionan una visión inmediata del sistema sin necesidad de navegar por miles de líneas de código.
- **Separación de tecnologías**: La claridad de que Node.js maneja APIs y LLM, .NET Core maneja lógica de negocio, y Python maneja ML, reduce la confusión sobre dónde implementar nueva funcionalidad.

#### Para IAs (asistentes de desarrollo)

- **Estructura predictiva**: La nomenclatura consistente (0001, 0002, 0003) y la ubicación fija de los archivos permite a los modelos de IA navegar y comprender la arquitectura rápidamente.
- **Metadatos estructurados**: El formato markdown con campos de estado (Aceptado, Propuesta) permite a los sistemas automatizados filtrar decisiones por estado.
- **Referencias cruzadas**: Los enlaces entre README, ADRs y diagramas crean un grafo de conocimiento que los sistemas de IA pueden recorrer para respuesta a preguntas sobre el proyecto.

## Estructura de Directorios

```
docs/architecture/
├── README.md                    # Este archivo - índice y visión global
├── adr/
│   ├── 0001-patron-arquitectonico.md   # Patrón arquitectónico aceptado
│   ├── 0002-sistema-de-persistencia.md # Persistencia aceptada
│   └── 0003-estrategia-seguridad.md    # Seguridad propuesta
└── diagrams/
    ├── c4-context.puml           # Diagrama C4 Contexto
    └── secuencia-proceso-clave.puml  # Diagrama de secuencia
```