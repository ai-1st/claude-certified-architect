---
title: "Sección 7 — Jerarquía de configuración CLAUDE.md y reglas específicas de ruta"
linkTitle: "7. CLAUDE.md y reglas"
weight: 7
description: "Dominios 3.1, 3.3 — jerarquía user/project/directory de CLAUDE.md, patrones @import y convenciones con glob en .claude/rules/."
---

## Qué cubre esta sección

Cómo Claude Code encuentra, fusiona y aplica instrucciones persistentes a través de los ámbitos **managed-policy / user / project / subdirectory**, cómo mantener conjuntos grandes de instrucciones modulares con `@import` y `.claude/rules/`, y cómo usar globs YAML `paths` para que las convenciones se carguen solo cuando Claude toca archivos coincidentes. El examen evalúa dos modos de fallo específicos: poner estándares de equipo en ámbito de usuario (de modo que un compañero nuevo nunca los vea) y usar un `CLAUDE.md` de subdirectorio para convenciones transversales (archivos de prueba repartidos por el árbol) en vez de una regla con alcance de ruta. Sample Question 6 evalúa directamente el segundo patrón.

## Material fuente (de la guía oficial)

**3.1 — Jerarquía y organización modular.** Conocer la jerarquía de `CLAUDE.md` user / project / subdirectory; saber que `~/.claude/CLAUDE.md` es por usuario (nunca compartido vía VCS); conocer `@import` para contenido modular; conocer `.claude/rules/` como alternativa a un archivo monolítico; diagnosticar problemas de jerarquía con `/memory`.

**3.3 — Reglas específicas de ruta.** Saber que `.claude/rules/*.md` usa frontmatter YAML `paths` con globs, se carga solo cuando Claude toca archivos coincidentes y supera a un `CLAUDE.md` de subdirectorio para convenciones que atraviesan el árbol (por ejemplo `**/*.test.tsx`).

## La jerarquía CLAUDE.md

### Ubicaciones y precedencia

Claude Code carga varios archivos `CLAUDE.md` en un orden definido. El modelo mental crucial es que **los archivos se concatenan, no se sobreescriben**: cada archivo aplicable termina en el contexto de la sesión. El orden importa porque las instrucciones más cercanas a tu directorio de trabajo aparecen *más tarde* en el contexto, lo que les da prioridad efectiva.

| Ámbito | Ruta | Caso de uso | ¿Compartido por VCS? |
| --- | --- | --- | --- |
| Managed policy | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md`, Linux/WSL `/etc/claude-code/CLAUDE.md`, Windows `C:\Program Files\ClaudeCode\CLAUDE.md` | Reglas de toda la organización desplegadas por IT; no se pueden excluir | No — desplegado vía MDM / Group Policy / Ansible |
| User | `~/.claude/CLAUDE.md` | Preferencias personales en todos los proyectos | No — nunca se commitea |
| Project | `./CLAUDE.md` *o* `./.claude/CLAUDE.md` | Convenciones de equipo, comandos de build, arquitectura | Sí — commiteado |
| Local project | `./CLAUDE.local.md` | URLs personales de sandbox, datos de prueba | No — ponerlo en `.gitignore` |
| Subdirectory | `path/to/dir/CLAUDE.md` | Instrucciones para una parte del codebase | Sí si se commitea |
| User rules | `~/.claude/rules/*.md` | Reglas personales en todos los proyectos | No |
| Project rules | `.claude/rules/*.md` | Reglas de equipo por tema o por ruta | Sí |

Fuentes: docs oficiales de memoria de Claude Code en [docs.anthropic.com/en/docs/claude-code/claude-md](https://docs.anthropic.com/en/docs/claude-code/claude-md) y el espejo en [code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory).

### Cómo se fusionan los archivos al inicio de sesión

Claude camina hacia arriba por el árbol de directorios desde tu directorio de trabajo actual, recogiendo cada `CLAUDE.md` y `CLAUDE.local.md` que encuentre. El contenido se ordena de **raíz del filesystem → directorio de trabajo**; dentro de un directorio, `CLAUDE.local.md` se añade después de `CLAUDE.md`. Lanzar en `foo/bar/` produce aproximadamente:

```
1. managed-policy CLAUDE.md (if present)
2. ~/.claude/CLAUDE.md + ~/.claude/rules/*
3. /foo/CLAUDE.md
4. /foo/bar/CLAUDE.md
5. /foo/bar/CLAUDE.local.md
6. .claude/rules/*.md  (rules without `paths` load here)
```

Los archivos `CLAUDE.md` de subdirectorios **por debajo** de tu directorio de trabajo **no** se cargan al inicio: se cargan de forma perezosa cuando Claude lee un archivo en ese subdirectorio. Después de `/compact`, el `CLAUDE.md` de la raíz del proyecto se reinyecta desde disco, pero los `CLAUDE.md` anidados solo se recargan cuando Claude vuelve a tocar su subárbol.

### Diagnosticar con `/memory`

`/memory` es la herramienta de depuración más importante para este dominio. Lista cada `CLAUDE.md`, `CLAUDE.local.md` y archivo de reglas cargado en la sesión actual, permite alternar auto memory y abre cualquier archivo en tu editor. Si un archivo que esperas no aparece en la lista, Claude no puede verlo.

Flujo estándar cuando "Claude no sigue mi CLAUDE.md":

1. Ejecuta `/memory`. ¿Está listado el archivo?
2. Si no, ¿está en una ubicación cargada? `~/.claude/CLAUDE.md` aplica solo a *tus* sesiones; un compañero no lo verá.
3. Si aparece, ¿las instrucciones son específicas y no contradictorias? Dos reglas con guía contradictoria dejan a Claude elegir arbitrariamente.
4. Para reglas con alcance de ruta, usa el hook `InstructionsLoaded` para registrar cuándo se carga cada archivo y por qué.

Trampa canónica de jerarquía: un desarrollador pone estándares de equipo en `~/.claude/CLAUDE.md`, funciona perfecto para él, luego una persona nueva pregunta "¿por qué Claude sigue usando el framework de pruebas equivocado?" Corrección: mover esas reglas a `./CLAUDE.md` o `./.claude/CLAUDE.md` y commitearlas.

## Organización modular con `@import`

### Sintaxis y limitaciones

Dentro de cualquier `CLAUDE.md`, el token `@path/to/file` expande el archivo referenciado inline al inicio de sesión.

- Las rutas relativas se resuelven **desde el archivo que contiene el import**, no desde el directorio de trabajo. Las rutas absolutas y `~/...` también funcionan.
- Los imports se anidan hasta **5 saltos de profundidad**: `CLAUDE.md` → `@docs/guide.md` → `@docs/sub/detail.md` → … (cinco niveles totales como máximo).
- La primera vez que Claude Code ve imports externos en un proyecto, un diálogo de aprobación lista los archivos. Si lo rechazas una vez, permanecen deshabilitados.
- Importar **no** ahorra tokens: cada archivo importado se carga completo al inicio. Los imports son para *organización*, no reducción de contexto. Usa `.claude/rules/` con `paths` si quieres carga perezosa.

### Ejemplo: imports por paquete en un monorepo

```markdown
# Monorepo CLAUDE.md (root)
This is a pnpm workspace. Run `pnpm install` at the root.

## Package-specific standards
- API service: @services/api/STANDARDS.md
- Web frontend: @apps/web/STANDARDS.md
- Shared types: @packages/types/STANDARDS.md

See @README.md for project overview and @package.json for scripts.

# Individual Preferences (cross-worktree personal notes)
- @~/.claude/my-project-instructions.md
```

Cada maintainer de paquete posee su archivo de estándares enlazado sin obligar a todos los demás equipos a leerlo. El import `~/.claude/...` es el workaround canónico para el hecho de que un `CLAUDE.local.md` gitignored solo existe en el worktree donde lo creaste.

## Archivos temáticos en `.claude/rules/`

`.claude/rules/` es una **función documentada de Claude Code**, no una convención exclusiva de Cursor. Los docs oficiales de memoria la describen bajo "Organize rules with `.claude/rules/`". Todos los archivos `.md` en ese directorio se descubren recursivamente, en dos sabores:

1. **Reglas incondicionales** — sin frontmatter `paths`. Se cargan al inicio con la misma prioridad que `.claude/CLAUDE.md`.
2. **Reglas con alcance de ruta** — frontmatter `paths` con uno o más globs. Se cargan de forma perezosa cuando Claude lee un archivo que coincide con uno de los patrones.

Las reglas a nivel de usuario en `~/.claude/rules/` aplican a cada proyecto de tu máquina y se cargan *antes* de las reglas de proyecto, así que las reglas de proyecto ganan en conflicto.

### Schema de frontmatter (YAML)

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "src/api/**/*.tsx"
---

# API development rules
- All API endpoints must include input validation.
- Use the standard error response format.
```

Notas de la spec y de un bug YAML conocido ([anthropics/claude-code#13905](https://github.com/anthropics/claude-code/issues/13905)): `paths` debe ser una lista YAML; los patrones que empiezan con `*` o `{` **deben ir entre comillas** para ser YAML válido; omitir `paths` hace que la regla se cargue incondicionalmente.

### Semántica de glob `paths`

Matchers glob estándar: `*` coincide dentro de un segmento de ruta, `**` recurre por directorios:

| Patrón | Coincide con |
| --- | --- |
| `*.md` | Archivos Markdown solo en la raíz del proyecto |
| `**/*.ts` | Archivos TypeScript en cualquier directorio |
| `**/*.test.tsx` | Archivos de prueba React en cualquier parte del árbol |
| `src/**/*` | Cada archivo bajo `src/` |
| `src/components/*.tsx` | Solo hijos directos |
| `src/**/*.{ts,tsx}` | TypeScript y TSX bajo `src/` (brace expansion) |
| `terraform/**/*` | Todo bajo `terraform/` |

El ejemplo canónico del examen `**/*.test.tsx` funciona: es el caso de manual para una regla con alcance de ruta.

### Ejemplo: regla transversal para tests (sample Question 6)

`.claude/rules/tests.md` cubre archivos de prueba que viven junto al código fuente (`Button.test.tsx` junto a `Button.tsx`) en cualquier parte del repo:

```markdown
---
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
  - "**/*.spec.ts"
---

# Test conventions
- Use `describe` / `it` (Jest style), never `test()`.
- One assertion per `it` block; use `it.each` for tables.
- Mock with `vi.mock`; never mutate `globalThis`.
```

Cambia los `paths` por `["terraform/**/*", "**/*.tf"]` y tienes el mismo truco para archivos de infraestructura en cualquier parte del árbol.

## CLAUDE.md frente a subdirectory CLAUDE.md frente a path-rules: cuándo usar cada uno

| Necesidad | Mejor elección |
| --- | --- |
| Estándar universal de equipo (build, etiqueta del repo) | `CLAUDE.md` raíz o `.claude/CLAUDE.md` |
| Preferencias personales entre proyectos | `~/.claude/CLAUDE.md` o `~/.claude/rules/` |
| Preferencias personales en un repo | `CLAUDE.local.md` (gitignored) |
| Convención acotada a una carpeta | `path/to/CLAUDE.md` de subdirectorio (lazy) |
| Convención repartida por muchas carpetas (tests, `.tf`, `.sql`) | `.claude/rules/*.md` con glob `paths` |
| Instrucciones monolíticas grandes | Dividir en `.claude/rules/` o `@import`s |
| Debe ejecutarse en un punto específico del ciclo de vida | Un **hook**, no `CLAUDE.md` |
| Política org-wide que usuarios no pueden desactivar | Managed-policy `CLAUDE.md` + `settings.json` gestionado |

Heurística decisiva: **¿la convención sigue el *tipo/patrón* de archivo o la *ubicación* del archivo?** Patrón → regla de ruta con glob. Ubicación → `CLAUDE.md` de subdirectorio. Exactamente lo que evalúa la pregunta 6.

## Un layout de referencia

```
your-monorepo/
├── CLAUDE.md                          # High-level architecture, build commands
├── CLAUDE.local.md                    # Gitignored personal notes
├── .claude/
│   └── rules/
│       ├── tests.md                   # paths: ["**/*.test.{ts,tsx}"]
│       ├── terraform.md               # paths: ["terraform/**/*", "**/*.tf"]
│       ├── api-conventions.md         # paths: ["src/api/**/*.ts"]
│       └── security.md                # No paths -> loads every session
├── services/billing/
│   ├── CLAUDE.md                      # Lazy-loaded inside services/billing/
│   └── STANDARDS.md                   # @services/billing/STANDARDS.md from root
└── apps/web/CLAUDE.md                 # Lazy-loaded for the web app subtree

~/.claude/
├── CLAUDE.md                          # Personal preferences across projects
└── rules/preferences.md               # Personal coding preferences
```

Cada mecanismo está en su punto óptimo: archivo raíz pequeño, reglas modulares por tema, alcance de ruta para convenciones transversales, `CLAUDE.md` de subdirectorio para reglas verdaderamente ligadas a ubicación y `~/.claude/` para preferencias personales.

## Antipatrones y trampas de examen

- **Estándares de equipo en `~/.claude/CLAUDE.md`.** Te siguen a ti, no al repo; compañeros nuevos no reciben nada. Mueve convenciones compartidas a `./CLAUDE.md` o `./.claude/CLAUDE.md` y commitealas.
- **Un `CLAUDE.md` gigante de 2,000 líneas.** Todo `CLAUDE.md` entra completo en la ventana de contexto al inicio; Anthropic recomienda menos de ~200 líneas por archivo. Divide en archivos temáticos `.claude/rules/` y prefiere reglas con alcance de ruta para que la mayor parte del contenido quede fuera del contexto hasta necesitarse.
- **`CLAUDE.md` de subdirectorio para una convención transversal.** Los archivos de prueba junto al código fuente en docenas de carpetas no pueden cubrirse con archivos ligados a directorio sin duplicación absurda. Usa un único `.claude/rules/tests.md` con `paths: ["**/*.test.tsx"]`. Esto es Sample Question 6.
- **Tratar `@import` como una optimización para ahorrar tokens.** No lo es: los imports se cargan completos al inicio. El ahorro de tokens es `.claude/rules/` con `paths` (carga perezosa).
- **Reglas exigibles por máquina en `CLAUDE.md`.** `CLAUDE.md` es contexto consultivo, no enforcement. Todo lo que *debe* ocurrir (lint antes de commit, bloquear escrituras a `secrets/`) pertenece en un hook o `permissions.deny`.
- **Asumir que Claude lee `AGENTS.md`.** No lo hace nativamente. Si tu repo usa `AGENTS.md`, pon `@AGENTS.md` al inicio de `CLAUDE.md` (recomendado; portable) o `ln -s AGENTS.md CLAUDE.md` (Unix; Windows necesita Admin/Developer Mode). `/init` en un repo con `AGENTS.md` lo lee e incorpora las partes relevantes.
- **Reglas conflictivas entre archivos.** Claude puede elegir arbitrariamente. Audita periódicamente y usa `claudeMdExcludes` en `.claude/settings.local.json` para saltar archivos de equipos irrelevantes en un monorepo.

## Puntos de enfoque para el examen

- Conoce los cuatro ámbitos más managed policy y cuáles se comparten vía VCS. `~/.claude/CLAUDE.md` es la trampa: por usuario, nunca commiteado.
- Los archivos se **concatenan** desde la raíz del filesystem hacia el directorio de trabajo; `CLAUDE.local.md` se añade después de `CLAUDE.md` en el mismo directorio.
- `@import` permite rutas relativas, absolutas y con prefijo `~`; se anida **5 saltos** como máximo; **no** ahorra tokens.
- `.claude/rules/*.md` con frontmatter `paths` es el **único** mecanismo que da carga perezosa basada en glob. Sin `paths`, la regla se carga en cada sesión.
- La convención sigue *tipo/patrón* de archivo → regla con alcance de ruta. La convención sigue *ubicación* → `CLAUDE.md` de subdirectorio.
- `/memory` responde "¿qué cargó realmente Claude?"; el hook `InstructionsLoaded` añade más detalle de reglas con alcance de ruta. Objetivo: menos de **200 líneas** por `CLAUDE.md`.

## Referencias

- Anthropic, *How Claude remembers your project* — [docs.anthropic.com/en/docs/claude-code/claude-md](https://docs.anthropic.com/en/docs/claude-code/claude-md) (mirror: [code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory))
- Anthropic, *Claude Code best practices* — [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)
- Anthropic, slash commands reference — [code.claude.com/docs/en/commands.md](https://code.claude.com/docs/en/commands.md)
- `anthropics/claude-cookbooks` CLAUDE.md example — [github.com/anthropics/claude-cookbooks/blob/main/CLAUDE.md](https://github.com/anthropics/claude-cookbooks/blob/main/CLAUDE.md)
- `paths` frontmatter YAML quoting bug — [github.com/anthropics/claude-code/issues/13905](https://github.com/anthropics/claude-code/issues/13905)
- Memory vs. Settings precedence inconsistency — [github.com/anthropics/claude-code/issues/18964](https://github.com/anthropics/claude-code/issues/18964)
- AGENTS.md open standard — [agents.md](https://agents.md/); v1.1 proposal [github.com/agentsmd/agents.md/issues/135](https://github.com/agentsmd/agents.md/issues/135)
