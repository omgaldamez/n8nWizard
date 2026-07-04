# CONTEXT — Diseñador de Arquitecturas de Agentes IA
## Para Claude Code · PCIA Módulo 4 · CENTRO CDMX

---

## 1. QUÉ ES ESTE PROYECTO

`index.html` es una herramienta visual single-file (HTML + CSS + JS puro, sin dependencias externas) para que los participantes del programa **Profesionalización Creativa con IA (PCIA)** de CENTRO Educación Continua (CDMX) diseñen la arquitectura de su agente de IA antes de construirla en n8n.

**Problema que resuelve:** Los participantes entran a la sesión 10 (cierre del Módulo 4) necesitando externalizar y comunicar la arquitectura de su producto. La herramienta les guía mediante un wizard y les permite armar un diagrama visual que exportan como SVG, como texto Markdown para Miro, como PlantUML para el diagrama de secuencia UML, y como prompt listo para pegar en Claude y generar el JSON del flujo n8n.

**Contexto pedagógico:**
- Módulo 4: "Datos, Memoria y Conectividad" — 10 sesiones × 2 horas
- Cohorte: ~12 creativos y profesionales (diseñadores, comunicadores, publicistas)
- Entran a la herramienta con: n8n funcionando en Docker, Supabase/PostgreSQL, Qdrant, Langfuse, OpenRouter, Telegram Bot
- Esta es la sesión final del módulo; la herramienta debe funcionar de forma autónoma (no requiere instructor)

---

## 2. ARCHIVO PRINCIPAL

```
index.html    ← todo el proyecto: HTML + CSS + JS, ~2400+ líneas
```

Sin `package.json`, sin `node_modules`, sin build step. Se abre directamente en Chrome.

**Restricción dura:** el archivo debe permanecer como single-file. No extraer CSS ni JS a archivos separados.

**JSONs de referencia en el proyecto** (flujos n8n reales del programa):
- `S7 - POST Agenda.json` — flujo de agenda web: Webhook POST → Switch → consultar calendar / agendar cita / guardar mensaje
- `S7 - Webhook GET.json` — lectura de tabla Supabase via webhook GET
- `M4S5_RegistroHoras-Intencion.json` — registro de usuarios + clasificador de intención
- `M4S5-Programado.json` — re-engagement con Schedule Trigger + Loop

---

## 3. ARQUITECTURA INTERNA DEL HTML

### Estado global (`S`)

```js
const S = {
  nodes: {},      // id → {id, type, x, y, label?, swData?, promptData?}
  conns: {},      // id → {id, from, to, fields: [{n, t, ex}]}
  sel: null,      // 'n:id' | 'c:id' | null
  mode: 'sel',    // 'sel' | 'conn'
  cFrom: null,    // nodeId cuando conectMode está activo
  drag: null,     // {id, ox, oy, mx, my, moved}
  nc: 1, cc: 1,   // contadores de IDs de nodos y conexiones
  vx: 0, vy: 0, vs: 1,   // viewport: pan X/Y y zoom scale
  panState: null, // {sx, sy, ox, oy} para pan con botón medio
  wizStep: 0,
  wd: {           // wizard data
    channels: [], mem: false, perfil: false, rag: 'none',
    obs: false, sw: false, llm: 'gemini-2.5-flash',
    clasifica: false, intents: ['CONVERSACION','FAQ','ACCION'],
    registro: false, schedule: false, timezone: 'America/Mexico_City'
  },
  swNode: null, swIn: [], swOut: [],
  // Diagrama de secuencia interactivo:
  seqOrder: null,       // array ordenado de actorIds (null = orden topológico)
  seqHidden: new Set(), // actorIds ocultos
  showSeqFields: true,  // mostrar campos de contrato en mensajes
  seqSel: null,         // actorId seleccionado en el panel de info
};
```

**Variables globales adicionales de módulo:**
```js
let curTab = 'arch';           // tab activa: 'arch' | 'seq'
let _justDragged = false;      // guard para suprimir clic fantasma post-drag
let _tipTimer = null;          // timeout del tooltip (400ms delay)
let _mmCollapsed = false;      // estado del mini-mapa
let _mmView = null;            // {wr, s, ox, oy} — proyección del mini-mapa para clic
let _lodBucket = 2;            // nivel de detalle actual: 0 | 1 | 2
let _contractId = null;        // connId abierto en el editor de contrato
let _contractFields = [];      // copia editable de los campos del contrato activo
let _seqDrag = null;           // estado de drag en la tab de secuencia
```

### Funciones clave

| Función | Qué hace |
|---------|----------|
| `render()` | Redibuja todo: conns → nodes → status → minimap → seq si tab activa |
| `applyView()` | Aplica transform al `<g id="canvas-root">`, actualiza zoom-lbl, background-position del grid, dispara LOD check y `renderMinimap()` |
| `svgPt(e)` | Convierte coordenadas de pantalla a coordenadas mundo |
| `zoom(delta, cx, cy)` | Zoom centrado en punto de pantalla. Clamp 0.15–4x. Sensibilidad suave: `mag = min(abs(deltaY),100)/100`, factor = 1 + mag*0.08 |
| `connGeom(fn, tn, off)` | Calcula la bezier cúbica de una conexión. Devuelve `{d, p0, p1, p2, p3}` |
| `bezMid(p0, p1, p2, p3)` | Punto real al 50% de la bezier: `(P0 + 3P1 + 3P2 + P3) / 8` |
| `contentBounds(pad)` | Bounding box de todos los nodos con padding |
| `lodBucket()` | Nivel de detalle según `NH * S.vs`. 0=solo rect, 1=ícono, 2=texto completo |
| `escA(s)` | Escapa `&`, `"`, `<` para atributos HTML en plantillas de string |
| `buildFromWiz()` | Construye el canvas desde los datos del wizard (6 pasos) |
| `applyPat(idx)` | Carga una de las plantillas predefinidas en el canvas |
| `addConn(fromId, toId)` | Agrega conexión (previene auto-loop y duplicados) |
| `genMD()` | Genera el Markdown de arquitectura para exportar |
| `genPlantUML()` | Genera el bloque @startuml…@enduml del diagrama de secuencia |
| `genN8NPrompt(mode)` | Genera el prompt completo para Claude. `mode`: `'paid'` (preguntas aclaratorias) o `'free'` (genera directo). Incluye tabla de contratos, nombres de nodos, reglas de timezone, prompts definidos |
| `renderSeq()` | Dibuja el diagrama de secuencia interactivo. Usa `S.seqOrder`, `S.seqHidden`, `S.showSeqFields` |
| `topoSort()` | Sort topológico (Kahn) para el orden de actores en el diagrama de secuencia |
| `openContractEditor(connId)` | Abre el editor de contrato JSON de una conexión |
| `inheritFields()` | Hereda los campos de la conexión upstream del nodo origen al contrato activo |
| `suggestNodeOutput()` | Agrega campos típicos del nodo origen al contrato, transformando `$json.campo` → `$('Label').item.json.campo` |
| `openPromptEditor(nodeId)` | Abre el modal ov-prompt para nodos de tipo LLM/agente |
| `autoGenPrompt()` | Auto-genera system+user prompt desde descripción usando PROMPT_TPL |
| `showMemPanel(nodeId)` | Muestra el panel de SQL de nodos de memoria |
| `saveDiagram()` | Descarga el estado completo como JSON `{v:1, nodes, conns, wd, nc, cc}` |
| `loadDiagram(file)` | Lee un JSON guardado, valida, migra tipos desconocidos a `custom`, restaura estado |
| `onNodeDbl(e, id)` | Doble clic: nodos PE_TYPES abren editor de prompt; subworkflows abren contrato; resto abre `prompt()` para renombrar |
| `renderSeqInfo(id)` | Panel de info del actor seleccionado: label, tipo, color, prompts, contratos, botones de acción |
| `seqHideActor(id)` | Oculta actor del diagrama de secuencia (añade a `S.seqHidden`) |
| `seqShowAll()` | Muestra todos los actores (`S.seqHidden.clear()`) |
| `seqResetOrder()` | Restaura el orden topológico (`S.seqOrder = null`) |
| `toggleSeqFields()` | Alterna visibilidad de campos de contrato en mensajes del diagrama |
| `tipEnter(id, el)` | Registra el timer de tooltip (400ms) |
| `hideTip()` | Cancela el timer y oculta `#node-tip` |
| `toggleMinimap()` | Alterna colapso del mini-mapa |
| `renderMinimap()` | Redibuja el mini-mapa y el rectángulo de viewport |

### Capas SVG

```
<svg id="cv" width="100%" height="100%">
  <defs>arrowhead marker</defs>
  <g id="canvas-root">  ← recibe el transform de zoom/pan
    <g id="cl"></g>     ← conexiones (bezier + badges de contrato)
    <g id="nl"></g>     ← nodos
  </g>
</svg>
```

---

## 4. CATÁLOGO DE COMPONENTES (`C`)

Cada componente tiene: `l` (label), `cat` (categoría), `col` (color hex), `ic` (ícono emoji), `d` (descripción corta).

| Categoría | Tipos |
|-----------|-------|
| `entrada` | telegram, webhook_in, web_ui, schedule, trigger, gdrive_in, gcal_in, gmail_in |
| `proceso` | normalizador, cerebro, subworkflow (+ prop `sw:true`), orquestador, agente_esp, clasificador, if_node, set_contrato, code_node |
| `fuente` | memoria, perfil, rag_naive, rag_enhanced, rag_multi |
| `modelo` | llm, llm_chain |
| `salida` | router_out, resp_tel, resp_http, webhook_out, gdrive_out, gcal_out, gmail_out |
| `obs` | langfuse |
| `datos` | supabase, qdrant, registro_usu |
| `custom` | custom |

**Colores por categoría:**
- entrada: `#3B82F6` (azul) / Google: `#34A853` Drive, `#1A73E8` Calendar, `#EA4335` Gmail
- proceso: `#7C3AED` (violeta), `#EA580C` if_node, `#6B7280` set_contrato/code_node
- fuente/memoria: `#10B981` (verde) / RAG: `#F59E0B` ámbar
- modelo LLM: `#EC4899` (rosa)
- salida: `#6366F1` (índigo)
- observabilidad: `#EAB308` (amarillo)
- datos: `#14B8A6` (teal)

---

## 5. PLANTILLAS PREDEFINIDAS (10)

1. **Agente simple** — Telegram → Normalizador → Cerebro → LLM → Resp. Telegram
2. **Con memoria** — Agrega Memoria (Supabase) bidireccional al Cerebro
3. **RAG naive** — Agrega RAG Naive (Qdrant) al Cerebro
4. **RAG enhanced + memoria** — RAG con filtros + memoria + Langfuse
5. **RAG multimodal** — RAG con análisis de imágenes (Gemini Vision)
6. **Multi-canal + sub-workflow** — Telegram, Webhook, Web → Normalizador → Subworkflow → RAG/LLM → Router → Salidas
7. **Multi-agente con router** — Orquestador → 3 Agentes Especializados → LLM → Router
8. **Ingesta RAG** — Google Drive / Webhook → Code Node → Set → Qdrant → Resp. HTTP
9. **Agenda Web** — Webhook POST → if_node (Switch por tipo) → 3 ramas: (gcal_in → code_node CDMX → llm_chain → resp_http) / (gcal_out → gmail_out → resp_http) / (supabase INSERT → set_contrato → resp_http)
10. **Re-engagement** — schedule → registro_usu (SELECT inactivos) → code_node (mensaje por status) → resp_tel → supabase (UPDATE recordatorio=TRUE)

Cada plantilla define nodos con posiciones (x, y) absolutas centradas en `CX = 550`, y conexiones como pares de índices `[fromIdx, toIdx]`.

---

## 6. WIZARD — 6 PASOS + SELECTOR DE PLANTILLAS

- **Paso 0** — Selector de plantillas (10 tarjetas, clic carga directamente)
- **Paso 1** — Canal(es) de entrada (multi-select: Telegram, Webhook, Web, Schedule, Trigger)
- **Paso 2** — Clasificador de intención (toggle + etiquetas editables + tabla registro_usuarios)
- **Paso 3** — Fuentes de memoria (Memoria conversacional, Perfil de usuario)
- **Paso 4** — RAG (none / naive / enhanced / multimodal)
- **Paso 5** — Opciones adicionales (Langfuse, sub-workflow, re-engagement, zona horaria)
- **Paso 6** — LLM principal (17 opciones incluyendo Gemini, GPT-4o, Claude, Llama, DeepSeek)

`buildFromWiz()` construye el canvas en capas. Si `wd.clasifica` está activo, inserta un nodo `clasificador` entre el normalizador y el cerebro. Si `wd.registro` está activo, añade `registro_usu` en la misma capa.

---

## 7. TABS

| Tab | Descripción |
|-----|-------------|
| ⬡ Arquitectura | Canvas principal con nodos y conexiones |
| ↔ Secuencia | Diagrama de secuencia UML interactivo |

`setTab(t)` alterna entre `#cwrap` (arquitectura) y `#seq-wrap` (secuencia).

---

## 8. FEATURES

### Canvas y navegación

- **Drag & drop** de nodos (mousedown → document mousemove → mouseup). `drag.moved` previene clic fantasma post-drag.
- **Zoom:** `Ctrl/Cmd + rueda` o pinch de trackpad. Sensibilidad escalada: `mag = min(abs(deltaY),100)/100`, factor = 1+mag*0.08. Clamp 0.15–4x.
- **Pan:** rueda sin modificador desplaza el canvas. Botón medio también hace pan.
- **Botones de zoom** (+/−/⌂) en esquina inferior derecha.
- **Modo conectar** — clic en nodo origen → clic en destino → crea conexión.
- **Teclado:** `Delete`/`Backspace` elimina la selección activa. `Escape` cancela `cFrom`.

### Nodos

- Rectángulos 170×58px, borde redondeado, barra de acento izquierda coloreada por categoría.
- **LOD adaptativo:** `NH*S.vs < 10px` → solo rect. `< 20px` → rect + ícono centrado. `≥ 20px` → texto completo.
- **Indicadores en nodo** (puntos circulares):
  - **Punto rosa** (bottom-left): el nodo tiene prompt definido (`promptData.system` o `promptData.user`)
  - **Punto verde** (bottom-right): el nodo recibe datos de un nodo de tipo fuente/datos/modelo
- **Doble clic:**
  - Nodos PE_TYPES (`cerebro`, `llm`, `llm_chain`, `clasificador`, `agente_esp`, `orquestador`) → abre editor de prompts
  - Subworkflows → abre modal de contrato JSON
  - Resto → `prompt()` para renombrar
- **Tooltip:** hover 400ms → `<div id="node-tip">` con descripción del componente.

### Conexiones

- Bezier cúbicos via `connGeom()`.
- Punto medio del badge usa `bezMid()` (t=0.5 real, no promedio de extremos).
- Offset de 12px para conexiones bidireccionales.
- Línea punteada para conexiones de categoría `obs` (Langfuse).
- **Badge de contrato** en el punto medio: muestra nombres de campos o `+ contrato`. Clic abre `openContractEditor()`.

### Editor de contratos (conexiones)

`openContractEditor(connId)` abre el modal `#ov-contract` con:

**Presets** (11 bloques semánticos predefinidos en `CONTRACT_PRESETS`):
- `telegram_in` — campos del trigger de Telegram
- `normalizado` — campos normalizados (telegram_id, mensaje, canal, timestamp)
- `con_intent` — agrega campo `intent`
- `con_memoria` — agrega campo `historial`
- `con_rag` — agrega campo `contexto_rag`
- `respuesta_llm` — campos de salida del LLM
- `metadata_rag` — campos para ingesta RAG (fuente, titulo, fecha_ingesta, url)
- `perfil_usuario` — campos de perfil de usuario
- `langfuse` — campos para trazas de Langfuse
- `solicitud_agenda` — campos de formulario web (tipo, slot, nombre, correo, telefono, session_id)
- `slots_cdmx` — resultado del Code CDMX (slots array, resumen, mensaje)

**Acciones:**
- **↑ Heredar anterior** (`inheritFields`) — copia los campos de la conexión que llega al nodo origen. Propaga contratos a lo largo del flujo.
- **↓ Salida típica** (`suggestNodeOutput`) — sugiere campos de salida típicos del nodo origen según su tipo. Transforma `$json.campo` → `$('Label').item.json.campo` para referencia explícita.

`NODE_OUTPUT_HINTS` — mapeo de 30+ tipos de nodo a sus campos de salida típicos, incluyendo:
- `code_node`: chunks, texto, metadata, **slots, resumen** (para agenda)
- `gcal_in`: evento (summary), inicio (start.dateTime), descripcion
- `registro_usu`: usuario_existe, session_id
- `schedule`: timestamp, timezone
- etc.

### Editor de prompts (nodos LLM/agente)

`openPromptEditor(nodeId)` abre el modal `#ov-prompt` para nodos PE_TYPES con:
- **Etiqueta del nodo** — editable
- **Descripción** + botón "✦ Generar" — auto-genera system+user prompt usando `PROMPT_TPL` según el tipo de nodo
- **System Prompt** — textarea editable
- **Chips de campos dinámicos** — botones para insertar expresiones n8n de los contratos entrantes. Usan `data-expr` attribute para seguridad (evita XSS con expresiones con comillas).
- **User Prompt** — textarea editable con inserción de chips
- **Preview** — muestra el prompt final con variables sustituidas
- El prompt se guarda en `node.promptData = {system, user}`

`PROMPT_TPL` — templates por tipo de nodo que generan `{system, user}`:
- `cerebro` — sistema de asistente con campos dinámicos como variables del user prompt
- `clasificador` — clasificador de intención con categorías de `S.wd.intents`
- `agente_esp` — agente especializado con rol específico
- `orquestador` — distribuidor de tareas a agentes
- `llm` / `llm_chain` — cadena LLM simple

### Prompt n8n — Generador

`genN8NPrompt(mode)` — dos modos:
- **`'paid'`** (Claude de paga): hace preguntas aclaratorias antes de generar (contratos, subworkflows, tablas Supabase, nodos de configuración)
- **`'free'`** (Claude gratuito): genera el JSON directamente sin preguntas

Contenido generado:
- Stack técnico (n8n v2.1.4, Supabase, Qdrant, OpenRouter, timezone)
- Reglas obligatorias (condicionales según el diagrama):
  - `hasWebhook` → regla de Response Mode + header CORS en Respond to Webhook (no en trigger)
  - `hasCalendar` → regla de Luxon timezone: `DateTime.fromISO(slot, {zone: 'America/Mexico_City'}).plus({minutes: 30}).toISO()`
  - `hasSchedule` → regla de columna `recordatorio BOOLEAN` + reset en UPSERT
- **Tabla de nombres exactos de nodos** — para que las expresiones `$('Nombre').item.json.campo` coincidan
- **Sintaxis de expresiones n8n:**
  - Nodo inmediato: `{{ $json.campo }}`
  - Nodo específico: `{{ $('Nombre del Nodo').item.json.campo }}`
- **Prompts definidos por nodo** — si el nodo tiene `promptData`, se incluye el system+user prompt exacto
- **Tabla de contratos JSON** — si las conexiones tienen campos definidos
- **Patrones clave** — SELECT EXISTS, UPSERT, information_schema, re-engagement SQL, Code CDMX (si hay Calendar)

### Diagrama de secuencia (interactivo)

- **Sort topológico (Kahn)** para el orden base de actores.
- Actor virtual "Usuario" siempre primero.
- `S.seqOrder` persiste el orden personalizado (null = topológico).
- `S.seqHidden` excluye actores del diagrama.
- **Drag de actores** — reordena `S.seqOrder` en tiempo real.
- **Botón × (hover)** en cada actor — lo oculta (`seqHideActor`).
- **Panel de info** (`renderSeqInfo`) al hacer clic en un actor:
  - Label, tipo, descripción del componente
  - Preview del system prompt si tiene `promptData`
  - Campos de contratos entrantes y salientes con expresiones
  - Botón "🧠 Editar prompts" (solo para PE_TYPES)
  - Botón "✕ Ocultar" actor
- **Controles** (`#seq-ctrls`): ⊕ Mostrar todos, ↺ Resetear orden, ≡ Campos (toggle)
- **Campos en mensajes**: cuando `showSeqFields`, muestra nombres de campos bajo las flechas.
- 39 pares de tipos con etiquetas predefinidas en `MSG_MAP` (español).
- Flechas sólidas para llamadas, punteadas para retornos.
- Export SVG del diagrama de secuencia.

### Guardar / Cargar diagrama

**Formato JSON guardado:**
```json
{
  "v": 1,
  "nodes": { ... },   // incluye promptData si fue definido
  "conns": { ... },
  "wd": { ... },
  "nc": 12,
  "cc": 8
}
```

Validaciones: campo `"v"` requerido, tipos desconocidos → `custom`, conexiones huérfanas descartadas.

### Mini-mapa

`<div id="minimap">` — 160×100px, colores de categoría, rectángulo de viewport, clic centra. Botón `▾` colapsa. Se actualiza en `render()` y `applyView()`.

### Export

| Botón | Output |
|-------|--------|
| ⬇ Guardar | JSON del estado completo con campo `v:1` |
| ⬆ Cargar | Input file oculto. Lee JSON, valida, restaura estado |
| ↓ SVG | Descarga SVG recortado al bounding box de nodos |
| ↓ SVG Secuencia | Descarga SVG del diagrama de secuencia |
| ⬡ Texto / Miro | Modal con Markdown de arquitectura + bloque PlantUML |
| ⬡ Prompt n8n | Modal con 2 botones de modo (paga/gratuito) + prompt generado |
| 📋 SQL | Guía de meta-prompting SQL |

---

## 9. STACK TÉCNICO DEL PROGRAMA (M4)

```
n8n:        v2.1.4 (Docker, self-hosted, Windows)
BD:         Supabase / PostgreSQL
Vector DB:  Qdrant v1.14.1
LLM:        OpenRouter → lmChatOpenRouter (nodo n8n)
Modelo:     google/gemini-2.5-flash (principal)
Embeddings: gemini-embedding-001 (3072 dimensiones)
Canal:      Telegram Bot (principal)
Obs:        Langfuse (integración via REST API: Set → Code → HTTP Request con Basic Auth)
Timezone:   America/Mexico_City — UTC-6 permanente sin horario de verano (desde 2023)
```

### Patrones de n8n documentados en la herramienta

- **SELECT EXISTS** para verificar usuarios (siempre devuelve 1 fila boolean)
- **UPSERT registro_usuarios** (INSERT ... ON CONFLICT DO UPDATE). Campos: `session_id TEXT PRIMARY KEY`, `primer_contacto TIMESTAMPTZ`, `ultima_interaccion TIMESTAMPTZ`, `status TEXT DEFAULT 'nuevo'`, `recordatorio BOOLEAN DEFAULT FALSE`
- **TIMESTAMPTZ** — para todos los campos de tiempo. México = UTC-6 permanente. Comparaciones en UTC sin conversión (NOW() y stored son ambos UTC).
- **Clasificador-first** — agente (NO LLM Chain), temperature 0.1, maxTokens 20. Clasifica intención ANTES de verificar usuario.
- **No Switch node** — usar IF encadenados (Switch causa error de importación en v2.1.4)
- **Postgres Chat Memory** — subnodo automático, tabla s2_test, session_key = chat.id
- **Qdrant retrieve-as-tool** — topK: 5, metadata.* prefix, gemini-embedding-001
- **Splitter chunkSize 10000** — evita doble-splitting en ingesta RAG
- **Subworkflow trigger** — executeWorkflowTrigger + executeWorkflow + Set (contrato)
- **Langfuse REST API** — 3 nodos: Set → Code (base64) → HTTP Request (Basic Auth)
- **Re-engagement** — Schedule → query inactivos (WHERE ultima_interaccion < NOW() - INTERVAL '...' AND recordatorio = FALSE) → Loop (batch 1) → Code → Telegram → UPDATE recordatorio=TRUE
- **information_schema** — siempre antes de meta-prompting SQL (obtener esquema real para contexto)
- **Google Calendar — timezone Luxon**: `DateTime.fromISO($json.body.slot, {zone: 'America/Mexico_City'}).plus({minutes: 30}).toISO()` para campo `end`. Sin `{zone}`, Luxon interpreta ISO local como UTC → evento 6h adelantado.
- **Code CDMX (slots)** — trabajo determinista, NO usar LLM para cálculo de disponibilidad. UTC_OFFSET_HOURS = -6, slots :00/:30, 9am–5pm, `getDay() === 0 || 6` para excluir fines de semana, convertir CDMX→UTC para comparar con eventos, devolver ISO local sin Z.
- **Webhook con CORS** — header `Access-Control-Allow-Origin: *` va en el nodo "Respond to Webhook", NO en el Webhook trigger (responseMode=responseNode ignora headers del trigger).
- **Expresiones n8n**: nodo inmediato = `{{ $json.campo }}`; nodo específico = `{{ $('Nombre del Nodo').item.json.campo }}`. Nombres de nodo sensibles a espacios y mayúsculas.

---

## 10. CONVENCIONES DE CÓDIGO

- Funciones JS en camelCase. Estado global en `S`. Componentes en `C`. Patrones en `PAT`.
- SVG generado con `document.createElementNS('http://www.w3.org/2000/svg', tag)` (helper: `svgEl(tag)`).
- Render siempre redibuja todo desde estado (innerHTML = '' + reconstrucción). No hay diff.
- IDs de nodos: `n1`, `n2`, ... IDs de conexiones: `c1`, `c2`, ...
- CSS con variables en `:root` (`--bg`, `--sur`, `--sur2`, `--bdr`, `--txt`, `--muted`, `--acc`, `--accl`, `--entrada`, `--proceso`, `--mem`, `--rag`, `--modelo`, `--salida`, `--obs`, `--datos`).
- `escA(s)` — usar siempre para interpolar valores de usuario en atributos SVG/HTML. Escapa `&`, `"`, `<`.
- Chips con expresiones n8n (que contienen comillas simples) usan el patrón `data-expr` + `this.dataset.expr` en el onclick para evitar XSS:
  ```js
  `<button data-expr="${expr.replace(/"/g,'&quot;')}" onclick="insertAtCursor('pe-user',this.dataset.expr)">${f.n}</button>`
  ```

---

## 11. CONVENCIONES EDITORIALES DEL PROGRAMA

- Sin em-dashes.
- Sin lenguaje motivacional ("hoy vamos a", "¿Todos listos?", "desmitificar").
- Ejercicios = "ejercicios individuales", nunca "tarea".
- Textos en español neutro (México como referencia de timezone).
- Ejemplos siempre en contexto de productos digitales (chatbots, agentes, multi-agent flows).
- Los materiales para participantes nunca revelan numeración interna de sesiones.

---

## 12. ESTRUCTURA HTML

```
<div id="app">                       ← grid: 48px toolbar + (220px sidebar + flex canvas)
  <div id="tb">                      ← toolbar con tabs, botones, guardar/cargar, controles
  <div id="pal">                     ← paleta de componentes izquierda
  <div id="cwrap">                   ← canvas wrap (overflow:hidden — sin scroll)
    <svg id="cv" 100%×100%>          ← canvas principal
      <defs>arrowhead</defs>
      <g id="canvas-root">           ← recibe transform zoom/pan
        <g id="cl">  ← conexiones
        <g id="nl">  ← nodos
    <div id="zoom-ctrl">             ← controles +/−/⌂ (bottom-right, position:absolute)
    <div id="legend">                ← leyenda de colores (top-right, position:absolute)
    <div id="status">                ← barra de estado inferior
    <div id="mem-panel">             ← panel SQL de nodo de memoria (bottom-left)
    <div id="minimap">               ← mini-mapa (encima del status, bottom-left)
      <svg id="mm-svg" 160×100>
      <button id="mm-toggle">
  <div id="seq-wrap">                ← tab de secuencia UML interactiva
    <div id="seq-ctrls">             ← botones: Mostrar todos | Resetear orden | Campos
    <div id="seq-info">              ← panel de info del actor seleccionado
    <div id="seq-svg-wrap">          ← SVG scrollable del diagrama
</div>

<div id="node-tip">                  ← tooltip de hover (position:fixed, fuera de #app)

<!-- Modales -->
<div id="ov-wiz">     ← wizard (6 pasos + selector de plantillas)
<div id="ov-pric">    ← tabla de precios de LLMs
<div id="ov-exp">     ← export texto (Markdown + PlantUML)
<div id="ov-sw">      ← editor de contrato de subworkflow
<div id="ov-n8n">     ← prompt n8n (2 modos: paga/gratuito)
<div id="ov-contract"> ← editor de contrato de conexión (presets + heredar + salida típica)
<div id="ov-sql">     ← guía de meta-prompting SQL
<div id="ov-prompt">  ← editor de prompts de nodo (system + user + chips dinámicos)

<script>
  const C = { ...componentes... }           // catálogo de 35 tipos
  const PAT = [ ...plantillas... ]          // 10 plantillas predefinidas
  const LLMS = [ ...pricing... ]            // 17 modelos con precios
  const MSG_MAP = { ...etiquetas seq... }   // 39 pares de etiquetas de secuencia
  const CONTRACT_PRESETS = { ... }          // 11 presets de campos de contrato
  const NODE_OUTPUT_HINTS = { ... }         // campos típicos de salida por tipo de nodo
  const PROMPT_TPL = { ... }                // templates de prompt por tipo de nodo PE
  const S = { ...estado global... }         // estado mutable
</script>
```

---

## 13. CÓMO USAR ESTE CONTEXTO EN CLAUDE CODE

Al iniciar una sesión en Claude Code:

1. Este archivo es el contexto completo del proyecto.
2. El único archivo fuente editable es `index.html`.
3. Para probar cambios: abrir el archivo directamente en Chrome (no requiere servidor).
4. Workflow típico:
   - Leer este contexto antes de cualquier modificación.
   - Identificar la sección del código a modificar.
   - Hacer cambios quirúrgicos con ediciones dirigidas.
   - Los JSONs de referencia son ejemplos del flujo real para orientar contratos y patrones.
5. Siempre mantener el archivo como single-file.
6. Las IDs de los elementos HTML son semánticas — no renombrar sin actualizar JS también.

---

## 14. PENDIENTES / MEJORAS IDENTIFICADAS

### Funcionalidad faltante
- [ ] **Modo touch/tablet** — zoom/pan con pinch y dos dedos reales (trackpad ya funciona via `ctrlKey+wheel`)
- [ ] **Undo/redo** — historial de estados (snapshot de `S.nodes` + `S.conns` antes de cada mutación)
- [ ] **Snap to grid** — alinear nodos a cuadrícula de 10px al soltar
- [ ] **Autoarrange** — reordenar automáticamente por capas (usando el sort topológico que ya existe)
- [ ] **Búsqueda en paleta** — input de filtrado de componentes

### Mejoras de UX
- [ ] Color picker para nodos `custom`
- [ ] Selección múltiple con drag de rectángulo

---

## 15. HISTORIAL DE SESIONES DE DISEÑO

| Sesión | Qué se construyó |
|--------|-----------------|
| 1 | PRD de la app, decisión de formato SVG (no Mermaid), wizard inicial |
| 2 | HTML/CSS/JS base: canvas SVG, 7 plantillas, drag-drop, conexiones bezier, export SVG/Markdown, pricing LLM |
| 3 | Tab de Secuencia UML: sort topológico, lifelines, mensajes predefinidos por par de tipos, export PlantUML |
| 4 | Bug fix display:block en tab Secuencia, sincronía de render al cambiar de tab |
| 5 | Wizard expandido a 6 pasos: clasificador, registro_usuarios, timezone, re-engagement; 7 componentes nuevos; Google Workspace; badge de contrato JSON en conexiones; editor de contrato; panel de memoria SQL; guía meta-prompting SQL; botón Prompt n8n; zoom/pan |
| 6 | Fix sensibilidad de zoom (1.12 → escala suave max 8% por 100px deltaY) |
| 7 | Auditoría completa + fixes P0/P1 + 5 features: save/load JSON, rename por dblclick, contratos en prompt n8n, tooltip, mini-mapa. Fixes: stopPropagation en clic de conexiones, llm_chain en catálogo, clic fantasma post-drag, export SVG independiente de zoom/pan, clearAll y delSel limpian cFrom y mem-panel, midpoint bezier real t=0.5, LOD adaptativo, overflow:hidden + rueda=pan + Ctrl+rueda=zoom, escA() para inputs |
| 8 | Prompt n8n en dos modos (Claude paga vs gratuito) con preguntas aclaratorias o generación directa; regla de webhook con Response Mode; sistema de contratos completo: 9 presets semánticos, heredar upstream, salida típica (NODE_OUTPUT_HINTS con 30+ tipos); nueva plantilla Ingesta RAG; sintaxis $('Node').item.json.campo para referencias no-inmediatas; tabla de nombres de nodos en prompt generado |
| 9 | Editor de prompts para nodos LLM/agente (modal ov-prompt): descripción → auto-gen con PROMPT_TPL, system/user textarea, chips dinámicos de campos con data-expr para seguridad, preview en tiempo real, punto rosa indicador en nodo; prompts incluidos en el prompt n8n |
| 10 | Diagrama de secuencia interactivo: drag para reordenar actores (S.seqOrder), botón × hover para ocultar (S.seqHidden), controles (Mostrar todos / Resetear orden / toggle Campos), campos de contrato bajo flechas (S.showSeqFields), panel de info al clic (renderSeqInfo) con prompts + contratos + botón editar, handlers onSeqMM/onSeqMU |
| 11 | 2 nuevas plantillas: Agenda Web (Webhook → Switch 3 ramas: Calendar/Gmail/Supabase) y Re-engagement (Schedule → Supabase → Code → Telegram → Supabase); 2 nuevos presets de contrato: solicitud_agenda y slots_cdmx; CODE_node hints con slots/resumen; reglas condicionales en genN8NPrompt: calendarRule (Luxon timezone CDMX) y scheduleRule (recordatorio BOOLEAN); patrón Code CDMX en sección "Patrones clave" del prompt |

---

*Actualizado julio 2026. Archivo de referencia para sesiones de Claude Code y Claude.ai.*
