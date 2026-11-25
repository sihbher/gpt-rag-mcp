# Análisis Completo de gpt-rag-mcp

Este documento consolida el conocimiento técnico verificable del repositorio `gpt-rag-mcp` para que un LLM o un desarrollador pueda usarlo como contexto único. Cuando el repositorio no define algo de forma nativa, se indica explícitamente en lugar de inferirlo.

## Tabla de contenidos

- [¿Qué es gpt-rag-mcp?](#qué-es-gpt-rag-mcp)
- [Conceptos Clave](#conceptos-clave)
- [Capacidades Completas](#capacidades-completas)
- [Prerrequisitos](#prerrequisitos)
- [Arquitectura Final](#arquitectura-final)
- [Fuera de Alcance](#fuera-de-alcance)
- [Escenarios de Ejemplo](#escenarios-de-ejemplo)
- [Índice de Samples Incluidos](#índice-de-samples-incluidos)

## ¿Qué es gpt-rag-mcp?

`gpt-rag-mcp` es un servidor MCP ligero, escrito en Python, que expone herramientas y prompts sencillos por HTTP para agregar capacidades agentic a soluciones basadas en GPT-RAG.

- Propuesta de valor principal: separar capacidades auxiliares de un agente en un microservicio MCP desplegable, consumible por clientes compatibles con Model Context Protocol.
- Repositorio: `https://github.com/Azure/gpt-rag-mcp`
- Licencia: MIT
- Runtime principal: Python 3.12
- Protocolo principal: Model Context Protocol sobre `streamable_http_app()` de `FastMCP`
- Estado de la solución: microservicio de ejemplo/plantilla, no una plataforma RAG completa por sí misma

## Conceptos Clave

### 1. Servidor MCP

El corazón del proyecto es una instancia de `FastMCP` registrada en `src/server.py`. Esa instancia publica herramientas y prompts que un cliente MCP puede descubrir e invocar.

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Demo")
```

Relación con el resto del sistema:

- `FastMCP` mantiene el registro de herramientas y prompts.
- Starlette hospeda el transporte HTTP.
- El cliente AI Chat o el orquestador GPT-RAG consumen el endpoint `/mcp`.

### 2. Herramienta MCP

Una herramienta es una función Python decorada con `@mcp.tool()`. En este repositorio hay dos herramientas reales de negocio y una utilidad trivial.

```python
@mcp.tool()
def wikipedia_search(query: str) -> list[str]:
    return search_wikipedia(query)
```

Relación con otros conceptos:

- El cliente invoca la herramienta a través del servidor MCP.
- La herramienta puede delegar en módulos locales, como `src/tools/wikipedia.py`.
- En este repo las herramientas son síncronas y no persisten estado.

### 3. Prompt MCP

Un prompt es una función decorada con `@mcp.prompt()` que devuelve texto preparado para el modelo.

```python
@mcp.prompt()
def greet_user_prompt(name: str, style: str = "friendly") -> str:
    return greet_user(name, style)
```

Relación con otros conceptos:

- Sirve como plantilla reutilizable de instrucciones.
- Se apoya en la lógica local `src/prompts/greeting.py`.
- No llama al modelo; solo genera el texto de entrada para un modelo externo.

### 4. Transporte HTTP montado en Starlette

El proyecto no expone el servidor MCP “en crudo”; lo monta dentro de una app Starlette con dos rutas MCP y un health check.

```python
app = Starlette(
    lifespan=mcp_lifespan,
    routes=[
        Route("/healthz", endpoint=healthz),
        Mount("/", app=mcp.streamable_http_app()),
        Mount("/mcp", app=mcp.streamable_http_app()),
    ],
)
```

Implicaciones:

- El mismo servidor MCP queda disponible en `/` y `/mcp`.
- `healthz` facilita readiness/liveness probes.
- La app resultante es apta para contenedores y para Azure Container Apps.

### 5. Gestión del ciclo de vida de sesión

El `lifespan` de Starlette abre y cierra el `session_manager` de MCP de forma explícita.

```python
@asynccontextmanager
async def mcp_lifespan(_app):
    async with contextlib.AsyncExitStack() as stack:
        await stack.enter_async_context(mcp.session_manager.run())
        yield
```

Esto evita dejar la infraestructura MCP parcialmente inicializada o cerrar sesiones sin cleanup.

### 6. Contrato de despliegue con GPT-RAG

Este repositorio no contiene la infraestructura completa. `README.md` y `scripts/preProvision.*` indican que la infraestructura base vive en el repo padre `Azure/GPT-RAG`. Aquí solo se despliega la imagen del servidor y se actualiza un Container App existente.

```yaml
infra:
  provider: bicep
  path: infra
  module: main
hooks:
  predeploy:
    posix:
      run: scripts/deploy.sh
```

Observación importante:

- `azure.yaml` referencia `infra/`, pero ese directorio no existe en este repo.
- El comportamiento real de despliegue depende del hook `predeploy`, no de Bicep local.

### 7. Dependencia externa a Wikipedia

La única integración de datos externa del código es la librería `wikipedia`, que consulta contenido público por Internet.

```python
import wikipedia

def search_wikipedia(query: str) -> list[str]:
    return wikipedia.search(query)
```

Consecuencias:

- Se requiere conectividad saliente a Internet.
- No existe caché local.
- No hay control de cuota, backoff o rate limiting en el código del repo.

## Capacidades Completas

### Inventario de herramientas MCP

| ID | Categoría | Descripción detallada | Inputs | Output | Configuración relevante |
| --- | --- | --- | --- | --- | --- |
| `add` | Herramienta MCP | Suma dos enteros. Es una herramienta de demostración para validar conectividad y wiring del servidor. | `a: int`, `b: int` | `int` | Ninguna |
| `wikipedia_search` | Herramienta MCP | Busca títulos de artículos en Wikipedia a partir de un texto libre. | `query: str` | `list[str]` | Requiere salida a Internet y disponibilidad del servicio Wikipedia |

### Inventario de prompts MCP

| ID | Categoría | Descripción detallada | Inputs | Output | Configuración relevante |
| --- | --- | --- | --- | --- | --- |
| `greet_user_prompt` | Prompt MCP | Devuelve una instrucción textual para que otro modelo redacte un saludo. | `name: str`, `style: str = friendly` | `str` | Estilos soportados: `friendly`, `formal`, `casual` |

### Endpoints expuestos

| Endpoint | Método | Tipo | Descripción | Respuesta |
| --- | --- | --- | --- | --- |
| `/healthz` | `GET` | HTTP | Probe operacional para confirmar que la app responde | `{"status":"ok"}` |
| `/` | MCP sobre HTTP | HTTP streamable | Montaje raíz del servidor MCP | Descubrimiento e invocación MCP |
| `/mcp` | MCP sobre HTTP | HTTP streamable | Ruta alternativa, recomendada por el README para clientes | Descubrimiento e invocación MCP |

### Módulos de código y responsabilidad

| Módulo/archivo | Tipo | Responsabilidad |
| --- | --- | --- |
| `src/server.py` | Entrada principal | Construye el servidor MCP, registra herramientas/prompts, crea la app Starlette y configura CORS |
| `src/tools/wikipedia.py` | Módulo de integración | Encapsula la llamada a la librería `wikipedia` |
| `src/prompts/greeting.py` | Módulo de prompts | Genera instrucciones de saludo por estilo |
| `scripts/deploy.sh` | Automatización | Obtiene configuración desde App Configuration, construye/pushea imagen y actualiza Container App |
| `scripts/deploy.ps1` | Automatización | Equivalente PowerShell del flujo de despliegue |
| `scripts/preProvision.sh` | Guardrail operativo | Bloquea `azd provision` y redirige a la infraestructura del repo padre |
| `scripts/preProvision.ps1` | Guardrail operativo | Mismo propósito en PowerShell |
| `azure.yaml` | Configuración de azd | Declara plantilla y hooks de despliegue |
| `Dockerfile` | Containerización | Empaqueta la app con Python 3.12, `uv`, puerto 8080 y `linux/amd64` en build script |
| `launch.json` | Depuración local | Define una configuración Uvicorn para desarrollo |

### Características del motor/framework

| Área | Estado en el repositorio |
| --- | --- |
| Registro de herramientas | Sí, vía decoradores `@mcp.tool()` |
| Registro de prompts | Sí, vía decoradores `@mcp.prompt()` |
| Transporte streamable HTTP | Sí |
| Montaje dual de rutas MCP | Sí, `/` y `/mcp` |
| Gestión de sesiones MCP | Sí, mediante `mcp.session_manager.run()` |
| Persistencia de estado | No |
| Cola de trabajos/worker | No |
| Orquestador de pipelines propio | No |
| Pub/sub | No |
| Retry/backoff nativo | No |
| Telemetría estructurada | No |

### Características de UI

No existe interfaz de usuario dentro del repositorio. El README asume una UI o AI Chat Application ya desplegada en la solución GPT-RAG externa.

### Características de API

- API pública mínima.
- Orientada a MCP, no a REST de dominio.
- `healthz` es el único endpoint REST claramente identificable.
- CORS actualmente abierto a `*`.

### Características de workers/procesos

No hay workers dedicados, procesos batch ni demonios separados. Todo el comportamiento corre dentro del mismo proceso Uvicorn/Starlette.

### Extensibilidad: cómo crear componentes personalizados

El patrón de extensión es muy simple: agregar funciones Python y decorarlas.

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Demo")

@mcp.tool()
def ping(target: str) -> str:
    return f"pong:{target}"

@mcp.prompt()
def explain_capability(topic: str) -> str:
    return f"Explain the following topic clearly: {topic}"
```

Pasos reales en este repo:

1. Crear o ampliar un módulo en `src/tools/` o `src/prompts/`.
2. Importar la función en `src/server.py`.
3. Decorarla con `@mcp.tool()` o `@mcp.prompt()`.
4. Reconstruir la imagen Docker.
5. Ejecutar `scripts/deploy.sh` o `scripts/deploy.ps1` para actualizar el Container App.

## Prerrequisitos

### Servicios cloud requeridos

Este repo no aprovisiona infraestructura completa. A partir de `README.md` y los scripts, los servicios mínimos para desplegar este componente son los siguientes.

| Servicio | Uso en este repositorio | SKU/nivel recomendado | Observaciones |
| --- | --- | --- | --- |
| Azure Container Apps | Ejecutar el servidor MCP contenedorizado | Consumption para dev; Dedicated/Workload Profiles para productivo | El repo actualiza una app existente, no la crea |
| Azure Container Registry | Almacenar la imagen `azure-gpt-rag/mcp:<tag>` | Basic para dev; Standard para equipos o producción | Requiere push de imágenes |
| Azure App Configuration | Fuente de variables de despliegue (`CONTAINER_REGISTRY_NAME`, `MCP_APP_NAME`, etc.) | Standard | Los scripts leen con label `gpt-rag` |
| Infraestructura GPT-RAG externa | Frontend, orquestador y recursos complementarios | Depende del repo padre | Este repo asume que ya existe |

### Toolchain de desarrollo

| Herramienta | Versión mínima/verificada | Motivo |
| --- | --- | --- |
| Python | 3.12 | `pyproject.toml` y Dockerfile |
| Docker CLI | Actual con soporte multiplataforma | Build y push de imagen |
| Azure CLI | Actual | Login, ACR, App Configuration y Container Apps |
| Azure Developer CLI (`azd`) | Opcional | Leer variables del entorno y flujo de despliegue |
| Git | Actual | Obtener el tag por commit y trabajar con el repo |
| VS Code | Recomendado | Depuración y trabajo local |
| Node.js + `npx` | Actual | Ejecutar `@modelcontextprotocol/inspector` |

### Permisos y roles necesarios

Roles exactos no están codificados en el repo, pero el flujo requiere como mínimo capacidades equivalentes a las siguientes:

| Alcance | Capacidad necesaria | Motivo |
| --- | --- | --- |
| App Configuration | Lectura de key-values | `az appconfig kv show --auth-mode login` |
| Azure Container Registry | Push/login | `az acr login` y `docker push` |
| Container App | Actualización y reinicio de revisiones | `az containerapp update` y `revision restart` |
| Resource Group/subscription | Lectura básica | Resolver recursos y estado |

### Cuotas y límites relevantes

| Límite o consideración | Impacto |
| --- | --- |
| Imagen construida para `linux/amd64` | Evita incompatibilidades al desplegar desde hosts ARM hacia Azure Container Apps |
| Puerto `8080` | El contenedor no usa puerto privilegiado |
| Dependencia de salida a Internet | `wikipedia_search` falla sin conectividad externa |
| Sin control de concurrencia explícito | La capacidad depende del runtime Uvicorn/Starlette y del escalado de Container Apps |
| Sin caché ni almacenamiento local | Cada búsqueda vuelve a consultar la fuente externa |

## Arquitectura Final

### Diagrama de componentes en ASCII

```text
┌─────────────────────────────┐
│ AI Chat App / Cliente MCP   │
│ (externo a este repo)       │
└──────────────┬──────────────┘
               │ HTTPS / MCP
               ▼
┌─────────────────────────────┐
│ Azure Container App         │
│ gpt-rag-mcp                 │
│ Uvicorn + Starlette         │
└──────────────┬──────────────┘
               │
     ┌─────────┼─────────┐
     │         │         │
     ▼         ▼         ▼
┌────────┐ ┌────────────┐ ┌─────────────────┐
│ add    │ │ greet_...  │ │ wikipedia_...   │
│ local  │ │ local      │ │ llama a internet│
└────────┘ └────────────┘ └────────┬────────┘
                                   │ HTTPS
                                   ▼
                           ┌───────────────┐
                           │ Wikipedia     │
                           └───────────────┘

Despliegue:
azd hook -> scripts/deploy.* -> ACR -> Container App
                 ▲
                 │ lectura de configuración
                 ▼
         Azure App Configuration
```

### Mapa de comunicación entre servicios

| Origen | Destino | Protocolo | Puerto/path | Propósito |
| --- | --- | --- | --- | --- |
| Cliente MCP o AI Chat App | `gpt-rag-mcp` | HTTPS | `/` o `/mcp` | Descubrir e invocar herramientas y prompts |
| Azure probe / balanceador | `gpt-rag-mcp` | HTTP/HTTPS | `/healthz` | Verificar salud del servicio |
| `gpt-rag-mcp` | Wikipedia | HTTPS | API pública de la librería `wikipedia` | Resolver búsquedas externas |
| `scripts/deploy.*` | Azure App Configuration | Azure CLI HTTPS | endpoint `*.azconfig.io` | Leer nombres de recursos |
| `scripts/deploy.*` | Azure Container Registry | Azure CLI + Docker | login server ACR | Autenticación y push de imagen |
| `scripts/deploy.*` | Azure Container Apps | Azure CLI HTTPS | administración ARM/ACA | Actualizar imagen y reiniciar revisión |

### Flujo de datos de extremo a extremo

1. Un cliente compatible con MCP se conecta a `/mcp`.
2. Descubre herramientas (`add`, `wikipedia_search`) y prompts (`greet_user_prompt`).
3. Invoca una capacidad concreta con argumentos JSON.
4. Starlette entrega la llamada al runtime de `FastMCP`.
5. `FastMCP` ejecuta la función Python registrada.
6. Si la función es `wikipedia_search`, el módulo local llama al paquete `wikipedia` y este consulta Internet.
7. El resultado vuelve al cliente como respuesta MCP.

### Ciclo de vida de una tarea

1. Uvicorn arranca la app `src.server:app`.
2. Starlette ejecuta `mcp_lifespan`.
3. `mcp.session_manager.run()` inicializa el runtime MCP.
4. Se atienden requests en `/`, `/mcp` y `/healthz`.
5. Cada tool/prompt ejecuta lógica síncrona de corta duración.
6. Al apagarse la app, el `AsyncExitStack` cierra el runtime MCP limpiamente.

### Esquema de base de datos

No aplica. El repositorio no define base de datos, cola, caché, vector store ni almacenamiento persistente.

### Modos de despliegue comparados

| Modo | Descripción | Ventajas | Limitaciones |
| --- | --- | --- | --- |
| Local | Uvicorn o depuración desde VS Code | Iteración rápida | No replica la topología real de Azure |
| Azure acoplado a GPT-RAG | Imagen publicada a ACR y Container App existente | Compatible con la solución recomendada por el README | Depende de infraestructura externa ya creada |
| Enterprise endurecido | Mismo servicio, pero detrás de red privada, identidades administradas y observabilidad del repo padre | Mejor seguridad y gobierno | No está definido localmente en este repo |

### Modelo de seguridad

| Área | Estado actual |
| --- | --- |
| Autenticación HTTP propia | No implementada en el código |
| Autorización/RBAC dentro de la app | No implementada |
| CORS | Abierto a `*` |
| Secretos en runtime de app | La app no consume secretos en código; los scripts sí consumen configuración de Azure |
| Identidad administrada | No definida dentro del código de la app |
| Superficie de red | Pública por defecto salvo endurecimiento externo |

Conclusión de seguridad: la seguridad real depende del perímetro de Azure y de la solución GPT-RAG que lo hospeda. El código del servidor es deliberadamente mínimo y no aplica defensa en profundidad por sí solo.

## Fuera de Alcance

| Gap nativo | Descripción | Esfuerzo estimado | Librerías/servicios sugeridos |
| --- | --- | --- | --- |
| Autenticación de clientes | No valida tokens, API keys ni identidades federadas | Medio | `fastapi-azure-auth`, Entra ID, APIM |
| Autorización fina | No hay permisos por herramienta o tenant | Medio | middleware custom, Entra claims, OPA |
| Persistencia | No guarda sesiones, resultados ni métricas | Bajo a medio | Redis, Cosmos DB, PostgreSQL |
| RAG real | No indexa documentos ni consulta un vector store | Alto | Azure AI Search, pgvector, LangChain/Semantic Kernel |
| Procesamiento de documentos | No parsea PDFs, OCR ni formularios | Alto | Azure AI Document Intelligence, `pypdf`, `unstructured` |
| Batch/colas | No hay jobs asíncronos ni fan-out nativo | Medio | Azure Queue Storage, Service Bus, Celery |
| Observabilidad | No hay logs estructurados, métricas ni trazas | Bajo a medio | OpenTelemetry, Azure Monitor, Application Insights |
| Rate limiting/reintentos | No hay protección frente a saturación externa | Bajo | `slowapi`, `tenacity`, gateway externo |
| Workflow YAML propio | El repo no define DSL declarativo | Alto | Temporal, Prefect, LangGraph, Durable Functions |
| Testing automatizado | No hay suite de pruebas en el repo | Medio | `pytest`, `httpx`, mocks |

## Escenarios de Ejemplo

Nota importante: este repositorio no define un motor YAML propio. Los siguientes ejemplos usan YAML declarativo de integración para un cliente/orquestador externo compatible con MCP, y solo invocan capacidades que sí existen en el repo: `add`, `wikipedia_search` y `greet_user_prompt`.

### 1. Banca: saludo inicial y contexto macroeconómico

```yaml
workflow: onboarding_finanzas
mcp_server: https://mi-servicio/mcp
steps:
  - call: greet_user_prompt
    args:
      name: "Gerente de Riesgo"
      style: formal
  - call: wikipedia_search
    args:
      query: "inflation"
```

### 2. Salud: búsqueda pública sobre una enfermedad

```yaml
workflow: investigacion_salud_publica
mcp_server: https://mi-servicio/mcp
steps:
  - call: wikipedia_search
    args:
      query: "diabetes mellitus"
```

### 3. Legal: contexto general sobre una jurisdicción

```yaml
workflow: contexto_legal
mcp_server: https://mi-servicio/mcp
steps:
  - call: wikipedia_search
    args:
      query: "European Union law"
```

### 4. Manufactura: investigación de estándares industriales

```yaml
workflow: manufactura_estandares
mcp_server: https://mi-servicio/mcp
steps:
  - call: wikipedia_search
    args:
      query: "ISO 9001"
```

### 5. Retail: saludo personalizado para un asistente de tienda

```yaml
workflow: retail_bienvenida
mcp_server: https://mi-servicio/mcp
steps:
  - call: greet_user_prompt
    args:
      name: "Cliente frecuente"
      style: casual
```

### 6. Educación: validación aritmética simple

```yaml
workflow: suma_educativa
mcp_server: https://mi-servicio/mcp
steps:
  - call: add
    args:
      a: 17
      b: 25
```

### 7. Sector público: enriquecimiento de contexto geográfico

```yaml
workflow: sector_publico_geografia
mcp_server: https://mi-servicio/mcp
steps:
  - call: wikipedia_search
    args:
      query: "Bogota"
```

### 8. Investigación: fan-out simple manejado por el cliente

```yaml
workflow: investigacion_paralela
mcp_server: https://mi-servicio/mcp
parallel:
  - call: wikipedia_search
    args:
      query: "artificial intelligence"
  - call: wikipedia_search
    args:
      query: "machine learning"
join: merge_titles
```

### 9. Servicios compartidos: routing condicional en el cliente

```yaml
workflow: routing_condicional
mcp_server: https://mi-servicio/mcp
conditions:
  - when: "topic != ''"
    call: wikipedia_search
    args:
      query: "${topic}"
  - when: "topic == ''"
    call: greet_user_prompt
    args:
      name: "Visitante"
      style: friendly
```

Conclusión de escenarios: el servidor sirve bien como bloque de capacidades auxiliares, incluyendo flujos multi-etapa, paralelos y condicionales cuando un cliente/orquestador externo aporta esa lógica.

## Índice de Samples Incluidos

No se encontraron carpetas `samples/` ni `examples/` en este repositorio.

| Sample | Estado | Ruta | Comentario |
| --- | --- | --- | --- |
| Samples oficiales del repo | No presentes | N/A | Los ejemplos deben inferirse desde `README.md` y el código fuente |
