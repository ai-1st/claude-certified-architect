---
title: "Sección 6 — Integración MCP: errores, servidores y recursos"
linkTitle: "6. Integración MCP"
weight: 6
description: "Dominios 2.2, 2.4 — sobres estructurados de error (isError, isRetryable), ámbitos de .mcp.json y expansión de env, recursos frente a herramientas."
---

## Qué cubre esta sección

Dos dominios MCP críticos operativamente: cómo las herramientas deben *reportar fallos* para que el agente pueda recuperarse con inteligencia (2.2), y cómo los servidores y recursos deben conectarse a Claude Code para que un equipo obtenga las capacidades correctas en el ámbito correcto (2.4). MCP es un contrato entre un servidor y un agente: la calidad de ese contrato (errores, descripciones, recursos) determina si el agente toma buenas decisiones o se agita sin rumbo.

## Material fuente (de la guía oficial)

### 2.2 Respuestas de error estructuradas

Conocimiento: las cuatro clases de error (transient, validation, business, permission); por qué respuestas uniformes `"Operation failed"` rompen la recuperación; reintentable frente a no reintentable.

Habilidades: devolver `errorCategory`, `isRetryable` y una descripción legible por humanos; marcar violaciones de reglas de negocio con `retriable: false` más una explicación amigable para el cliente; hacer que subagentes se recuperen localmente de errores transient y escalen solo lo que no pueda resolverse localmente (con resultados parciales y lo intentado); distinguir *fallos de acceso* de *resultados vacíos válidos*.

### 2.4 Integración de servidores

Conocimiento: ámbito de proyecto (`.mcp.json`) frente a usuario (`~/.claude.json`); expansión `${VAR}` para secretos; herramientas de todos los servidores se descubren al conectar y están disponibles simultáneamente; los recursos exponen catálogos de contenido para reducir llamadas exploratorias a herramientas.

Habilidades: configurar servidores compartidos en `.mcp.json` con expansión de env-var; configurar servidores personales en ámbito de usuario; escribir descripciones ricas de herramientas para que el agente no vuelva a integradas como `Grep`; elegir servidores comunitarios (Jira, GitHub, Postgres) sobre personalizados; exponer catálogos de contenido como recursos.

## MCP a 30,000 pies

El [Model Context Protocol](https://modelcontextprotocol.io/specification/latest) es un protocolo abierto JSON-RPC 2.0 que conecta hosts LLM con proveedores de capacidades (servidores) mediante una sesión con estado y negociación de capacidades. Los servidores exponen tres primitivas:

| Primitiva | Controlada por | Propósito | Ejemplo |
| --- | --- | --- | --- |
| **Tool** | Modelo | Acción ejecutable con efectos secundarios | `create_issue`, `run_query`, `send_email` |
| **Resource** | Aplicación / usuario | Contenido de solo lectura identificado por URI | `jira://issues/PROJ-123`, `postgres://schema/orders` |
| **Prompt** | Usuario | Plantillas preconstruidas invocadas por elección del usuario (slash commands, opciones de menú) | `/code-review`, `/summarize-incident` |

Transportes definidos en la especificación: **stdio** (subproceso local), **Streamable HTTP** (llamado `streamable-http` en la spec, alias `http` en config de Claude Code) y el transporte legado **SSE**. Consulta la [referencia de schema](https://modelcontextprotocol.io/specification/2025-11-25/schema) para el wire format.

## Respuestas de error estructuradas

### Taxonomía de categorías de error

| Categoría | Qué significa | `isRetryable` típico | Qué debe hacer el agente |
| --- | --- | --- | --- |
| `transient` | Timeout, 5xx, rate limit, connection reset | `true` | Backoff y reintento local dentro del subagente |
| `validation` | Schema mismatch, enum incorrecto, campo faltante | `false` | Reformular input; no reintentar la misma llamada |
| `business` | Violación de política/límite/estado ("refund > $500", "ticket already closed") | `false` | Exponer la explicación para el cliente; detener |
| `permission` | 401/403, scope faltante, RBAC denegado | `false` | Escalar a coordinador o humano |
| `not_found` | Consulta válida, cero coincidencias | `false` (e `isError` es debatible; ver abajo) | Tratar como resultado vacío legítimo, no como fallo |

### Flag `isError`: schema y ejemplo

La forma oficial de [`CallToolResult`](https://modelcontextprotocol.io/specification/2025-11-25/schema):

```ts
interface CallToolResult {
  content: ContentBlock[];
  structuredContent?: { [key: string]: unknown };
  isError?: boolean;  // default false
  _meta?: { [key: string]: unknown };
}
```

La especificación es explícita: *los errores originados por la propia herramienta* deben reportarse dentro del resultado con `isError: true`, **no** como error de protocolo JSON-RPC. Los errores de protocolo son invisibles para el modelo, y el LLM necesita ver el error para autocorregirse. Los errores de protocolo se reservan para "tool not found" o "server doesn't support tool calls."

Un fallo de herramienta bien estructurado se ve así:

```json
{
  "isError": true,
  "content": [
    { "type": "text", "text": "Refund of $750 exceeds the $500 single-transaction policy. Ask the customer to split the refund or open a manager-approval ticket." }
  ],
  "structuredContent": {
    "errorCategory": "business",
    "isRetryable": false,
    "code": "REFUND_LIMIT_EXCEEDED",
    "limit": 500,
    "requested": 750,
    "customerMessage": "We can only process refunds up to $500 in one transaction."
  }
}
```

Compáralo con el antipatrón: `{ "isError": true, "content": [{ "type": "text", "text": "Operation failed" }] }`, que obliga al agente a reintentar a ciegas o rendirse.

### Árbol de decisión reintentable frente a no reintentable

```
isError === true ?
├── No  → success path (or empty-result path if content is intentionally empty)
└── Yes
    ├── errorCategory === "transient" && isRetryable === true
    │     → backoff + retry locally (subagent), cap attempts (e.g. 3)
    ├── errorCategory === "validation"
    │     → re-plan with corrected input; do NOT replay the same call
    ├── errorCategory === "permission"
    │     → escalate to coordinator with the missing scope/role
    └── errorCategory === "business"
          → surface customerMessage; stop; do NOT retry
```

Los metadatos estructurados hacen ejecutable este árbol. Sin `errorCategory` e `isRetryable`, el agente tiene que *adivinar* si vale la pena reintentar un fallo, que es la mayor causa de bucles runaway de "tool storm".

### Recuperación local de subagente frente a escalación al coordinador

Un subagente posee la recuperación local de fallos transient: reintentar con backoff, caer a un endpoint secundario, servir una lectura cacheada obsoleta, y solo escala hacia arriba cuando la clase de error no es reintentable *o* se agotó el presupuesto local de reintentos. Cuando escala, devuelve un sobre estructurado que incluye el `errorCategory`/`isRetryable` final, una descripción para los logs del coordinador, **resultados parciales** recogidos antes del fallo y **qué se intentó** (qué endpoints, cuántos reintentos) para que el coordinador no repita el trabajo.

Críticamente, un *resultado vacío válido* (por ejemplo "no hay órdenes que coincidan con este filtro") **no** es un error. Confundir "no pude ejecutar la consulta" con "la consulta devolvió cero filas" hace que el coordinador reintente innecesariamente o reporte mal el estado al usuario.

## Configuración de servidores MCP en Claude Code

### Ámbitos: local / project / user

Los docs MCP de [Claude Code](https://docs.claude.com/en/docs/claude-code/mcp) definen tres ámbitos:

| Ámbito | Almacenamiento | ¿Compartido por git? | Usar para |
| --- | --- | --- | --- |
| `local` (por defecto) | `~/.claude.json`, indexado por ruta de proyecto actual | No | Servidores personales/experimentales en un proyecto, secretos que no quieres compartir |
| `project` | `.mcp.json` en la raíz del repo | **Sí** | Herramientas compartidas por el equipo (Jira, APIs internas, deploy bot del equipo) |
| `user` | `~/.claude.json`, cross-project | No | Utilidades personales que quieres en todas partes (tu scratch DB, tu servidor de notas) |

Añadir con la CLI:

```bash
claude mcp add --scope project --transport http jira https://mcp.atlassian.com/v1/sse
claude mcp add --scope user    --transport stdio notes -- npx -y @me/notes-mcp
claude mcp add --scope local   --transport stdio scratch -- python ./scratch_server.py
```

Cuando existe el mismo nombre de servidor en varios ámbitos, **local gana, luego project, luego user**, después servidores provistos por plugins y luego conectores de `claude.ai`. Esa precedencia permite que un desarrollador sobrescriba localmente una config compartida por el equipo sin editar `.mcp.json`.

### Schema de `.mcp.json` (con ejemplo)

Un `.mcp.json` de ámbito proyecto vive en la raíz del repo y se commitea. Schema mínimo:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp",
      "headers": {
        "Authorization": "Bearer ${GITHUB_PERSONAL_ACCESS_TOKEN}"
      }
    },
    "postgres-readonly": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL_RO}"],
      "env": {
        "PGSSLMODE": "require"
      }
    },
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    }
  }
}
```

El `type` de cada entrada es uno de `stdio`, `http` (alias `streamable-http`) o `sse`. Para `stdio` suministras `command` + opcionales `args` y `env`. Para `http`/`sse` suministras `url` y opcionalmente `headers`. Los servidores de ámbito proyecto piden aprobación al usuario la primera vez que una sesión los carga (reiniciar con `claude mcp reset-project-choices`).

### Expansión de variables de entorno

Claude Code expande dos formas dentro de `.mcp.json`:

- `${VAR}` — valor de `VAR`; **falla al parsear** si no está definida
- `${VAR:-default}` — valor de `VAR` si existe, si no `default`

La expansión funciona en `command`, `args`, `env`, `url` y `headers`. Patrón recomendado: commitear `.mcp.json` con placeholders `${GITHUB_TOKEN}` y mantener tokens reales en el shell de cada desarrollador, runner de CI o 1Password CLI. Gotcha: `${CLAUDE_PROJECT_DIR}` está definido en el entorno del *servidor*, no en el de Claude Code; referenciarlo en un `command` o `args` de nivel superior normalmente necesita `${CLAUDE_PROJECT_DIR:-.}` como default.

### Verificar conexión (comando `/mcp`)

Dentro de Claude Code, `/mcp` lista cada servidor conectado, el conteo de herramientas/recursos/prompts que anuncia, su estado de auth y cualquier backoff de reconexión en progreso. Los servidores HTTP/SSE reconectan automáticamente hasta cinco veces con backoff exponencial; los stdio, al ser subprocesos locales, no. `/mcp` también es donde ejecutas el flujo OAuth para servidores remotos que devuelven `401`/`403` con header `WWW-Authenticate`.

## Herramientas MCP frente a recursos MCP

### Cuándo modelar algo como herramienta

Las herramientas son **controladas por el modelo** y pueden tener efectos secundarios. Usa una herramienta cuando el agente debe *decidir* hacer algo:

- `create_jira_issue`, `update_pr_status`, `run_sql(query)`, `send_slack_message`
- Cualquier cosa que mute estado, llame una API externa o requiera argumentos sobre los que el modelo deba razonar

### Cuándo modelarlo como recurso

Los recursos son **controlados por la aplicación**, de solo lectura, identificados por URIs e idealmente cacheables. Usa un recurso cuando el agente se beneficia de *contexto pasivo* sin tener que pedirlo:

- Un directorio de todos los tickets Jira abiertos del sprint actual → `jira://sprints/current/issues`
- El schema Postgres de la base de datos de órdenes → `postgres://schema/public/orders`
- El contenido de un runbook o design doc → `confluence://pages/12345`

Como los recursos son direccionables por URI y no mueven estado, cuestan menos en reintentos y son más amigables con caching que envolver los mismos datos como una llamada de herramienta.

### Ejemplos (catálogo de issues, introspección de schema)

El patrón de "catálogo de contenido" de la guía: en vez de forzar al agente a llamar `list_issues` y luego `get_issue` para cada coincidencia (un N+1 exploratorio), el servidor expone un recurso `issues://summary` que el host puede adjuntar automáticamente. El agente ve el catálogo en contexto y salta directamente a la llamada `get_issue` relevante. La misma idea aplica a la introspección de schema: exponer `db://schema` como recurso convierte "what columns does `orders` have?" en una respuesta sin llamadas a herramientas.

## Elegir servidores comunitarios frente a personalizados

### Cuándo construir uno propio

Construye un servidor MCP personalizado solo cuando la integración sea **específica del equipo** y no exista una opción comunitaria mantenida: un microservicio interno, tu pipeline de deploy, un CRM casero. El coste de un servidor personalizado no es el código; es el mantenimiento continuo: deriva de schemas, refresh de auth, upgrades de transporte, revisión de seguridad.

### Servidores populares que conviene conocer

El repo [`modelcontextprotocol/servers`](https://github.com/modelcontextprotocol/servers) contiene implementaciones de referencia y enlaces al [MCP Registry](https://registry.modelcontextprotocol.io/) oficial. Nombres que conviene recordar: **Filesystem** (operaciones de archivo sandboxed), **Fetch** (URL → texto amigable para LLM), **Git**, **Memory** (persistencia de knowledge graph), **Sequential Thinking**, más **GitHub** alojado por proveedor (`api.githubcopilot.com/mcp`), **Postgres**, **Slack**, **Sentry**, **Notion**, **Atlassian/Jira**, **Stripe**, **PayPal**, **HubSpot**, **Asana** y **Playwright**. Directorios comunitarios ([mcpforge.org](https://www.mcpforge.org/directory), [mcpindex.net](https://mcpindex.net/), [mcpdir.dev](https://mcpdir.dev/)) catalogan miles más, pero el registro oficial es la fuente de verdad para procedencia.

Una habilidad final que la guía destaca: **mejorar las descripciones de herramientas MCP**. Si tu herramienta MCP de Jira dice "search Jira" mientras la integrada `Grep` está descrita con lujo de detalle, el modelo buscará con `Grep` contra una caché local. Las descripciones de herramientas son la única señal del modelo para elegir herramienta: escríbelas como el docstring de una función que el modelo nunca ha visto.

## Puntos de enfoque para el examen

- `isError: true` vive dentro de `CallToolResult`, **no** como error JSON-RPC, porque el modelo debe *ver* el fallo para recuperarse.
- Incluye siempre `errorCategory` e `isRetryable` en `structuredContent`; "Operation failed" es la respuesta incorrecta canónica.
- Distingue fallos de acceso de resultados vacíos; `isError: false` con `content` vacío es un caso válido y común.
- `.mcp.json` es de ámbito proyecto y se commitea; `~/.claude.json` es local/de usuario y privado.
- La expansión `${VAR}` y `${VAR:-default}` sirve para mantener secretos fuera de git, no para lógica de config en runtime.
- Tools = controladas por el modelo, con efectos secundarios; Resources = controlados por la aplicación, solo lectura, direccionables por URI.
- Prefiere servidores comunitarios; los personalizados existen solo para workflows específicos del equipo.
- Los subagentes se recuperan localmente de errores transient; solo errores no recuperables (con resultados parciales y log de intentos) se propagan al coordinador.

## Referencias

- MCP Specification (latest): https://modelcontextprotocol.io/specification/latest
- MCP Schema (2025-11-25): https://modelcontextprotocol.io/specification/2025-11-25/schema
- MCP Server Primitives Overview: https://modelcontextprotocol.io/specification/2025-11-25/server
- Claude Code — Connect to tools via MCP: https://docs.claude.com/en/docs/claude-code/mcp
- Claude Code — MCP installation scopes: https://docs.claude.com/en/docs/claude-code/mcp#mcp-installation-scopes
- Claude Code — Environment variable expansion in `.mcp.json`: https://docs.claude.com/en/docs/claude-code/mcp#environment-variable-expansion-in-mcp-json
- Reference servers: https://github.com/modelcontextprotocol/servers
- MCP Registry: https://registry.modelcontextprotocol.io/
