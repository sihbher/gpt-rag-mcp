# Arquitectura Detallada de Componentes de gpt-rag-mcp

## Tabla de contenidos

- [Stack Tecnológico Completo](#stack-tecnológico-completo)
- [Estructura de Directorios](#estructura-de-directorios)
- [Arquitectura de Componentes](#arquitectura-de-componentes)
- [Modelo de Datos](#modelo-de-datos)
- [Flujo de Ejecución de Tareas](#flujo-de-ejecución-de-tareas)
- [Motor de Procesamiento / Orquestación](#motor-de-procesamiento--orquestación)
- [Modelo de Seguridad](#modelo-de-seguridad)
- [Comparación de Modos de Despliegue](#comparación-de-modos-de-despliegue)
- [Patrones de Extensibilidad](#patrones-de-extensibilidad)

## Stack Tecnológico Completo

### Backend

| Elemento | Versión | Observaciones |
| --- | --- | --- |
| Python | `>=3.12` | Declarado en `pyproject.toml` y `uv.lock` |
| FastAPI | `0.121.1` | Dependencia declarada, aunque `src/server.py` usa `Starlette` directamente |
| MCP SDK | `1.21.0` | Base del servidor MCP |
| Starlette | Transitiva vía FastAPI/MCP | Framework web realmente instanciado |
| Uvicorn | Transitiva vía MCP | Proceso ASGI de ejecución |
| Wikipedia | `>=1.4.0` | Integración externa para búsquedas |
| Pydantic | `2.12.4` | Transitiva vía FastAPI/MCP |
| HTTPX | `0.28.1` | Transitiva vía MCP |

Notas técnicas:

- El `pyproject.toml` define el proyecto como `mcp-server`, pero `uv.lock` registra el paquete virtual local como `mcp-server-demo`. Eso sugiere arrastre de un nombre anterior.
- La dependencia `FastAPI` está presente aunque el entrypoint construye una `Starlette`. No es un problema funcional, pero sí un detalle de stack a documentar.

### Frontend

No existe frontend en este repositorio.

### Infraestructura

| Elemento | Estado |
| --- | --- |
| Azure Developer CLI | Usado vía `azure.yaml` y `azd env get-values` |
| Bicep | Declarado en `azure.yaml`, pero sin carpeta `infra/` local |
| Docker | Sí, mediante `Dockerfile` |
| ACR | Sí, para almacenamiento de imágenes |
| Azure Container Apps | Sí, como runtime objetivo |
| App Configuration | Sí, como fuente de valores de despliegue |
| CI/CD | No definido dentro del repo |

### Versiones mínimas de runtime

| Runtime | Mínimo |
| --- | --- |
| Python | 3.12 |
| Node.js | Necesario para `npx @modelcontextprotocol/inspector`, versión no fijada |
| Docker | Compatible con build de imágenes Linux/amd64 |

## Estructura de Directorios

```text
gpt-rag-mcp/
├── Analysis/                  # Documentación generada de análisis
├── scripts/                   # Automatización de despliegue y guardrails de aprovisionamiento
│   ├── deploy.ps1
│   ├── deploy.sh
│   ├── preProvision.ps1
│   └── preProvision.sh
├── src/                       # Código fuente principal
│   ├── prompts/
│   │   ├── __init__.py
│   │   └── greeting.py
│   ├── resources/
│   │   └── __init__.py
│   ├── tools/
│   │   ├── __init__.py
│   │   └── wikipedia.py
│   └── server.py
├── .azure/                    # Estado local de entornos azd y artefactos de trabajo
├── .logs/                     # Logs locales de despliegue
├── Dockerfile
├── README.md
├── azure.yaml
├── launch.json
├── pyproject.toml
├── uv.lock
└── VERSION
```

### Propósito de cada archivo/carpeta clave

| Ruta | Propósito |
| --- | --- |
| `src/server.py` | Punto de entrada ASGI y wiring del servidor MCP |
| `src/tools/wikipedia.py` | Adaptador mínimo a la librería `wikipedia` |
| `src/prompts/greeting.py` | Plantillas de saludo por estilo |
| `scripts/deploy.sh` | Flujo Bash de build/push/update para Azure |
| `scripts/deploy.ps1` | Flujo PowerShell de build/push/update para Azure |
| `scripts/preProvision.*` | Impiden aprovisionar infraestructura desde este repo |
| `azure.yaml` | Define nombre de plantilla y hooks de despliegue |
| `Dockerfile` | Crea la imagen de runtime |
| `launch.json` | Configuración de depuración local con Uvicorn |
| `.azure/` | Estado local del entorno `azd`; no es parte del producto ejecutable |
| `.logs/` | Logs locales; no son parte del runtime de negocio |

## Arquitectura de Componentes

### 1. Componente API/Servicio ASGI

- Responsabilidad principal: exponer MCP y health check.
- Implementación: `src/server.py`.

Interfaces expuestas:

| Método | Ruta | Descripción |
| --- | --- | --- |
| `GET` | `/healthz` | Devuelve estado operativo básico |
| MCP sobre HTTP | `/` | Montaje raíz del servidor MCP |
| MCP sobre HTTP | `/mcp` | Ruta alternativa/recomendada por README |

Dependencias externas:

| Dependencia | Operación |
| --- | --- |
| `mcp` | Registro de tools/prompts y runtime MCP |
| `starlette` | Hosting HTTP/ASGI |
| `wikipedia` | Solo cuando se invoca `wikipedia_search` |

Flujo interno:

```text
Request HTTP/MCP
   ▼
Starlette router
   ▼
FastMCP runtime
   ▼
tool/prompt registrado
   ▼
respuesta MCP
```

### 2. Componente Tool Library

- Responsabilidad principal: encapsular integraciones o lógica reusable para tools.
- Implementación actual: `src/tools/wikipedia.py`.

Interfaz expuesta:

```python
def search_wikipedia(query: str) -> list[str]
```

Flujo interno:

```text
query str
  ▼
search_wikipedia()
  ▼
wikipedia.search()
  ▼
list[str]
```

### 3. Componente Prompt Library

- Responsabilidad principal: encapsular plantillas de prompt reutilizables.
- Implementación actual: `src/prompts/greeting.py`.

Interfaz expuesta:

```python
def greet_user(name: str, style: str = "friendly") -> str
```

Flujo interno:

```text
name + style
   ▼
styles dict
   ▼
prompt final en texto
```

### 4. Componente de despliegue

- Responsabilidad principal: actualizar un Container App existente con una nueva imagen.
- Implementación: `scripts/deploy.sh` y `scripts/deploy.ps1`.

Dependencias externas:

| Servicio | Operación |
| --- | --- |
| Azure App Configuration | Leer valores de entorno operativos |
| Azure Container Registry | Login y push |
| Azure Container Apps | Update y restart |
| Git | Obtener short SHA para tag |

Flujo interno:

```text
leer APP_CONFIG_ENDPOINT
   ▼
resolver configName
   ▼
leer keys desde App Configuration
   ▼
login ACR
   ▼
docker build/push
   ▼
az containerapp update
   ▼
restart revision
```

## Modelo de Datos

El repositorio no define entidades persistentes, colecciones ni tablas. El “modelo de datos” real es transitorio y se limita a payloads de invocación/respuesta.

### Payloads operativos principales

| Entidad lógica | Campos | Tipo | Descripción |
| --- | --- | --- | --- |
| Health response | `status` | `string` | Devuelve `ok` |
| Tool input `add` | `a`, `b` | `int`, `int` | Operandos a sumar |
| Tool output `add` | valor de retorno | `int` | Resultado de la suma |
| Tool input `wikipedia_search` | `query` | `string` | Término a buscar |
| Tool output `wikipedia_search` | valor de retorno | `list[string]` | Títulos sugeridos por Wikipedia |
| Prompt input `greet_user_prompt` | `name`, `style` | `string`, `string` | Datos para construir el prompt |
| Prompt output `greet_user_prompt` | valor de retorno | `string` | Instrucción textual para otro modelo |

### Ejemplos JSON reales del dominio del repo

Health:

```json
{
  "status": "ok"
}
```

Entrada conceptual de `wikipedia_search`:

```json
{
  "query": "artificial intelligence"
}
```

Salida conceptual de `wikipedia_search`:

```json
[
  "Artificial intelligence",
  "History of artificial intelligence",
  "Applications of artificial intelligence"
]
```

No existen partition keys, índices, migraciones ni esquemas de base de datos porque no hay persistencia local.

## Flujo de Ejecución de Tareas

### Paso a paso técnico

1. El proceso arranca con Uvicorn ejecutando `src.server:app`.
2. Starlette inicializa el lifespan `mcp_lifespan`.
3. `mcp.session_manager.run()` abre el runtime de sesión MCP.
4. El cliente invoca `/mcp` o `/`.
5. El router delega la request al `mcp.streamable_http_app()` montado.
6. `FastMCP` identifica la tool o el prompt solicitado.
7. Si la llamada es `add`, ejecuta una función puramente local.
8. Si la llamada es `greet_user_prompt`, llama a `greet_user(name, style)`.
9. Si la llamada es `wikipedia_search`, llama a `search_wikipedia(query)`, que a su vez llama a `wikipedia.search(query)`.
10. La respuesta vuelve al cliente como payload MCP.

### Manejo de errores, reintentos y timeout

| Aspecto | Estado en el repo |
| --- | --- |
| Manejo explícito de excepciones en tools | No |
| Retries | No |
| Timeouts configurables | No |
| Circuit breaker | No |
| Logging estructurado | No |

Esto significa que cualquier error de red o de librería externa tenderá a emerger directamente al cliente o al runtime del servidor, salvo lo que maneje internamente el SDK MCP o Starlette.

## Motor de Procesamiento / Orquestación

Este repositorio no contiene un engine de pipelines, workflows o routing propio. El “motor” real es el runtime de registro y despacho de `FastMCP`.

### Cómo funciona la orquestación interna

```text
decorador @mcp.tool/@mcp.prompt
   ▼
registro en FastMCP
   ▼
mount HTTP streamable
   ▼
despacho por nombre de capacidad
   ▼
ejecución de función Python
```

### Clases base disponibles para extender

No hay clases base propias del repositorio. La superficie de extensión local es funcional y decorativa.

Firmas relevantes del código local:

```python
@mcp.tool()
def add(a: int, b: int) -> int: ...

@mcp.tool()
def wikipedia_search(query: str) -> list[str]: ...

@mcp.prompt()
def greet_user_prompt(name: str, style: str = "friendly") -> str: ...
```

### Ciclo de vida de un componente

1. Definición en Python.
2. Registro con decorador MCP.
3. Import en `src/server.py`.
4. Descubrimiento por el cliente MCP.
5. Ejecución bajo demanda.
6. Terminación al finalizar la request.

### Sistema de condiciones/routing

No existe routing condicional de negocio. El único routing es HTTP:

- `/healthz` para health.
- `/` y `/mcp` para el servidor MCP.

### Sistema de eventos/pub-sub

No existe pub/sub en el código del repositorio.

## Modelo de Seguridad

### Identidades utilizadas

No hay Managed Identity ni Service Principal definidos en el código de la aplicación. Los scripts asumen una identidad ya autenticada en Azure CLI.

### Asignaciones RBAC por servicio

| Identidad | Role/capacidad esperada | Scope |
| --- | --- | --- |
| Operador autenticado en Azure CLI | Leer App Configuration | Recurso de App Configuration |
| Operador autenticado en Azure CLI | Push/Login en ACR | Registro de contenedores |
| Operador autenticado en Azure CLI | Actualizar y reiniciar Container App | Resource Group o recurso ACA |

### Flujo de autenticación entre servicios

```text
Operador local
  ▼
az login
  ▼
scripts/deploy.*
  ├─ App Configuration
  ├─ ACR
  └─ Container Apps
```

La aplicación en runtime no implementa autenticación propia hacia los clientes.

### Secretos y configuración sensible

- `APP_CONFIG_ENDPOINT` se usa en scripts de despliegue, no en la app Python.
- Los nombres de recursos se recuperan desde App Configuration.
- No hay consumo de secretos desde Key Vault en el código de la app.

Riesgo técnico relevante: `allow_origins=["*"]` deja la política CORS demasiado abierta para escenarios de producción si el servicio queda expuesto públicamente.

## Comparación de Modos de Despliegue

| Característica | Modo Básico | Modo Enterprise |
| --- | --- | --- |
| Ingreso | Público | Restringido o privado |
| Container App | Único servicio MCP | Igual, pero con controles corporativos |
| Observabilidad | Básica o inexistente | Azure Monitor / App Insights / trazas |
| Configuración | App Configuration | App Configuration con gobierno corporativo |
| Seguridad | CORS abierto y control externo mínimo | Autenticación frontal, red privada, políticas de acceso |
| Integración GPT-RAG | Opcional, mínima | Acoplamiento formal con solución padre |

### Cambios por recurso Azure para un modo enterprise

| Recurso | Cambio | Justificación |
| --- | --- | --- |
| Container Apps | Ingreso restringido, revisiones controladas, observabilidad | Reducir exposición y mejorar operación |
| ACR | Políticas de acceso y retención | Gobierno de imágenes |
| App Configuration | RBAC estricto y etiquetas por entorno | Separación dev/test/prod |
| Red | VNet, subnets y puntos privados definidos externamente | Aislamiento de tráfico |

### Topología de red enterprise en ASCII

```text
Internet / usuarios corporativos
          │
          ▼
    Frontend GPT-RAG
          │
          ▼
   Orquestador / cliente MCP
          │
          ▼
  Subred de aplicaciones privadas
          │
          ▼
  Azure Container App: gpt-rag-mcp
        ├─ Private access a App Configuration
        └─ Pull desde ACR controlado
```

Nota: esta topología no está materializada en el repo; es la forma recomendada de endurecer el servicio cuando se inserta en una solución empresarial.

## Patrones de Extensibilidad

### Cómo añadir nuevos componentes/tools/prompts

1. Crear la función en `src/tools/` o `src/prompts/`.
2. Importarla en `src/server.py`.
3. Registrarla con `@mcp.tool()` o `@mcp.prompt()`.
4. Verificar que el tipo de retorno sea serializable.
5. Reconstruir la imagen y desplegar.

Ejemplo:

```python
from tools.my_tool import lookup_catalog

@mcp.tool()
def catalog_lookup(sku: str) -> dict:
    return lookup_catalog(sku)
```

### Cómo registrar nuevos componentes en el catálogo

No existe un archivo de catálogo externo en este repo. El “catálogo” es implícito y vive en el propio código por medio de los decoradores MCP.

### Firma exacta de métodos a implementar

El patrón real es función Python tipada y serializable:

```python
def my_tool(arg1: str, arg2: int) -> dict: ...
def my_prompt(subject: str, tone: str = "neutral") -> str: ...
```

### Paso a paso para contribuir

1. Hacer fork del repositorio.
2. Crear una rama para el cambio.
3. Implementar la tool/prompt o ajuste deseado.
4. Validar localmente con Uvicorn o MCP Inspector.
5. Actualizar documentación si cambia la superficie MCP.
6. Publicar la imagen o abrir PR según el flujo del equipo.
