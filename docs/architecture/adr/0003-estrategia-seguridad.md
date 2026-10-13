# 0003 - Estrategia de Seguridad

**Estado**: Propuesta

## Contexto

Sky Fussion Analytics maneja datos satelitales sensibles y potentially información empresarial de clientes. El sistema debe soportar:

- **Autenticación de humanos**: Desarrolladores, analistas geospaciales, operadores del sistema
- **Autenticación de agentes**: Agentes orquestados dentro del framework que ejecutan tareas automáticas
- **Integración con LLM**: Consumo de servicios de Large Language Models que requieren API keys y tokens
- **Múltiples entornos**: Desarrollo, testing, staging y producción con diferentes requisitos de seguridad
- **Acceso externo**: APIs consumidas por aplicaciones frontend y terceros autorizados

## Decisión

Se propone una estrategia de seguridad **por capas con autenticación multi-principal y autorización basada en roles** (RBAC extendida). Los componentes clave son:

### 1. Autenticación

#### A. Humanos (Desarrolladores, Analistas, Operadores)

- **Mecanismo**: JWT (JSON Web Tokens) firmados con clave simétrica o asimétrica
- **Fuente de credenciales**: 
  - Credenciales locales (username/password almacenado con hash en SQL)
  - Proveedores OAuth2/OIDC (Google, Azure AD) via `.env` configuration
  - API Keys para consola de desarrollador
- **Flujo**:
  1. Usuario credentials → `/api/auth/login` (Node.js middleware)
  2. Validador → Verificación contra SQL Server / Neo4j
  3. Generación de token JWT con claims de rol y expiration
  4. Token retornado al cliente para uso en `Authorization: Bearer <token>` header

#### B. Agentes Orquestados

- **Mecanismo**: Tokens de servicio (service tokens) con firma HMAC
- **Almacenamiento**: `.env.example` con variables `AGENT_TOKEN_SECRET`
- **Flujo**:
  1. Agente se identifica presentando token en header `X-Agent-Token`
  2. Middleware valida firma y permisos asociados
  3. Acceso concedido o denegado según rol del agente

#### C. Integración LLM

- **Mecanismo**: API Keys management via `.env` variables nunca commiteadas
- **Almacenamiento**: Variables de entorno en nivel de contenedor Docker, nunca en código fuente
- **Flujo**:
  1. Servicio backend solicita token a proveedor LLM
  2. `.env` provee la key temporal o rotada
  3. Request al proveedor LLM incluye header `Authorization: Bearer <LLM-key>`
  4. Never hardcode en source code

### 2. Autorización y Control de Acceso

#### A. Roles y Permisos (RBAC Extendido)

| Rol | Permisos Principales |
|---|---|
| `system-admin` | Full access to all endpoints, user management, configuration |
| `geospatial-analyst` | Analysis APIs, report generation, data query |
| `developer` | API access (read/write), agent management, debugging |
| `agent-prediction` | Execute prediction tasks, read own results |
| `agent-reporting` | Generate reports, read aggregated data |
| `anonymous` | Public metadata access, no authenticated endpoints |

#### B. Middleware Pipeline (Orden de ejecución)

1. **CORS** - Configuración por origen (solo dominios autorizados)
2. **Rate Limiting** - Límite por IP y por token (prevenir abuso de endpoints)
3. **Autenticación JWT** - Validación de token y carga de claims
4. **Autorización por Rol** - Verificación de que el claim rol tiene acceso al endpoint
5. **Validación de Input** - Sanitización y validación de todos los parámetros entrantes

#### C. Endpoints Críticos y sus Protecciones

| Endpoint | Protección |
|---|---|
| `POST /api/auth/login` | Rate limiting estricto, bloqueo temporal después de N intentos fallidos |
| `POST /api/analysis/request` | Autenticación obligatoria, validación de payload, rate limiting por usuario |
| `GET /api/analysis/{id}` | Autorización: usuario debe ser dueño o tener rol `geospatial-analyst`/`system-admin` |
| `POST /api/agents/*` | Autenticación de agente + validación de permisos de tarea |
| `GET /api/llm/status` | API Key requerida en header, rate limiting por key |
| `GET /api/system/config` | Solo `system-admin` |

### 3. Capa de Defensa en Profundidad

- **HTTPS everywhere**: Todos los endpoints forzados a HTTPS, incluido el desarrollo local via certificado autofirmado o `https-local` tooling
- **Headers de seguridad**: 
  - `X-Content-Type-Options: nosniff`
  - `X-Frame-Options: DENY`
  - `X-XSS-Protection: 1; mode=block`
  - `Strict-Transport-Security` con preload
- **Helmet middleware**: En Node.js Express/.NET pipelines para establecer headers seguros
- **Input validation**: Bibliotecas de validación (zod, FluentValidation) en todas las rutas
- **Error handling**: No exponer stack traces o detalles de base de datos en respuestas de error al cliente
- **Secret management**: 
  - `.env.example` template que incluya todas las variables necesarias (sin valores reales)
  - Secrets inyectados en contenedores Docker via `docker-compose.yml` environment section
  - Rotación periódica de keys y tokens
  - Nunca commitear archivos `.env` (añadidos a `.gitignore`)

### 4. Consideraciones Específicas del Dominio

#### Datos Espaciales y Geo-Sensibilidad

- Validación de coordenadas y bounding boxes en todos los inputs
- Sanitización de queries que interactúan con Neo4j para prevenir inyección de grafo
- Rate limiting más agresivo en endpoints que retornan datos espaciales masivos

#### Pipeline de ML

- Aislamiento del entorno de ejecución Python para prevenir exposición de librerías del sistema
- Validación de tamaños de archivo entrantes para prevenir DoS en el sistema de archivos
- Escaneo de modelos ML entrantes para detectar código malicioso

## Consecuencias

### Positivas

- **Seguridad por capas**: Si un control falla, los demás siguen protegiendo el sistema
- **Flexibilidad de autenticación**: Soporta tanto humanos con OAuth como agentes con tokens HMAC
- **Auditoría completa**: Los claims en tokens y logs de middleware proveen rastro de quién hizo qué y cuándo
- **Cumplimiento**: Estructura lista para requisitos de regulaciones (GDPR, etc.) mediante datos de auditoría apropiados

### Desafiantes

- **Operational complexity**: Gestionar múltiples mecanismos de auth (JWT, service tokens, API keys) requiere orquestación cuidadosa
- **UX considerations**: Flujo de login y token renewal debe ser fluido para humanos, pero seguro
- **Overhead de validación**: Cada endpoint requiere validación explícita, lo que puede ralentizar el desarrollo inicial
- **Fricción en desarrollo**: Los desarrolladores deben configurar `.env` files localmente, lo que puede causar fricción en onboarding inicial

## Estado Próximo y Roadmap

1. **Fase 1 (Actual)**: Implementar autenticación JWT para humanos + service tokens para agentes (basado en patrones existentes en `backend-node` y `src/backend` middleware)
2. **Fase 2**: Agregar integración OAuth2/OIDC opcional via proveedores de identidad corporativa
3. **Fase 3**: Implementar auditoría completa y reporte de actividades sospechosas
4. **Fase 4**: Evaluación de requisitos de cifrado en reposo para datos satelitales sensibles

## Relación con otros ADRs

- **Depende de ADR-0001 (Patrón Arquitectónico)**: La arquitectura modular requiere que cada módulo implemente seguridad de forma consistente pero adaptada a su contexto (API Node.js vs. .NET Core vs. Python ML pipeline)
- **Soporta ADR-0002 (Persistencia)**: Las decisiones de control de acceso influyen en qué datos pueden leerse/escribirse en Neo4j, SQL y JSON storage respectively
- **Fundamenta la estructura de diagramas**: Los diagramas C4 deben reflejar los componentes y flujos de seguridad definidos en este ADR