# Definiciones

| Mecanismo | Definición |
|---|---|
| **Skill** | Directorio con un archivo `SKILL.md` (frontmatter YAML + instrucciones en markdown) que empaqueta conocimiento, procedimientos y opcionalmente scripts ejecutables para una tarea específica. |
| **MCP (Model Context Protocol)** | Estándar abierto que conecta Claude Code a herramientas y fuentes de datos externas (APIs, bases de datos, servicios) mediante un servidor que expone *tools*, *resources* y *prompts*. |
| **.claude/rules/** | Directorio de archivos `.md` con instrucciones de proyecto, segmentadas por tema o por ruta (`path-scoped`), en vez de un único CLAUDE.md gigante. |
| **CLAUDE.md** | Archivo markdown del proyecto (`./CLAUDE.md` o `./.claude/CLAUDE.md`), personal (`~/.claude/CLAUDE.md`) o de política gestionada por la organización, con contexto general: convenciones, comandos de build, estructura del repo. |

**Diferencia explícita en una frase:** CLAUDE.md y `.claude/rules/` son **contexto pasivo cargado automáticamente** (texto que Claude lee) — el CLAUDE.md de la raíz y de los directorios superiores se carga completo al inicio, y las reglas de `.claude/rules/` también, salvo las que tengan `paths:`, que se cargan solo al trabajar con archivos que coincidan con esa ruta; Skill es **procedimiento activado por relevancia** (instrucciones + posibles scripts); MCP es **acción sobre sistemas externos** (tools reales que ejecutan algo fuera del modelo).

---

# Scope

**Scope** (alcance) es el **nivel desde el que se carga y se aplica una configuración** en Claude Code. No describe *qué* hace un mecanismo, sino *desde dónde rige*. Cada scope responde a dos preguntas:

1. **¿En qué proyectos está disponible?** → solo el proyecto actual, o todos los proyectos de tu máquina.
2. **¿Se comparte con el equipo?** → se versiona en el repo (visible para el equipo) o queda privado para ti.

> Estas definiciones de scope son transversales: son el eje común que organiza CLAUDE.md, las skills, los MCP y las rules. Por eso van aquí, **fuera de las tablas** de cada mecanismo.

## Tipos de scope

> Los nombres exactos cambian un poco según el mecanismo, pero todos se reducen a estos niveles. En MCP se eligen con la bandera `--scope`.

- **Local** — Solo tú, solo en el proyecto actual. La configuración se guarda **fuera del repo** (por ejemplo, MCP la guarda en `~/.claude.json` bajo la ruta de ese proyecto). **No se comparte por git.** ❌
- **Project / Compartido con el equipo** — Para el proyecto actual y compartido con todo el equipo. La configuración vive en archivos **dentro del repo** (`CLAUDE.md`, `.claude/...`, `.mcp.json`) y **sí se commitea.** ✅
- **User (antes "global")** — Solo tú, pero en **todos tus proyectos** de la máquina. Vive en tu home (`~/.claude/...` o `~/.claude.json`). **No se comparte por git.** ❌
- **Managed policy / Organización** — Lo despliega IT/DevOps para toda la organización; se carga antes que todo lo demás y no se puede excluir. Se distribuye por MDM/Group Policy, **no por git.** ❌

**Correcciones a la idea inicial (que tenía errores):**

- ❌ *"local: sí se puede hacer commit para compartir con el equipo"* → **Falso.** El scope que se commitea y se comparte es **project**, no **local**. El scope **local** es privado y se guarda fuera del repo.
- ✅ *"local: solo tú en ese proyecto"* → Correcto, esa es la definición de **local**.
- ✅ *"compartido entre el equipo: solo de un proyecto específico"* → Correcto, eso es el scope **project**.
- ⚠️ *"global: no se puede commitear, sirve para todos los proyectos"* → La idea es correcta, pero el nombre actual es **user**. En versiones antiguas de MCP este scope se llamaba "global", y el actual "local" se llamaba "project"; de ahí viene buena parte de la confusión.

## Scope de Claude md

| Scope | Explicación del alcance | Ruta donde se configura | Comando para configurar | ¿Se puede hacer git commit para compartir con el equipo? |
|---|---|---|---|---|
| **Managed policy** (organización) | Instrucciones para toda la organización; se cargan antes que las de usuario y proyecto y no se pueden excluir | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`<br>Linux/WSL: `/etc/claude-code/CLAUDE.md`<br>Windows: `C:\Program Files\ClaudeCode\CLAUDE.md` | **No aplica** — es un archivo que despliega IT/DevOps por MDM/Group Policy; no hay comando de Claude Code que lo cree | ❌ No — se distribuye por gestión de dispositivos (MDM/GPO), no por el repo |
| **User** (usuario) | Tus preferencias personales para **todos** tus proyectos | `~/.claude/CLAUDE.md` | **No aplica** — se edita como archivo a mano; `/memory` solo lo abre, no lo crea | ❌ No — vive en tu home, fuera de cualquier repo |
| **Project** (proyecto) | Instrucciones del proyecto compartidas con el equipo | `./CLAUDE.md` o `./.claude/CLAUDE.md` | `/init` genera un CLAUDE.md de proyecto inicial (o lo mejora si ya existe) | ✅ Sí — se versiona y se comparte por control de código |
| **Local** | Preferencias personales solo para este proyecto | `./CLAUDE.local.md` (agrégalo a `.gitignore`) | **No aplica** — archivo manual; al elegir la opción personal de `/init` se crea y se agrega a `.gitignore` | ❌ No — está pensado para `.gitignore`, no se comparte |

> Los `CLAUDE.md` ubicados en subdirectorios por debajo de tu directorio de trabajo **no se cargan al inicio**: se incluyen bajo demanda cuando Claude lee archivos de ese subdirectorio.

## Scope de Skill vs MCP vs .claude/rules/ vs CLAUDE.md

El scope de **CLAUDE.md** ya quedó cubierto en la tabla anterior. Aquí van las otras tres mecánicas — **Skill**, **MCP** y **.claude/rules/** — cada una con su propia tabla.

### Skill

| Scope | Explicación del alcance | Ruta donde se configura | Comando para configurar | ¿Se puede hacer git commit para compartir con el equipo? |
|---|---|---|---|---|
| **Personal** | Disponible en **todos** tus proyectos, solo para ti | `~/.claude/skills/<nombre-skill>/SKILL.md` | **No aplica** — no existe un comando de Claude Code para crear skills; se crea el directorio y el `SKILL.md` manualmente | ❌ No — vive en tu home, fuera del repo |
| **Project** | Solo en este proyecto; se comparte con el equipo | `<raíz-proyecto>/.claude/skills/<nombre-skill>/SKILL.md` | **No aplica** — se crea el directorio y el `SKILL.md` a mano y luego se versiona | ✅ Sí — se commitea `.claude/skills/` |
| **Plugin** | Disponible donde el plugin esté habilitado | `<plugin>/skills/<nombre-skill>/SKILL.md` | `/plugin install <nombre>@<marketplace>` | ❌ No por el repo del proyecto — se distribuye vía marketplace/plugin, no commiteando en tu repo |

### MCP

| Scope | Explicación del alcance | Ruta donde se configura | Comando para configurar | ¿Se puede hacer git commit para compartir con el equipo? |
|---|---|---|---|---|
| **Local** (por defecto) | Solo en el proyecto actual y privado para ti | `~/.claude.json` (bajo la ruta de ese proyecto) | `claude mcp add --transport http <nombre> <url>` (local es el scope por defecto; equivale a `--scope local`) | ❌ No — se guarda en `~/.claude.json`, fuera del repo |
| **Project** | Solo en el proyecto actual, compartido con el equipo | `.mcp.json` en la raíz del proyecto | `claude mcp add --transport http --scope project <nombre> <url>` | ✅ Sí — `.mcp.json` está pensado para versionarse |
| **User** | En **todos** tus proyectos, privado para ti | `~/.claude.json` | `claude mcp add --transport http --scope user <nombre> <url>` | ❌ No — se guarda en tu `~/.claude.json` |

> Aviso de nombres: en versiones antiguas el scope `local` se llamaba `project` y el `user` se llamaba `global`. Además, administradores pueden desplegar MCP a nivel de organización mediante configuración gestionada (enterprise).

### .claude/rules/

Las rules tienen **dos ejes independientes**: *dónde viven* (proyecto vs. usuario) y *cómo se cargan* (CON o SIN `paths:`):

- **SIN `paths:`** → se cargan en cada sesión, con la misma prioridad que `.claude/CLAUDE.md`; aplican a todo el proyecto.
- **CON `paths:`** → solo se cargan cuando Claude lee o edita archivos que coinciden con el glob de `paths:` (ahorra contexto).

| Scope | Explicación del alcance | Ruta donde se configura | Comando para configurar | ¿Se puede hacer git commit para compartir con el equipo? |
|---|---|---|---|---|
| **Proyecto — SIN `paths:`** | Se carga en cada sesión, con la misma prioridad que `.claude/CLAUDE.md`; aplica a todo el proyecto | `<raíz-proyecto>/.claude/rules/*.md` | **No aplica** — son archivos `.md` que creas a mano; no hay comando de Claude Code que los genere | ✅ Sí — se commitea `.claude/rules/` |
| **Proyecto — CON `paths:`** | Solo se carga cuando Claude lee/edita archivos que coinciden con el glob de `paths:`; reduce el contexto consumido | `<raíz-proyecto>/.claude/rules/*.md` con frontmatter `paths:` | **No aplica** — archivo `.md` con frontmatter `paths:`, creado a mano | ✅ Sí — se versiona igual que cualquier rule de proyecto |
| **Usuario** | Aplica a **todos** tus proyectos de la máquina; se carga antes que las rules de proyecto (estas tienen prioridad). También admite `paths:` | `~/.claude/rules/*.md` | **No aplica** — archivos `.md` creados a mano | ❌ No — vive en tu home, fuera del repo |

# Comparativa General (**NO** Específica de Context7) de Skill vs MCP vs .claude/rules/ vs CLAUDE.md

## Tabla comparativa

| Aspecto | Skill | MCP | .claude/rules/ | CLAUDE.md |
|---|---|---|---|---|
| **¿Cuándo se dispara (ejecuta)?** | Cuando Claude detecta match entre la tarea y la `description` del frontmatter, o se invoca con `/nombre-skill` | Cuando Claude decide llamar una tool del servidor para resolver parte de la tarea | Al inicio de sesión, automáticamente, si el archivo no tiene scoping de ruta; si tiene `paths:`, se activa al leer/editar archivos que coincidan con esa ruta (no en cada tool use) | Al inicio de sesión: el CLAUDE.md de la raíz y de los directorios superiores se carga completo siempre; los CLAUDE.md de subdirectorios se cargan bajo demanda cuando Claude lee archivos de ese subdirectorio |
| **¿Contra qué hace match para activarse?** | El texto de la `description` del frontmatter vs. el contenido del prompt/tarea actual | No hay "match" semántico — Claude elige la tool por su nombre/descripción cuando la necesita para una acción concreta | La ruta (`paths:`) de los archivos con los que se está trabajando, si está scopeado; si no, no hace match con nada, se carga igual | No hace match — se carga siempre, sin condición |
| **¿En qué momento está disponible?** | Disponible desde el inicio (su metadata se carga), pero el cuerpo de instrucciones solo se lee cuando se dispara | Disponible desde que el servidor conecta; con Tool Search (default), solo nombres/instrucciones del servidor están "disponibles" de entrada, la tool completa se carga al necesitarse | Disponible (cargado) desde el inicio de sesión si aplica a la ruta actual | Disponible (cargado) desde el inicio de sesión, durante toda la sesión |
| **¿Usa tools como `query-docs`?** | No directamente — puede invocar scripts propios vía bash, pero no expone "tools" tipo MCP | Sí — esa es exactamente la naturaleza de MCP (tools como `resolve-library-id`, `query-docs`, etc.) | No | No |
| **¿Puede ejecutar comandos?** | **Sí** (puede incluir scripts en `scripts/` y ejecutarlos vía bash) | **Sí** (las tools del servidor ejecutan acciones reales: queries, llamadas a API, etc., aunque no "comandos bash" sino funciones del servidor) | **No** (es texto de instrucciones, no ejecuta nada) | **No** (es texto de instrucciones, no ejecuta nada) |
| **Ejemplo de prompt para activarlo explícitamente** | `"Usa el skill codebase-visualizer para generar el árbol del proyecto"` o directo `/codebase-visualizer` | `"Usa la tool query-docs del MCP de Context7 para traer la doc de Next.js"` | *(No aplica — ver nota abajo)* | *(No aplica — ver nota abajo, pero distinto motivo)* |
| **¿Consume menos tokens?** | **Sí, el más eficiente de los 4** — solo la metadata (nombre + description) se carga al inicio; el cuerpo y los archivos de soporte (`reference.md`, `examples.md`, `scripts/`) se cargan on-demand y solo lo que se usa | Con Tool Search activo (default), bajo impacto: al inicio solo se cargan los nombres de las tools y las instrucciones del servidor. El costo sube solo si Tool Search se desactiva con `ENABLE_TOOL_SEARCH=false`, o en entornos donde no está disponible por defecto (Vertex AI, un `ANTHROPIC_BASE_URL` no first-party o modelos Haiku, que no soportan `tool_reference`), donde se cargan de entrada las definiciones completas de todas las tools | Costo fijo si no está scopeado por ruta (se carga siempre, igual que CLAUDE.md); si está scopeado, solo carga cuando aplica esa ruta | El más costoso de mantener "liviano" — se carga completo y permanece en contexto toda la sesión, sin excepciones ni progressive disclosure |

## Nota sobre `.claude/rules/` y CLAUDE.md: no tienen "prompt de activación"

A diferencia de Skill y MCP, **no existe un prompt para invocar manualmente `.claude/rules/` o CLAUDE.md**, porque no funcionan por invocación sino por carga automática de contexto:

- **CLAUDE.md**: el de la raíz y el de los directorios superiores al de trabajo se cargan completos al iniciar sesión, sin condición ni "match". Los CLAUDE.md ubicados en subdirectorios son la única excepción: se cargan bajo demanda cuando Claude lee archivos de ese subdirectorio.
- **.claude/rules/**: si el archivo no tiene scoping de ruta (`paths:`), se comporta igual que CLAUDE.md (siempre cargado). Si tiene scoping, se activa automáticamente cuando Claude lee o edita un archivo dentro de esa ruta — el "match" es la ruta del archivo en el que estás trabajando, no algo que el usuario solicite con una frase.

Por eso no hay un equivalente a `"usa el skill X"` o `"usa el MCP Y"` para estos dos: no se "llaman", simplemente están o no están en el contexto según dónde estés parado en el proyecto.

## Resumen en una línea por mecanismo

- **Skill** → procedimiento activado por relevancia semántica, el más eficiente en tokens por su carga progresiva.
- **MCP** → conexión a sistemas externos vía tools reales; con Tool Search, el costo de tenerlos conectados bajó mucho respecto a versiones anteriores de Claude Code.
- **.claude/rules/** → fragmento de contexto de proyecto, automático por ruta si está scopeado, o siempre cargado si no lo está.
- **CLAUDE.md** → contexto de proyecto sin scoping por ruta; el de la raíz se carga siempre y completo (los de subdirectorios cargan bajo demanda); el de mayor costo fijo de los cuatro.

# Comparativa Enfocada en Context7 Sobre Skill vs MCP en Claude Code

## Tabla Comparativa General

| Aspecto | Skill | MCP |
|---|---|---|
| ¿Puede conectarse a documentación? | Sí | Sí |
| Funcionalidad final | Misma documentación actualizada | Misma documentación actualizada |
| Cuándo se carga | La `description` se carga al inicio (para evaluar relevancia); el cuerpo solo cuando hay match o se invoca | Con Tool Search (activado por defecto), al inicio solo se cargan los nombres de las tools y las instrucciones del servidor; la definición completa se difiere hasta que Claude la necesita (no se declara en cada turno) |
| Disponibilidad | Disponible, pero pasivo hasta que se dispara | Disponible desde que el servidor conecta; con Tool Search solo los nombres están visibles de entrada y la definición completa se trae bajo demanda |
| Costo fijo por turno (sin usarlo) | Casi nulo — en contexto solo está el listado del skill (`description` + `when_to_use`, truncado a ~1.536 caracteres); el cuerpo no se carga hasta dispararse | Bajo con Tool Search: solo nombres de tools + instrucciones del servidor ocupan contexto, no las definiciones completas en cada turno. Alto solo si se desactiva Tool Search (`ENABLE_TOOL_SEARCH=false`) |
| Costo al ejecutar la búsqueda real | Tokens del output del comando (similar volumen) | Tokens del tool_result (similar volumen) |
| Mecanismo de ejecución | Comando CLI vía bash (ej. `ctx7 docs <id> <query>`) | Llamada a tool del servidor (`resolve-library-id`, `query-docs`) |
| ¿Se puede forzar/invocar explícitamente? | Sí — con `/nombre-skill`, o añadiendo "use context7" al prompt (frase que Context7 configura como disparador) | Sí — invocando la tool del servidor o añadiendo "use context7" al prompt |
| Ahorro de tokens relativo | Ligeramente mayor; la ventaja era grande con muchas tools acumuladas, pero con Tool Search la diferencia se reduce | Con Tool Search el overhead ya casi no escala con la cantidad de MCP servers (solo se suman nombres + instrucciones); vuelve a ser relevante solo si se desactiva Tool Search |

## Preguntas y Respuestas

| Pregunta | Respuesta |
|---|---|
| ¿Ambos pueden conectarse a documentación? | Sí, ambos traen la misma documentación actualizada; solo cambia el mecanismo de acceso. |
| ¿El skill consume tokens en cada llamada y el MCP no? | No es así. Ambos consumen tokens similares **cuando se ejecuta la búsqueda real** (el contenido de la doc pesa igual). En cuanto al costo fijo: con Tool Search (activado por defecto) el MCP ya no declara las definiciones completas en cada turno — al inicio solo carga los nombres de las tools y las instrucciones del servidor, y trae la definición completa al usarla. El skill mantiene en contexto solo su `description`; el cuerpo se carga al dispararse. |
| ¿El skill está disponible pero se invoca solo por match del description? | Sí, correcto. Claude Code lo evalúa por relevancia y solo lo ejecuta si el contexto de la pregunta coincide. |
| ¿El MCP está siempre disponible y eso hace que importe que no se usen las tools? | Ya no como antes. Con Tool Search (por defecto), las definiciones completas no quedan declaradas en cada turno: al inicio solo se cargan los nombres de las tools y las instrucciones del servidor, y la definición completa se trae solo cuando Claude la necesita. El costo fijo por tools sin usar es bajo, salvo que se desactive Tool Search (`ENABLE_TOOL_SEARCH=false`) o se use en un entorno donde no esté disponible (Vertex AI, `ANTHROPIC_BASE_URL` no first-party, modelos Haiku). |
| Al final, ¿cuándo ambos se invocan consumen tokens parecidos? | Sí. La respuesta con la documentación pesa lo mismo sea por tool_result (MCP) o por output de bash (skill). |
| ¿El skill ahorra más tokens que el MCP? | Con Tool Search activado (por defecto) la diferencia es pequeña incluso con varios MCP servers, porque añadir servidores tiene impacto mínimo en el contexto (solo se suman nombres + instrucciones). La ventaja del skill se vuelve notoria sobre todo si se desactiva Tool Search, donde el MCP vuelve a cargar todas las definiciones de entrada. |
| ¿Se puede forzar/invocar explícitamente cada uno? | Sí, ambos. Con MCP se invoca la tool del servidor o se añade la frase "use context7" al prompt. Con Skill se fuerza con `/nombre-skill`, o también con la frase "use context7" que Context7 configura como disparador; aunque normalmente el skill se dispara solo por relevancia de su `description`. |

## Resumen

La funcionalidad es idéntica (ambos traen la misma documentación). Con **Tool Search activado por defecto**, la diferencia de costo fijo se redujo mucho: el MCP ya no declara las definiciones completas en cada turno — al inicio solo carga los nombres de las tools y las instrucciones del servidor, y trae la definición completa al usarla. El skill mantiene en contexto solo su `description` y carga el cuerpo al dispararse. La diferencia de tokens vuelve a ser grande solo si se desactiva Tool Search (`ENABLE_TOOL_SEARCH=false`) o se usa en un entorno donde no esté disponible.
