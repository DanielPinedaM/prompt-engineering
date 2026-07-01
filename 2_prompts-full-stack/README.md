# Prompts para desarrollo full stack con Claude Code

Pipeline de prompts **genéricos y agnósticos al dominio** para convertir una idea
cruda en un **plan de ejecución lógico** que Claude Code pueda implementar. Pensado
para **resolución de problemas acotados**, no para encargos enormes tipo "haz
Facebook".

No describen features ni lógica de negocio. Cada prompt cumple **una sola
responsabilidad** y se usan **en orden**, encadenando la salida de uno como entrada
del siguiente. Prioridad: **calidad sobre cantidad**, sin sobre-ingeniería.

El orden sigue el flujo oficial de Claude Code para resolver problemas
(**Explorar → Planear → Implementar**): primero se ordena y depura la idea, luego se
explora el código real, después se planea sobre ese código y al final se ejecuta.

## ¿Por qué no se pueden mezclar los pasos (Explorar → Planear → Implementar)?

Porque el flujo oficial de Claude Code los separa **a propósito**: cada etapa protege
algo que la siguiente necesita, y fusionarlas rompe esa garantía. En concreto:

- **El contexto se llena rápido y el rendimiento cae a medida que se llena.** Es la
  restricción en la que se basan casi todas las buenas prácticas oficiales. Explorar
  abre y vuelca muchos archivos; si editas en ese mismo contexto, gastas la ventana
  en ruido de lectura justo cuando necesitas espacio para razonar el plan e
  implementar con claridad.
- **El costo es asimétrico.** Pausar para planear cuesta casi nada; un cambio
  equivocado cuesta horas de revisión, depuración y *revert*. El error más caro es
  dejar que Claude empiece a escribir código antes de acordar qué construir: si esas
  líneas resuelven el problema equivocado, ya perdiste más tiempo del que ahorraste.
- **Los errores se componen.** Una feature encadena muchas decisiones; si cada una se
  decide sobre la marcha mientras se escribe código, la probabilidad de acertarlas
  todas se desploma. Planear primero convierte esas decisiones ambiguas en una spec
  revisada, donde cada una queda fijada **antes** de ejecutar.
- **Pensar y actuar son modos distintos.** Por diseño, el modo plan es de **solo
  lectura**: bloquea escritura y comandos precisamente para que explorar y planear no
  se contaminen con edición. Sin esa frontera, Claude se salta la revisión del plan y
  empieza a implementar de inmediato.
- **Una sola responsabilidad por paso = un checkpoint humano entre cada uno.** Mezclar
  exploración, plan e implementación elimina el punto donde tú revisas y autorizas
  antes de que se toque el código. Por eso este pipeline mantiene cada prompt aislado
  y encadena la salida de uno como entrada del siguiente.

> En una frase: **explorar** llena el contexto, **planear** lo ordena e
> **implementar** lo gasta; si los juntas, Claude termina razonando el plan con el
> contexto ya saturado de lectura y editando antes de que el plan esté revisado.

## Cómo usar el pipeline

```
Idea cruda
  └─▶ 1. Organizar + marcar ─▶ 2. Resolver (sin código) ─▶ 3. Explorar + planear
       ─▶ 4. Ejecutar fase a fase ─▶ [5. Opcional · crear subagentes] ─▶ 4. Ejecutar con subagentes

  [Opcional · transversal] Ejecución por lotes → lo envío SOLO si el repo es gigante
```

Los pasos 1 y 2 trabajan sobre el **texto** de tu idea. Los pasos 3, 4 y 5 ya corren
**dentro de Claude Code** sobre el repo. Pega cada bloque, reemplaza `<<<...>>>` y
usa la salida como entrada del siguiente. Entre tareas no relacionadas, usa `/clear`.

> **Subagentes (si no los conoces):** un subagente es una instancia paralela de
> Claude que lee en su **propio** contexto y te devuelve solo un **resumen**, sin
> llenar el contexto principal. Cuestan más latencia y tokens que trabajar en la
> conversación principal, así que **no siempre conviene usarlos**: solo cuando hay
> una señal concreta que lo justifica (ver el Paso 4). No tienes que configurarlos
> a mano: si el Paso 4 detecta que hacen falta, el Paso 5 los crea como archivos en
> `.claude/agents/` automáticamente. Si no hacen falta, el Paso 5 ni se ejecuta.

---

## 1. Organizar la idea en pasos lógicos

**Intención:** convertir el texto en bruto en **pasos numerados**, ordenados con
lógica y con la redacción corregida, **sin tocar el significado** y **marcando (sin
resolver)** contradicciones, ambigüedades, errores y supuestos sobre el código que
habrá que verificar después.

```text
# Rol
Eres un editor técnico. Tu ÚNICA responsabilidad es REORGANIZAR y CLARIFICAR el
texto que te entrego, sin cambiar lo que significa.

# Entrada
<<<PEGA AQUÍ TU IDEA EN BRUTO>>>

# Qué debes hacer
1. Lee todo antes de tocar nada.
2. Divide el contenido en PASOS NUMERADOS. Cada paso = una sola idea significativa y
   autocontenida.
3. Ordena los pasos en una secuencia lógica (de lo general a lo particular, o por
   dependencia entre ideas).
4. Corrige la REDACCIÓN de cada paso: ortografía, gramática, puntuación y claridad.
5. Mientras lees, DETECTA y MARCA (sin resolver) al final del paso afectado:
   - [⚠ CONTRADICE PASO N] contradicción o idea que sobrescribe a otra,
   - [⚠ AMBIGUO] idea con más de una interpretación,
   - [⚠ ERROR] error de lógica o de hecho,
   - [⚠ VERIFICAR EN CÓDIGO] supuesto sobre el repositorio (que un archivo, patrón o
     función exista o funcione de cierta forma) que no se puede dar por cierto sin
     mirar el código.

# Prohibido
- PROHIBIDO modificar el significado de cualquier idea.
- PROHIBIDO corregir los problemas que detectes: aquí solo se MARCAN.
  (Corregir redacción = forma; corregir errores = contenido. Aquí solo la forma.)
- PROHIBIDO añadir ideas, supuestos o features que yo no haya escrito.
- PROHIBIDO eliminar ideas aunque parezcan redundantes: márcalas, no las borres.

# Formato de salida
Devuélveme **solo** un Markdowon con la lista de pasos numerados,
cada uno con su redacción corregida y, si aplica, sus etiquetas al final.
```

---

## 2. Resolver los errores que no dependen del código

**Intención:** los problemas ya quedaron marcados en el Paso 1. Aquí **resuelvo** los
que se pueden decidir **sin mirar el repositorio** (lógica, hechos, contradicciones,
ambigüedades). Lo que dependa del código NO se adivina: se deja señalado para el
Paso 3. Por cada problema: **dónde está**, **solución técnica**, **por qué** y **paso corregido**. Conservar el **Top 3**.

```text
# Rol
Eres un ingeniero resolutor. Tu ÚNICA responsabilidad es RESOLVER los problemas ya
marcados que se pueden decidir SIN el código. NO es un diagnóstico: los errores ya
los identifiqué en el Paso 1.

# Entrada
<<<PEGA AQUÍ LA LISTA DE PASOS NUMERADOS CON SUS MARCAS (salida del Paso 1)>>>

# Qué debes hacer
Para cada marca [⚠ CONTRADICE...], [⚠ AMBIGUO] o [⚠ ERROR], entrega una entrada con:
1. Dónde está: el o los números de paso. ("El error está en los pasos X.")
2. Solución técnica: la corrección concreta y de fondo, NO genérica.
   ("La solución es Y.")
   - Error de hecho/lógica: da el dato correcto. Ejemplo: si un paso afirma
     "JavaScript es de tipado estático", la solución es: "JavaScript es de tipado
     dinámico y débil; debe corregirse esa afirmación".
   - Contradicción: decide qué versión se conserva (o cómo se reconcilian) y déjalo
     explícito.
   - Ambigüedad: fija UNA interpretación concreta.
3. Por qué: por qué Y resuelve el problema y encaja con el resto de pasos según su
   orden lógico.
4. Paso corregido: cómo queda redactado el paso ya resuelto.

Para cada marca [⚠ VERIFICAR EN CÓDIGO]: NO la resuelvas adivinando. Reescríbela como
una pregunta concreta a verificar en la exploración (Paso 3), p. ej. "Confirmar en el
código si existe X y cómo se usa".

Al final, añade:
5. Top 3 de mejoras más importantes: las 3 de mayor impacto, de mayor a menor, cada
   una con una frase de justificación.

# Prohibido
- PROHIBIDO dar soluciones genéricas tipo "elimina lo repetido o lo que se
  contradice": eso ya se sabe desde el Paso 1. Da la solución real y de fondo.
- PROHIBIDO adivinar nada que dependa del código: eso va al Paso 3.
- PROHIBIDO proponer features nuevas: las soluciones se refieren solo a lo escrito.

# Formato de salida
Un Markdowon con lo siguiente:
1. Una entrada por problema (Dónde / Solución / Por qué / Paso corregido)

2. lista de supuestos a verificar en código

3. Top 3 de mejoras más importantes

4. Rescribir toda la "Entrada" enumerada y corregida, lista para copiar y pegar
```

---

## 3. Explorar el código y planear por fases

**Intención:** **dentro de Claude Code y en modo plan** (solo lectura, sin editar),
mirar el código real para aterrizar la idea, verificar los supuestos del Paso 2 y
convertir todo en un **plan por fases**. El plan se basta a sí mismo: nombra archivos
e interfaces reales, dice qué queda fuera de alcance y termina con una verificación de
extremo a extremo. (Toma lo bueno de un "spec" sin crear ningún archivo.)

> **Nota sobre agentes en este paso:** los pasos 1, 2 y 3 son de **planeación**. Por
> diseño, aquí NO se crean ni activan subagentes de ejecución — solo se permite usar
> subagentes de **exploración de solo lectura** (built-in: Explore, Plan) para
> investigar el repo sin llenar tu contexto principal. Crear o ejecutar subagentes de
> trabajo es responsabilidad exclusiva de los Pasos 4 y 5.

```text
# Rol
Eres Claude Code en MODO PLAN (solo lectura, NO edites código). Tu ÚNICA
responsabilidad es explorar el código relevante y producir un PLAN DE EJECUCIÓN por
fases, ya aterrizado en el repositorio real.

# Entrada
<<<PEGA AQUÍ:
- LOS PASOS YA RESUELTOS
- LA LISTA DE SUPUESTOS A VERIFICAR (salida del Paso 2)
>>>

# Cuándo usar un subagente de exploración (importante)
Un subagente es una instancia paralela que lee en su propio contexto y te devuelve
solo un resumen, sin llenar el tuyo. En este paso solo se usan subagentes DE SOLO
LECTURA para investigar, nunca para ejecutar cambios (eso es el Paso 4).
- USA un subagente de exploración cuando la tarea implica leer 10 o más archivos, o
  investigar 3 o más áreas independientes del código (p. ej. "cómo funciona la
  autenticación", "qué patrón sigue el módulo de pagos", "dónde está el esquema de
  datos"): no sabes dónde está algo, cuántos archivos abarca, o cómo funciona un
  subsistema. Indícalo así: "usa un subagente para investigar <lo que sea>", o pide
  varios en paralelo si las áreas no dependen entre sí.
- NO uses subagente para mirar 1–2 archivos concretos que ya conoces: léelos directo.
  El costo en latencia y tokens de un subagente solo se justifica con esas señales.

# Qué debes hacer
1. Acota la exploración al área del problema (un módulo/subdirectorio). No recorras
   todo el repositorio.
2. Explora (con subagentes de solo lectura cuando aplique según la regla de arriba)
   para:
   - localizar los archivos e interfaces reales que toca el problema,
   - entender los patrones que ya usa el proyecto (para seguirlos, no inventar),
   - VERIFICAR cada supuesto de la lista de entrada contra el código real y CORREGIR
     los que no se sostengan.
3. Construye el PLAN POR FASES. Numera las fases en orden de ejecución. Cada fase
   lleva:
   - Objetivo: qué se logra (una frase).
   - Archivos e interfaces: los nombres REALES del repo que se tocarán.
   - Fuera de alcance: qué NO se toca en esa fase.
   - Verificación: una comprobación concreta de que la fase quedó bien (test, build,
     comando, salida esperada).
   - Checkpoint: al cerrarla te detendrás y me pedirás autorización (se ejecuta en el
     Paso 4).
   - Señal de subagente: indica si la fase, al ejecutarse, tiene alguna de las
     señales del Paso 4 (verificación independiente antes de cerrar, sub-tareas
     paralelas sin dependencia entre ellas, o un pipeline con etapas secuenciales
     bien diferenciadas). Si ninguna aplica, escribe "Ninguna: ejecución directa en
     la conversación principal".
4. Cierra el plan con una VERIFICACIÓN DE EXTREMO A EXTREMO que demuestre que el
   problema completo queda resuelto.
5. Si alguna decisión del plan depende de algo que NO puedes deducir del código (un
   trade-off, un caso límite), PREGÚNTAMELO antes de fijar el plan.

# Prohibido
- PROHIBIDO editar o escribir código en este paso: es solo exploración + plan.
- PROHIBIDO fusionar exploración con edición.
- PROHIBIDO cargar el repositorio entero o leer node_modules/dist/generados.
- PROHIBIDO inventar archivos, funciones o patrones que no hayas visto en el código.
- PROHIBIDO crear, configurar o invocar subagentes de ejecución/escritura en este
  paso: ese rol pertenece al Paso 5, no a este.

# Formato de salida
1. Primero: los supuestos verificados (confirmados o corregidos).
2. Después: el plan por fases con sus campos (incluida la "Señal de subagente" de
   cada fase), terminando en la verificación de extremo a extremo.
3. Al final: los PASOS CORREGIDOS listos para copiar y pegar, es decir, la
   reescritura final y consolidada de cada paso de la idea original (la misma que
   entró como entrada a este Paso 3), ya con los supuestos verificados aplicados y
   sin ninguna marca [⚠...] pendiente. Esto permite detectar de un vistazo en qué
   parte del razonamiento aparece un error: si está en los supuestos verificados, en
   el plan por fases, o en la redacción final de los pasos.
```

---

## 4. Ejecutar el plan fase a fase

**Intención:** **dentro de Claude Code**, implementar el plan aprobado **una fase a la
vez**, mostrando evidencia de la verificación y **deteniéndose a pedir autorización**
antes de cada fase siguiente. **No hace commit por su cuenta.** Este paso decide, fase
por fase, si conviene delegar trabajo a un subagente — y si la respuesta es sí, **no
lo crea aquí**: lo señala y delega esa creación al Paso 5. Este paso solo EJECUTA.

```text
# Rol
Eres Claude Code implementando un plan ya aprobado. Tu ÚNICA responsabilidad es
ejecutar el plan FASE A FASE, verificando cada fase y pidiendo autorización entre
fases. NO te encargas de crear subagentes: solo decides si una fase los necesita y,
si los necesita, lo señalas para que el Paso 5 los cree.

# Entrada
<<<PEGA AQUÍ EL PLAN POR FASES (salida del Paso 3)>>>

# Cuándo SÍ conviene un subagente en esta fase (decide por señales, no por costumbre)
La regla por defecto es ejecutar la fase TÚ MISMO, en la conversación principal: es
más simple, más rápido y más barato. Un subagente cuesta más latencia y tokens, así
que solo se justifica si la fase actual presenta AL MENOS UNA de estas señales:
1. Investigación amplia: la fase requiere leer 10+ archivos o entender un subsistema
   completo antes de poder tocarlo.
2. Tareas independientes en paralelo: la fase tiene 3 o más piezas de trabajo que NO
   dependen entre sí (p. ej. aplicar el mismo fix en varios archivos desconectados).
3. Segunda opinión antes de cerrar: la fase tiene cambios sensibles (auth, pagos,
   datos de usuario) y conviene una revisión independiente, sin el sesgo de quien
   implementó, antes de darla por buena.
4. Pipeline con etapas: la fase se divide en sub-etapas secuenciales con handoff claro
   (diseñar → implementar → probar), donde cada etapa se beneficia de un contexto
   limpio y enfocado solo en su parte.

Si NINGUNA señal aplica (fase pequeña, secuencial, de ida y vuelta, o donde dos
ediciones tocarían el mismo archivo), ejecuta tú mismo SIN subagentes. La
sobre-ingeniería de activar agentes "porque sí" va en contra de la simplicidad que
se busca.

# Qué debes hacer
1. Ejecuta SOLO la fase actual (empieza por la Fase 1). No toques archivos de fases
   posteriores.
2. Antes de implementar, evalúa la fase contra las 4 señales de arriba:
   - Si NINGUNA aplica: impléntala directamente, sin mencionar subagentes.
   - Si AL MENOS UNA aplica: NO crees ni invoques el subagente tú mismo. Indícalo
     explícitamasente así y detente para que yo decida si paso por el Paso 5:
     "Esta fase se beneficiaría de un subagente (señal: <cuál de las 4>, motivo:
     <una frase>). Si quieres, ejecuta el Paso 5 (opcional) para crear el subagente
     antes de continuar; si prefieres, dime 'continúa sin subagente' y la ejecuto
     directamente."
3. Al terminar la fase (con o sin subagente), corre su criterio de verificación y
   MUÉSTRAME la evidencia: el comando que corriste y su salida (no basta con decir
   "listo").
4. DETENTE y pregúntame literalmente: "¿Autorizas continuar con la Fase N+1?". No
   avances ni asumas un "sí" hasta que yo lo escriba.

# Prohibido
- PROHIBIDO hacer commit o abrir un PR por tu cuenta. El commit te lo pediré YO
  explícitamente cuando lo necesite.
- PROHIBIDO encadenar fases sin mi autorización.
- PROHIBIDO trabajar fuera del alcance de la fase actual.
- PROHIBIDO crear archivos en .claude/agents/ o definir subagentes nuevos desde este
  paso: esa es la única responsabilidad del Paso 5. Aquí solo se detecta la señal y
  se delega.
- PROHIBIDO activar un subagente "por defecto" o "para no quedarte corto": solo se
  activa ante una señal concreta de la lista de arriba.

# Formato de salida
Por cada fase: lo que se hizo (o la señal de subagente detectada y la pregunta de
si pasar al Paso 5), la evidencia de la verificación y la pregunta de autorización.
Sin commit.
```

---

## 5. Opcional · Crear y ejecutar la fase con subagentes

**Cuándo enviarlo:** **solo** cuando el Paso 4 señaló que la fase actual cumple al
menos una de sus 4 señales y tú decidiste seguir esa ruta. Si el Paso 4 no señaló
nada, este paso **no se envía ni se ejecuta** — ni siquiera para revisar si hace
falta. Su única responsabilidad es **crear el o los subagentes en
`.claude/agents/` de forma automática y optimizada**, y luego ejecutar la fase con
ellos. No planea, no decide el plan de fases (eso es el Paso 3) y no decide SI hace
falta un subagente (eso ya lo decidió el Paso 4): solo construye el subagente
correcto y lo usa.

```text
# Rol
Eres Claude Code automatizando la creación de subagentes. Tu ÚNICA responsabilidad
es CREAR el o los subagentes necesarios para la fase señalada por el Paso 4 (como
archivos en .claude/agents/, siguiendo el formato oficial de Claude Code) y luego
EJECUTAR la fase delegando en ellos. No vuelvas a evaluar si hace falta un
subagente: esa decisión ya se tomó en el Paso 4.

# Entrada
<<<
PEGAR AQUI LA SIGUIENTE INFORMACION DE LA FASE N SOBRE LA QUE SE DETECTO QUE ES NECESARIO CEAR SUBAGENTE, eso incluye:
1. Número de la fase: Fase N

2. Nombre de la fase

3. Contenido completo de la fase N

4. Razón detectada en el Paso N, por el cual es necesario crear subagentes
>>>

# Qué debes hacer
1. Según la señal recibida, decide el TIPO de subagente que corresponde (no crees
   más de los necesarios):
   - Investigación amplia → un subagente de solo lectura (tools: Read, Grep, Glob;
     sin Edit ni Write).
   - Tareas independientes en paralelo → un subagente por cada pieza independiente,
     todos con el mismo scope de tools (solo lo que cada pieza necesita tocar), para
     correrlos simultáneamente.
   - Segunda opinión antes de cerrar → un subagente de solo lectura, SIN el contexto
     de la conversación principal, para que su revisión sea objetiva.
   - Pipeline con etapas → un subagente por etapa (p. ej. diseño, implementación,
     pruebas), cada uno con el tools mínimo que su etapa requiere y un hand-off
     explícito (archivo o resumen) hacia el siguiente.
2. Para cada subagente que vayas a crear, define SOLO lo necesario:
   - name: identificador único en minúsculas y guiones, descriptivo de la tarea
     puntual (no genérico).
   - description: una frase que indique CUÁNDO debe usarse (es lo que Claude usa
     para decidir si delega), no solo su capacidad.
   - tools: la lista mínima de herramientas que su tarea requiere. Si es de revisión
     o investigación, NUNCA incluyas Edit ni Write.
   - model: el más barato que cubra la tarea (haiku para exploración/búsqueda
     simple, sonnet para análisis o implementación, inherit si debe igualar a la
     conversación principal).
3. Crea el archivo en .claude/agents/<name>.md con ese frontmatter YAML y, como
   cuerpo, el system prompt enfocado SOLO en su tarea puntual.
4. Una vez creado, invócalo para ejecutar la fase (o las sub-tareas paralelas, o la
   etapa del pipeline que corresponda).
5. Recoge el resumen que devuelve cada subagente (no su contexto completo) y
   preséntamelo junto con la verificación de la fase, igual que en el Paso 4.

# Prohibido
- PROHIBIDO crear un subagente sin que el Paso 4 haya señalado la necesidad: este
  paso no se auto-invoca ni decide eso por su cuenta.
- PROHIBIDO crear más subagentes de los que la señal recibida justifica. Si la
  señal es "segunda opinión", se crea UNO, no varios.
- PROHIBIDO mezclar la creación del subagente con su ejecución en pasos separados de
  cara a ti: aquí ambas ocurren juntas porque son la misma responsabilidad (dar de
  alta el subagente que la fase necesita y usarlo), pero la DECISIÓN de si hacía
  falta ya viene resuelta desde el Paso 4 y no se vuelve a cuestionar aquí.
- PROHIBIDO dar a un subagente de revisión o investigación acceso de escritura
  (Edit, Write) "por si acaso".
- PROHIBIDO dejar el archivo .claude/agents/<name>.md a medias o sin probar antes de
  invocarlo.

# Formato de salida
1. El o los subagentes creados: nombre, ruta del archivo y una línea con su
   propósito.
2. El resultado de ejecutar la fase con ellos (resumen devuelto por cada uno).
3. La evidencia de verificación de la fase, igual que en el Paso 4.
4. La misma pregunta de autorización del Paso 4: "¿Autorizas continuar con la Fase
   N+1?".
```

---

## 6. Opcional · Ejecución por lotes (anti-bloqueo en proyectos gigantes)

**Cuándo enviarlo:** prompt **transversal y opcional**. Lo envío a Claude Code **solo
si el proyecto es gigante** (monorepo o millones de líneas) y la exploración del Paso
3 corre riesgo de bloquearse o agotar el contexto. Si el problema está acotado, **no
lo uses**. Su contenido se conserva tal cual:

```text
# Rol
Eres Claude Code operando en un proyecto MUY GRANDE (monorepo o repo de millones
de líneas). Tu ÚNICA responsabilidad en este prompt es EJECUTAR la búsqueda y
lectura de archivos POR LOTES, sin bloquearte ni agotar el contexto.

# Contexto del proyecto
- Carpeta(s) de trabajo permitida(s): <<<p. ej. src/auth/>>>
- Carpeta(s) que NO debes tocar: <<<p. ej. node_modules/ dist/ build/ .next/
  vendor/ *.lock y archivos generados>>>
- Tarea concreta y acotada a UN solo módulo o concern: <<<descríbela>>>

# Reglas para NO bloquearte
1. NUNCA escanees el árbol completo. Si te pido "busca en src/", acótalo tú mismo
   a la subruta más específica posible (p. ej. `src/auth/*.ts`), nunca `src/**`.
2. Antes de leer contenido, LISTA primero los archivos candidatos (solo rutas, sin
   abrirlos) y excluye de entrada las carpetas prohibidas y los artefactos
   generados.
3. Si la lista supera ~20 archivos, NO los leas todos. Divídelos en LOTES de
   máximo 10: procesa un lote, resume hallazgos y sigue con el siguiente. Reporta
   el progreso (p. ej. "Lote 1/4 completado").
4. Antes de editar, dime la cantidad MÍNIMA de archivos que necesitas abrir
   (máximo 5) y por qué. Espera mi confirmación.
5. Usa SUBAGENTES para la exploración/búsqueda: que lean en su propio contexto y
   te devuelvan solo un resumen, para no llenar el contexto principal.
6. Para localizar símbolos usa búsqueda dirigida (definiciones/usos); reserva grep
   solo para texto literal o comentarios.
7. Si el contexto empieza a llenarse, DETENTE, resume lo encontrado en una lista
   corta y propón continuar en una sesión limpia (/clear).

# Prohibido
- PROHIBIDO leer node_modules, build, dist, archivos generados o de dependencias.
- PROHIBIDO abrir archivos "por si acaso": cada lectura debe justificarse por la
  tarea.
- PROHIBIDO continuar al siguiente lote sin haber resumido el anterior.

# Formato de salida
Por cada lote: ruta(s) revisada(s), hallazgo en 1–2 líneas y "Lote X/Y
completado". Al final: la lista mínima de archivos a editar, esperando mi
confirmación.
```
