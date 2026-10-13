# 0002 - Sistema de Persistencia

**Estado**: Aceptado

## Contexto

El proyecto Sky Fussion Analytics requiere manejar tipos de datos muy diversos con diferentes patrones de acceso y requisitos de consistencia:

- **Datos espaciales y relacionales entre entidades** (relaciones entre satélites, áreas geográficas, entidades analíticas)
- **Metadatos de configuración** y datos de sesión de usuario
- **Resultados de procesamiento de ML** (modelos, predicciones, métricas)
- **Datos de entrada/salida** de procesamiento de imágenes satelitales (archivos grandes, formatos GeoTIFF, etc.)
- **Logs de actividad** y historial de operaciones del sistema
- **Configuraciones de agentes** y estados de orquestación

## Decisión

Se implementa una estrategia de **Persistencia Múltiple por Caso de Uso** (polyglot persistence), donde cada tipo de dato se almacena en la tecnología mejor adaptada a sus características:

### 1. Neo4j - Base de Datos de Grafo

- **Ubicación**: `backend-node/src/services/neo4j-service.js`
- **Casos de uso**:
  - Modelado de relaciones espaciales entre entidades analíticas
  - Graph traversals para encontrar conexiones entre datos satelitales
  - Almacenamiento de metadatos de grafo con consultas de patrones espaciales
  - Historial de transformaciones y procesamiento de imágenes
- **Ventajas**: Consultas recursivas eficientes, representación natural de relaciones espaciales, performance para traversals profundos que serían costosas en SQL

### 2. SQL / Entity Framework - Datos Transaccionales

- **Ubicación**: `src/backend/` (models, config directories)
- **Casos de uso**:
  - Usuarios y autenticación/autorización
  - Configuraciones del sistema y parámetros
  - Registros de transacciones y logs operativos
  - Metadatos estructurados con relaciones Many-to-Many
- **Ventajas**: ACID guarantees, madurez de herramientas, reporting y queries complejas, soporte para migraciones versionadas

### 3. Almacenamiento JSON - Metadatos y Configuraciones Ligeras

- **Ubicación**: `data/users.json`, y archivos JSON asociados
- **Casos de uso**:
  - Configuraciones de agentes y orquestación
  - Datos de sesión temporales
  - Metadatos de definiens de modelos ML
  - Estado de procesos en formato ligero
- **Ventajas**: Sin overhead de servidor, fácil de leer/escribir, ideal para datos jerárquicos sin esquema rígido, accesible sin conexión

### 4. Almacenamiento de Archivos - Datos Binarios Satelitales

- **Ubicación**: `data/raw/`, `data/processed/`, `data/output/`
- **Casos de uso**:
  - Archivos de imagen satelital raw (GeoTIFF, etc.)
  - Imágenes procesadas y clasificadas
  - Reportes y visualizaciones generadas
  - Modelos ML serializados
- **Ventajas**: Almacenamiento escalable, integración con sistemas de archivos existentes, compatibilidad con herramientas de procesamiento de imágenes

## Diagrama de Arquitectura de Persistencia

```
┌─────────────────────────────────────────────────────────────────┐
│                    Sky Fussion Analytics                        │
├─────────────────────┬─────────────────────┬───────────────────┤
│                     │                     │                   │
│   Neo4j Graph DB    │   SQL/EF DB         │   JSON/Filestore  │
│   (relaciones espaciales) │ (transacciones) │ (metadatos ligeros)│
│                     │                     │                   │
├─────────────────────┼─────────────────────┼───────────────────┤
│                     │                     │                   │
│  data/raw/          │  data/processed     │  data/output/     │
│   (archivos binarios) │   (imágenes ML)    │   (resultados)    │
│                     │                     │                   │
└─────────────────────┴─────────────────────┴───────────────────┘
```

## Consecuencias

### Positivas

- **Optimización por caso de uso**: Cada tecnología se usa donde brinda más valor (grafo para conexiones, SQL para transacciones, JSON para configuración)
- **Rendimiento**: Las operaciones críticas aprovechan las fortalezas de cada base de datos
- **Escalabilidad independiente**: Cada almacén puede escalar de forma apropiada a su carga de trabajo
- **Ecosistema maduro**: Aprovecha años de inversión y optimización en cada tecnología

### Desafiantes

- **Sincronización entre almacenes**: Los datos deben mantenerse consistentes en Neo4j, SQL y JSON cuando cambian. Se deben implementar patterns de eventual consistency or sagas.
- **Curva de aprendizaje**: Los desarrolladores deben estar cómodos con múltiples tecnologías de bases de datos
- **Operational overhead**: Más componentes para monitorizar, hacer backup y mantener
- **Migraciones**: Los esquemas cambian de forma diferente en cada sistema; requiere coordinación cuidadosa

## Pautas de Implementación

1. **Definir la fuente de verdad**: Para cada entidad de negocio, decidir cuál es la base de datos principal y cuáles son las réplicas/cachés
2. **Patrones de escritura**: Implementar escrituras idempotentes cuando se actualicen múltiples almacenes
3. **Versionado de esquemas**: Mantener scripts de migración para SQL y Neo4j separados
4. **Monitoreo unificado**: Configurar dashboards que agreguen métricas de todas las bases de datos en una sola vista
5. **Documentación en ADR**: Cualquier cambio en la estrategia de persistencia debe ser registrado en un nuevo ADR