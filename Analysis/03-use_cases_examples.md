# Casos de Uso y Ejemplos de gpt-rag-mcp

## Tabla de contenidos

- [Criterios de Selección de Casos de Uso](#criterios-de-selección-de-casos-de-uso)
- [Catálogo de Casos de Uso](#catálogo-de-casos-de-uso)
- [Tabla Resumen de Componentes por Caso de Uso](#tabla-resumen-de-componentes-por-caso-de-uso)
- [Índice de Samples Oficiales Incluidos en el Repositorio](#índice-de-samples-oficiales-incluidos-en-el-repositorio)
- [Combinaciones de Componentes Recomendadas](#combinaciones-de-componentes-recomendadas)

## Criterios de Selección de Casos de Uso

Este repositorio puede resolver problemas de tres tipos:

1. Exponer herramientas pequeñas y reutilizables a un agente vía MCP.
2. Publicar prompts reutilizables como capacidades desacopladas.
3. Enriquecer conversaciones con una búsqueda externa simple en Wikipedia.

Cómo mapear un caso de negocio a componentes del proyecto:

| Necesidad del negocio | Componente del repo |
| --- | --- |
| Operación matemática trivial | `add` |
| Generar un tono de saludo consistente | `greet_user_prompt` |
| Recuperar contexto público general | `wikipedia_search` |
| Publicar la capacidad como servicio consumible | `/mcp` en `src/server.py` |
| Ejecutarlo en Azure | `Dockerfile` + `scripts/deploy.*` |

Nota metodológica: este repositorio no trae un motor YAML ni samples oficiales. Los YAML siguientes representan plantillas de orquestación cliente para un consumidor MCP externo. Los nombres de capacidad usados sí son exactos y corresponden al catálogo real del repo: `add`, `wikipedia_search`, `greet_user_prompt`.

## Catálogo de Casos de Uso

### 1. Saludo personalizado para un asistente de retail

- Nombre: Bienvenida de tienda asistida
- Sector/Industria: Retail
- Descripción del problema: se necesita un saludo coherente y rápido para un asistente digital de tienda.
- Solución con gpt-rag-mcp: el cliente invoca `greet_user_prompt` para generar el texto base que luego usará su LLM principal.
- Componentes utilizados: `greet_user_prompt`

```yaml
workflow: retail_bienvenida
mcp_server: https://mi-servicio/mcp
steps:
  - call: greet_user_prompt
    args:
      name: "Cliente frecuente"
      style: friendly
```

- Notas de implementación: caso de baja complejidad; no requiere Internet si solo se usa el prompt.

---

### 2. Validación numérica de totales extraídos de tablas

- Nombre: Reconciliación de totales tabulares
- Sector/Industria: Finanzas / Backoffice
- Descripción del problema: un pipeline externo ya extrajo números desde una tabla y necesita validar una suma simple fuera del modelo principal.
- Solución con gpt-rag-mcp: el cliente invoca `add` y usa la respuesta como verificación exacta de subtotales o campos agregados.
- Componentes utilizados: `add`

```yaml
workflow: reconciliacion_tablas
mcp_server: https://mi-servicio/mcp
steps:
  - call: add
    args:
      a: 245
      b: 378
```

- Notas de implementación: la extracción de tablas no ocurre en este repositorio; aquí solo se valida el cálculo final.

---

### 3. Procesamiento de facturas con enriquecimiento de proveedor

- Nombre: Contexto público para facturas estructuradas
- Sector/Industria: Finanzas
- Descripción del problema: un pipeline externo procesa facturas y necesita enriquecer el contexto del proveedor, industria o país mencionado en el documento.
- Solución con gpt-rag-mcp: llamar a `wikipedia_search` con el proveedor o concepto relevante para añadir contexto público al flujo estructurado.
- Componentes utilizados: `wikipedia_search`

```yaml
workflow: facturas_contexto
mcp_server: https://mi-servicio/mcp
steps:
  - call: wikipedia_search
    args:
      query: "SAP SE"
```

- Notas de implementación: el parseo de factura y la extracción de campos estructurados deben existir fuera del repo.

---

### 4. Procesamiento de informes no estructurados en salud

- Nombre: Contexto para informe clínico no estructurado
- Sector/Industria: Salud
- Descripción del problema: un pipeline externo resume informes clínicos narrativos y necesita contexto general sobre una enfermedad o tratamiento mencionado en texto libre.
- Solución con gpt-rag-mcp: consultar `wikipedia_search` y pasar el resultado a una capa superior que redacte una respuesta segura.
- Componentes utilizados: `wikipedia_search`

```yaml
workflow: informes_salud_contexto
mcp_server: https://mi-servicio/mcp
steps:
  - call: wikipedia_search
    args:
      query: "diabetes mellitus"
```

- Notas de implementación: el tratamiento del documento no estructurado ocurre fuera del repositorio.

---

### 5. Contratos legales con enriquecimiento contextual

- Nombre: Contexto para contrato y jurisdicción
- Sector/Industria: Legal
- Descripción del problema: un pipeline externo procesa contratos y necesita enriquecer menciones a jurisdicciones, organismos o marcos regulatorios.
- Solución con gpt-rag-mcp: usar `wikipedia_search` como enriquecimiento previo a una respuesta legal supervisada.
- Componentes utilizados: `wikipedia_search`

```yaml
workflow: legal_contexto
mcp_server: https://mi-servicio/mcp
steps:
  - call: wikipedia_search
    args:
      query: "European Union law"
```

- Notas de implementación: el análisis contractual no es nativo del repo; este servicio solo aporta contexto público.

---

### 6. Investigación de estándares en manufactura

- Nombre: Lookup de estándar industrial
- Sector/Industria: Manufactura
- Descripción del problema: un asistente de operaciones necesita contexto inicial sobre una norma industrial.
- Solución con gpt-rag-mcp: llamar a `wikipedia_search` sobre la norma y usar los títulos como seed informativo.
- Componentes utilizados: `wikipedia_search`

```yaml
workflow: manufactura_norma
mcp_server: https://mi-servicio/mcp
steps:
  - call: wikipedia_search
    args:
      query: "ISO 9001"
```

- Notas de implementación: apropiado para orientación inicial, no para auditoría formal.

---

### 7. Web research básica para contenido online

- Nombre: Seed de investigación web temática
- Sector/Industria: Media / Contenido digital
- Descripción del problema: se quiere arrancar una investigación web con pistas iniciales de conocimiento general.
- Solución con gpt-rag-mcp: el cliente ejecuta `wikipedia_search` y luego usa esos títulos para búsquedas posteriores fuera del repo.
- Componentes utilizados: `wikipedia_search`

```yaml
workflow: media_seed_research
mcp_server: https://mi-servicio/mcp
steps:
  - call: wikipedia_search
    args:
      query: "renewable energy"
```

- Notas de implementación: el scraping real no está implementado en este repositorio.

---

### 8. Procesamiento masivo por lotes controlado por cliente

- Nombre: Batch de consultas temáticas
- Sector/Industria: Investigación / Analítica
- Descripción del problema: un equipo necesita lanzar varias búsquedas de contexto en serie sobre un conjunto de temas.
- Solución con gpt-rag-mcp: un orquestador externo recorre una lista y llama repetidamente a `wikipedia_search`.
- Componentes utilizados: `wikipedia_search`

```yaml
workflow: batch_busquedas
mcp_server: https://mi-servicio/mcp
for_each:
  topic:
    - "artificial intelligence"
    - "machine learning"
    - "data governance"
steps:
  - call: wikipedia_search
    args:
      query: "${topic}"
```

- Notas de implementación: el loop lo implementa el cliente, no el servidor.

---

### 9. Routing condicional de conocimiento general

- Nombre: Búsqueda con fallback de saludo
- Sector/Industria: Servicios compartidos
- Descripción del problema: si no hay una consulta de investigación, el agente debe al menos iniciar con un prompt amable.
- Solución con gpt-rag-mcp: el orquestador decide entre `wikipedia_search` o `greet_user_prompt` según si existe tema.
- Componentes utilizados: `wikipedia_search`, `greet_user_prompt`

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

- Notas de implementación: la evaluación condicional no existe en el repo; debe vivir en el cliente.

---

### 10. Fan-out/fan-in para comparación multi-documento basada en contexto público

- Nombre: Investigación paralela multi-tema
- Sector/Industria: Consultoría / Knowledge work
- Descripción del problema: comparar varios temas relacionados antes de redactar un resumen.
- Solución con gpt-rag-mcp: lanzar varias búsquedas paralelas con `wikipedia_search` y consolidar en el cliente.
- Componentes utilizados: `wikipedia_search`

```yaml
workflow: fanout_fanin_contexto
mcp_server: https://mi-servicio/mcp
parallel:
  - call: wikipedia_search
    args:
      query: "cloud computing"
  - call: wikipedia_search
    args:
      query: "edge computing"
  - call: wikipedia_search
    args:
      query: "distributed systems"
join:
  strategy: merge_titles
```

- Notas de implementación: el fan-in es responsabilidad del consumidor MCP.

---

### 11. Pre-enriquecimiento para indexación semántica (RAG)

- Nombre: Seed de términos para pipeline RAG externo
- Sector/Industria: Tecnología / RAG
- Descripción del problema: un pipeline RAG externo necesita términos semilla antes de indexar contenido.
- Solución con gpt-rag-mcp: usar `wikipedia_search` para generar títulos de referencia que luego un indexador externo procesa.
- Componentes utilizados: `wikipedia_search`

```yaml
workflow: rag_seed_terms
mcp_server: https://mi-servicio/mcp
steps:
  - call: wikipedia_search
    args:
      query: "retrieval augmented generation"
```

- Notas de implementación: la indexación semántica no es nativa del repo; este servidor solo aporta el seed inicial.

---

### 12. Grafo de conocimiento semilla desde conceptos públicos

- Nombre: Seed de nodos para knowledge graph
- Sector/Industria: Investigación / Data engineering
- Descripción del problema: se quiere obtener términos iniciales para poblar un grafo externo.
- Solución con gpt-rag-mcp: buscar un concepto y usar los títulos relacionados como nodos candidatos.
- Componentes utilizados: `wikipedia_search`, `add` opcional para contadores locales

```yaml
workflow: knowledge_graph_seed
mcp_server: https://mi-servicio/mcp
steps:
  - call: wikipedia_search
    args:
      query: "supply chain"
  - call: add
    args:
      a: 1
      b: 1
```

- Notas de implementación: la construcción del grafo ocurre fuera del repositorio.

## Tabla Resumen de Componentes por Caso de Uso

| Caso de Uso | Componentes Requeridos | Servicios Azure | Complejidad |
| --- | --- | --- | --- |
| Bienvenida de tienda asistida | `greet_user_prompt` | ACA, ACR, App Configuration | Baja |
| Reconciliación de totales tabulares | `add` | ACA, ACR, App Configuration | Baja |
| Contexto público para facturas estructuradas | `wikipedia_search` | ACA, ACR, App Configuration | Baja |
| Contexto para informe clínico no estructurado | `wikipedia_search` | ACA, ACR, App Configuration | Baja |
| Contexto para contrato y jurisdicción | `wikipedia_search` | ACA, ACR, App Configuration | Baja |
| Lookup de estándar industrial | `wikipedia_search` | ACA, ACR, App Configuration | Baja |
| Seed de investigación web temática | `wikipedia_search` | ACA, ACR, App Configuration | Media |
| Batch de consultas temáticas | `wikipedia_search` + orquestador externo | ACA, ACR, App Configuration | Media |
| Búsqueda con fallback de saludo | `wikipedia_search`, `greet_user_prompt` + lógica externa | ACA, ACR, App Configuration | Media |
| Investigación paralela multi-tema | `wikipedia_search` + agregación externa | ACA, ACR, App Configuration | Media |
| Seed de términos para pipeline RAG externo | `wikipedia_search` + indexador externo | ACA, ACR, App Configuration | Alta |
| Seed de nodos para knowledge graph | `wikipedia_search` + persistencia externa | ACA, ACR, App Configuration | Alta |

## Índice de Samples Oficiales Incluidos en el Repositorio

No existen carpetas `samples/` ni `examples/` en el árbol actual del repositorio.

| Número/nombre | Descripción | Componentes que demuestra | Ruta relativa |
| --- | --- | --- | --- |
| N/A | No hay samples oficiales en el repo | N/A | N/A |

## Combinaciones de Componentes Recomendadas

### Patrones frecuentes

| Patrón | Recomendación |
| --- | --- |
| Prompt + herramienta | Combinar `greet_user_prompt` con `wikipedia_search` cuando el agente necesite tono inicial y contexto público |
| Tool simple para smoke test | Usar `add` para validar que el cliente MCP ve y ejecuta herramientas correctamente |
| Enriquecimiento externo | Usar `wikipedia_search` como paso previo antes de un pipeline mayor en otro sistema |
| Despliegue controlado | Mantener el servicio desacoplado y desplegarlo como Container App dedicado |

### Anti-patrones a evitar

| Anti-patrón | Motivo |
| --- | --- |
| Tratar este repo como plataforma RAG completa | No tiene indexación, vector store ni retrieval propio |
| Usar Wikipedia como fuente autoritativa final en dominios regulados | Solo aporta contexto público general |
| Exponer el servicio públicamente sin controles adicionales | El código no implementa autenticación y CORS es permisivo |
| Esperar batch, fan-out o routing nativo | Esa lógica debe implementarse en el cliente/orquestador |
