# Guía General de la Solución: gpt-rag-mcp

## Tabla de contenidos

- [¿Qué es gpt-rag-mcp?](#qué-es-gpt-rag-mcp)
- [¿Qué problema resuelve?](#qué-problema-resuelve)
- [Conceptos clave](#conceptos-clave)
- [Componentes principales](#componentes-principales)
- [¿Cómo funciona el flujo completo?](#cómo-funciona-el-flujo-completo)
- [Servicios de Azure utilizados](#servicios-de-azure-utilizados)
- [Modos de despliegue](#modos-de-despliegue)
- [Ejemplo práctico](#ejemplo-práctico)
- [Resumen ejecutivo](#resumen-ejecutivo)

## ¿Qué es gpt-rag-mcp?

`gpt-rag-mcp` es como una pequeña “caja de herramientas” conectable a un asistente de IA: el asistente no necesita tener todo embebido en su propio código, sino que puede llamar a este servicio para pedir funciones concretas.

Propuesta de valor en una frase: permite añadir capacidades agentic a una solución GPT-RAG mediante un servidor MCP independiente, simple de desplegar y de consumir.

## ¿Qué problema resuelve?

| Desafío común | Cómo lo resuelve gpt-rag-mcp |
| --- | --- |
| El asistente necesita herramientas externas | Expone herramientas vía MCP para que el cliente las invoque remotamente |
| Se quiere aislar lógica auxiliar del frontend | Empaqueta esa lógica en un microservicio separado |
| Hay que desplegar rápido una capacidad nueva | Basta con agregar una función Python y redeployar la imagen |
| Se busca una interfaz estándar para agentes | Usa Model Context Protocol en vez de una API ad hoc |
| Se necesita un endpoint de salud para operaciones | Incluye `/healthz` para probes |
| Se quiere integrar con GPT-RAG ya desplegado | El README lo posiciona como componente adicional de esa solución |

## Conceptos clave

### MCP

MCP es el contrato por el cual un agente descubre y ejecuta capacidades externas.

```python
@mcp.tool()
def wikipedia_search(query: str) -> list[str]:
    return search_wikipedia(query)
```

En lenguaje simple: el agente pregunta “¿qué sabes hacer?” y el servidor responde “puedo buscar en Wikipedia, sumar números y generar un prompt de saludo”.

### Herramientas

Las herramientas son funciones invocables por el agente.

```yaml
steps:
  - call: wikipedia_search
    args:
      query: "inflation"
```

### Prompts

Los prompts son plantillas de texto listas para usar por otro modelo.

```python
@mcp.prompt()
def greet_user_prompt(name: str, style: str = "friendly") -> str:
    return greet_user(name, style)
```

### Diagrama simple de conceptos

```text
Usuario
  │
  ▼
Aplicación de chat / agente
  │ pide una capacidad
  ▼
Servidor MCP
  │ ejecuta
  ├─ herramienta
  └─ prompt
```

## Componentes principales

| Componente | Propósito | Tecnologías usadas | Puerto |
| --- | --- | --- | --- |
| Cliente MCP o AI Chat App | Consumir las capacidades del servidor | Externo al repo | N/A |
| Servidor `gpt-rag-mcp` | Exponer tools y prompts por HTTP | Python, Starlette, MCP SDK, Uvicorn | 8080 |
| Módulo de Wikipedia | Buscar contenido público | `wikipedia` para Python | Salida HTTPS |
| Azure Container Registry | Guardar la imagen del servicio | ACR | N/A |
| Azure Container Apps | Ejecutar el contenedor en Azure | ACA | 8080 interno |
| Azure App Configuration | Guardar variables operativas del despliegue | App Configuration | N/A |

### Interconexión

```text
Cliente/Agente
   │ HTTPS
   ▼
Container App con gpt-rag-mcp
   ├─ add
   ├─ greet_user_prompt
   └─ wikipedia_search ──► Wikipedia
```

## ¿Cómo funciona el flujo completo?

1. El usuario interactúa con una aplicación de chat o un agente compatible con MCP.
2. Esa aplicación se conecta al endpoint `/mcp` del servicio.
3. Descubre qué capacidades hay disponibles.
4. Decide invocar una de ellas, por ejemplo `wikipedia_search`.
5. El servidor ejecuta la función Python correspondiente.
6. Si hace falta, consulta un servicio externo, como Wikipedia.
7. Devuelve el resultado al agente.
8. El agente incorpora el resultado en la respuesta final que ve el usuario.

### Flujo de datos

```text
Usuario
  ▼
App de chat
  ▼
gpt-rag-mcp (/mcp)
  ▼
Tool o prompt
  ▼
Resultado para el agente
```

## Servicios de Azure utilizados

| Servicio | Propósito | SKU/Nivel recomendado |
| --- | --- | --- |
| Azure Container Apps | Ejecutar el microservicio MCP | Consumption para básico; Dedicated para productivo |
| Azure Container Registry | Almacenar imágenes del contenedor | Basic para básico; Standard para productivo |
| Azure App Configuration | Centralizar configuración de despliegue | Standard |
| Infraestructura GPT-RAG externa | Proveer frontend, orquestador y recursos de solución | Depende del entorno corporativo |

## Modos de despliegue

| Modo | Qué incluye | Cuándo usarlo |
| --- | --- | --- |
| Básico | Container App público, ACR y App Configuration | Demos, laboratorio, validación técnica |
| Enterprise/productivo | Lo anterior más red privada, control de acceso, observabilidad y gobierno del entorno GPT-RAG | Entornos corporativos o regulados |

### Topología enterprise simplificada

```text
Usuarios
  │
  ▼
Frontend / AI Chat
  │
  ▼
Orquestador GPT-RAG
  │
  ▼
gpt-rag-mcp en Container Apps
  │
  ├─ App Configuration
  └─ ACR

Todo idealmente dentro de una red controlada y con políticas de acceso.
```

## Ejemplo práctico

Caso representativo: un asistente financiero saluda al usuario y consulta contexto público sobre inflación.

```yaml
workflow: asistente_financiero_basico
mcp_server: https://mi-servicio/mcp
steps:
  - call: greet_user_prompt
    args:
      name: "Analista"
      style: formal
  - call: wikipedia_search
    args:
      query: "inflation"
```

Qué hace cada paso:

- `greet_user_prompt`: genera una instrucción para que otro modelo redacte un saludo apropiado.
- `wikipedia_search`: recupera títulos relacionados con inflación desde Wikipedia para enriquecer la conversación.

## Resumen ejecutivo

| Aspecto | Detalle |
| --- | --- |
| Tipo de solución | Microservicio MCP para herramientas y prompts |
| Arquitectura | Servicio Python único, sin base de datos propia |
| Nube | Azure, acoplado a infraestructura GPT-RAG externa |
| Servicios | Container Apps, ACR, App Configuration |
| Formatos soportados | HTTP para `healthz`, MCP sobre HTTP para herramientas y prompts |
| Seguridad | Básica en el código; el endurecimiento depende del entorno Azure |
| Despliegue | Docker + scripts `deploy.sh`/`deploy.ps1` + `azd` hooks |
