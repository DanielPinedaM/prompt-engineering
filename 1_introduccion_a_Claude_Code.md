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

## Scope de Claude md

## Scope de Skill vs MCP vs .claude/rules/ vs CLAUDE.md

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
