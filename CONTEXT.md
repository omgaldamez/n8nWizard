# CONTEXT — Diseñador de Arquitecturas de Agentes IA
## Para Claude Code · PCIA Módulo 4 · CENTRO CDMX

---

## 1. QUÉ ES ESTE PROYECTO 

`arquitectura-agentes7.html` es una herramienta visual single-file (HTML + CSS + JS puro, sin dependencias externas) para que los participantes del programa **Profesionalización Creativa con IA (PCIA)** de CENTRO Educación Continua (CDMX) diseñen la arquitectura de su agente de IA antes de construirla en n8n.

**Problema que resuelve:** Los participantes entran a la sesión 10 (cierre del Módulo 4) necesitando externalizar y comunicar la arquitectura de su producto. La herramienta les guía mediante un wizard y les permite armar un diagrama visual que exportan como SVG, como texto Markdown para Miro, como PlantUML para el diagrama de secuencia UML, y como prompt listo para pegar en Claude y generar el JSON del flujo n8n.

**Contexto pedagógico:**
- Módulo 4: "Datos, Memoria y Conectividad" — 10 sesiones × 2 horas
- Cohorte: ~12 creativos y profesionales (diseñadores, comunicadores, publicistas)
- Entran a la herramienta con: n8n funcionando en Docker, Supabase/PostgreSQL, Qdrant, Langfuse, OpenRouter, Telegram Bot
- Esta es la sesión final del módulo; la herramienta debe funcionar de forma autónoma (no requiere instructor)

---

## 2. ARCHIVO PRINCIPAL

```
arquitectura-agentes7.html    ← todo el proyecto: HTML + CSS + JS, ~2092 líneas
```

Sin `package.json`, sin `node_modules`, sin build step. Se abre directamente en Chrome.

**Restricción dura:** el archivo debe permanecer como single-file. No extraer CSS ni JS a archivos separados.

---

## 3. ARQUITECTURA INTERNA DEL HTML

### Estado global (`S`)

```js
const S = {
  nodes: {},      // id → {id, type, x, y, label?, swData?}
  conns: {},      // id → {id, from, to, fields: [{n, t, ex}]}
  sel: null,      // 'n:id' | 'c:id' | null
  mode: 'sel',    // 'sel' | 'conn'
  cFrom: null,    // nodeId cuando conectMode está activo
  drag: null,     // {id, ox, oy, mx, my, moved}  ← moved: boolean para distinguir drag de clic
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
```

### Funciones clave

| Función | Qué hace |
|---------|----------|
| `render()` | Redibuja todo: conns → nodes → status → minimap → seq si tab activa |
| `applyView()` | Aplica transform al `<g id="canvas-root">`, actualiza zoom-lbl, background-position del grid, dispara LOD check y `renderMinimap()` |
| `svgPt(e)` | Convierte coordenadas de pantalla a coordenadas mundo (sin scroll, usa solo clientX/Y) |
| `zoom(delta, cx, cy)` | Zoom centrado en punto de pantalla. Clamp 0.15–4x |
| `connGeom(fn, tn, off)` | Calcula la bezier cúbica de una conexión. Devuelve `{d, p0, p1, p2, p3}` |
| `bezMid(p0, p1, p2, p3)` | Punto real al 50% de la bezier: `(P0 + 3P1 + 3P2 + P3) / 8` |
| `contentBounds(pad)` | Bounding box de todos los nodos con padding. Devuelve `{x,y,w,h}` o `null` si no hay nodos |
| `lodBucket()` | Nivel de detalle según `NH * S.vs`. 0=solo rect, 1=ícono, 2=texto completo |
| `escA(s)` | Escapa `&`, `"`, `<` para atributos HTML en plantillas de string |
| `buildFromWiz()` | Construye el canvas desde los datos del wizard (6 pasos) |
| `applyPat(idx)` | Carga una de las 7 plantillas predefinidas en el canvas |
| `addConn(fromId, toId)` | Agrega conexión (previene auto-loop y duplicados) |
| `genMD()` | Genera el Markdown de arquitectura para exportar |
| `genPlantUML()` | Genera el bloque @startuml…@enduml del diagrama de secuencia |
| `genN8NPrompt()` | Genera el prompt completo para Claude. Incluye tabla de contratos JSON si las conexiones tienen campos definidos |
| `renderSeq()` | Dibuja el diagrama de secuencia UML en la tab Secuencia |
| `topoSort()` | Sort topológico (Kahn) para el orden de actores en el diagrama de secuencia |
| `openContractEditor(connId)` | Abre el editor de contrato JSON de una conexión |
| `showMemPanel(nodeId)` | Muestra el panel de SQL de nodos de memoria |
| `saveDiagram()` | Descarga el estado completo como JSON `{v:1, nodes, conns, wd, nc, cc}` |
| `loadDiagram(file)` | Lee un JSON guardado, valida campo `"v"`, migra tipos desconocidos a `custom`, restaura estado |
| `onNodeDbl(e, id)` | Doble clic: subworkflows abren el modal de contrato; el resto abre `prompt()` para renombrar |
| `tipEnter(id, el)` | Registra el timer de tooltip (400ms). Usa `getBoundingClientRect` del elemento SVG en pantalla |
| `hideTip()` | Cancela el timer y oculta `#node-tip` |
| `toggleMinimap()` | Alterna colapso del mini-mapa |
| `renderMinimap()` | Redibuja el mini-mapa y el rectángulo de viewport. No opera si `_mmCollapsed` |

### Capas SVG

```
<svg id="cv" width="100%" height="100%">   ← ocupa el 100% de #cwrap (overflow:hidden)
  <defs>arrowhead marker</defs>
  <g id="canvas-root">  ← recibe el transform de zoom/pan
    <g id="cl"></g>     ← conexiones (bezier + badges de contrato)
    <g id="nl"></g>     ← nodos
  </g>
</svg>
```

**Importante:** El SVG ya no tiene dimensiones fijas (3000×2000). `#cwrap` tiene `overflow:hidden`. La navegación es solo por zoom/pan — no hay scroll.

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
- entrada: `#3B82F6` (azul)
- proceso: `#7C3AED` (violeta)
- fuente/memoria: `#10B981` (verde)
- fuente/RAG: `#F59E0B` (ámbar)
- modelo LLM: `#EC4899` (rosa)
- salida: `#6366F1` (índigo)
- observabilidad: `#EAB308` (amarillo)
- datos: `#14B8A6` (teal)

---

## 5. PLANTILLAS PREDEFINIDAS (7)

1. **Agente simple** — Telegram → Normalizador → Cerebro → LLM → Resp. Telegram
2. **Con memoria** — Agrega Memoria (Supabase) bidireccional al Cerebro
3. **RAG naive** — Agrega RAG Naive (Qdrant) al Cerebro
4. **RAG enhanced + memoria** — RAG con filtros + memoria + Langfuse
5. **RAG multimodal** — RAG con análisis de imágenes (Gemini Vision)
6. **Multi-canal + sub-workflow** — Telegram, Webhook, Web → Normalizador → Subworkflow → RAG/LLM → Router → Salidas
7. **Multi-agente con router** — Orquestador → 3 Agentes Especializados → LLM → Router

Cada plantilla define nodos con posiciones (x, y) absolutas centradas en `CX = 550`, y conexiones como pares de índices `[fromIdx, toIdx]`.

---

## 6. WIZARD — 6 PASOS + SELECTOR DE PLANTILLAS

- **Paso 0** — Selector de plantillas (7 tarjetas, clic carga directamente)
- **Paso 1** — Canal(es) de entrada (multi-select: Telegram, Webhook, Web, Schedule, Trigger)
- **Paso 2** — Clasificador de intención (toggle + etiquetas editables + tabla registro_usuarios)
- **Paso 3** — Fuentes de memoria (Memoria conversacional, Perfil de usuario)
- **Paso 4** — RAG (none / naive / enhanced / multimodal)
- **Paso 5** — Opciones adicionales (Langfuse, sub-workflow, re-engagement, zona horaria)
- **Paso 6** — LLM principal (10 opciones incluyendo Gemini, GPT-4o, Claude, Llama, DeepSeek)

`buildFromWiz()` construye el canvas en capas. Si `wd.clasifica` está activo, inserta un nodo `clasificador` entre el normalizador y el cerebro. Si `wd.registro` está activo, añade `registro_usu` en la misma capa.

---

## 7. TABS

| Tab | Descripción |
|-----|-------------|
| ⬡ Arquitectura | Canvas principal con nodos y conexiones |
| ↔ Secuencia | Diagrama de secuencia UML auto-generado desde el canvas |

`setTab(t)` alterna entre `#cwrap` (arquitectura) y `#seq-wrap` (secuencia).

---

## 8. FEATURES

### Canvas y navegación

- **Drag & drop** de nodos (mousedown → document mousemove → mouseup). `drag.moved` previene que el mouseup emita un clic fantasma.
- **Zoom:** `Ctrl/Cmd + rueda` (o pinch de trackpad, que llega como `wheel` con `ctrlKey`). Centrado en cursor. Clamp 0.15–4x.
- **Pan:** rueda sin modificador desplaza el canvas. Botón medio del mouse también hace pan.
- **Botones de zoom** (+/−/⌂) en esquina inferior derecha.
- **Modo conectar** — clic en nodo origen → clic en destino → crea conexión.
- **Teclado:** `Delete`/`Backspace` elimina la selección activa. `Escape` cancela `cFrom` o el modo conectar.

### Nodos

- Rectángulos 170×58px, borde redondeado, barra de acento izquierda coloreada por categoría.
- **LOD adaptativo:** si `NH * S.vs < 20px`, se ocultan las etiquetas y se centra el ícono. Si `NH * S.vs < 10px`, solo se muestra el rectángulo coloreado.
- Selección visual (borde blanco grueso cuando `S.sel === 'n:id'`).
- Badge SW en subworkflows.
- **Doble clic:** en subworkflow abre el modal de contrato JSON. En cualquier otro nodo, abre `prompt()` para editar la etiqueta.
- **Tooltip:** al hacer hover 400ms, aparece un `<div id="node-tip">` en `position:fixed` con la descripción del componente. Se posiciona a la derecha del nodo si hay espacio, a la izquierda si no.

### Conexiones

- Bezier cúbicos via `connGeom()`. Devuelve `{d, p0, p1, p2, p3}`.
- El punto medio del badge usa `bezMid(p0,p1,p2,p3)` — el punto real al t=0.5 de la curva, no el promedio de los extremos.
- Offset de 12px para conexiones bidireccionales.
- Línea punteada para conexiones de categoría `obs` (Langfuse).
- **Clic en una conexión la selecciona** (stopPropagation evita que el evento burbujee hasta el listener de `#cv`). Una vez seleccionada, `delSel()` la elimina.
- **Badge de contrato** en el punto medio: si la conexión tiene campos definidos, muestra sus nombres; si no, muestra `+ contrato`. Clic abre `openContractEditor()`.

### Guardar / Cargar diagrama

Botones `⬇ Guardar` y `⬆ Cargar` en el toolbar.

**Formato JSON guardado:**
```json
{
  "v": 1,
  "nodes": { ... },
  "conns": { ... },
  "wd": { ... },
  "nc": 12,
  "cc": 8
}
```

**Validaciones al cargar:**
- Debe tener campo `"v"`.
- Tipos desconocidos se cargan como `custom` y el tipo original se preserva en `label`.
- Conexiones con nodos que no existen en el JSON se descartan silenciosamente.
- `nc` y `cc` se recalculan como `max(valor_guardado, max_id_encontrado + 1)` para evitar colisiones.

### Mini-mapa

`<div id="minimap">` en esquina inferior izquierda del canvas (sobre el status bar).
- 160×100px fijo.
- Los nodos se representan como rectángulos del color de su categoría.
- El rectángulo azul violeta muestra el área actualmente visible.
- Clic en el mini-mapa centra el viewport en esa posición del mundo.
- Botón `▾` colapsa/expande. Cuando colapsado, la clase `mm-collapsed` oculta el SVG.
- Se actualiza en `render()` y en `applyView()`.
- `_mmView` guarda la proyección `{wr, s, ox, oy}` que `renderMinimap()` calcula, para que el listener de clic la reutilice.

### Prompt n8n — Contratos JSON

`genN8NPrompt()` incluye una sección `### Contratos JSON entre nodos` si hay conexiones con al menos un campo definido:
- Tabla Markdown: Conexión | Campos que pasan
- Si los campos tienen `ex` (expresión de ejemplo), se lista en un bloque posterior.

### Diagrama de secuencia UML

- Sort topológico (Kahn) para ordenar actores por categoría.
- Siempre añade "Usuario" como primer actor virtual.
- 39 pares de tipos con etiquetas predefinidas (español).
- Flechas sólidas para llamadas, punteadas para retornos.
- Export SVG del diagrama de secuencia.
- Export PlantUML en el modal de texto. Las etiquetas de nodo escapan `"` a `'` para evitar PlantUML inválido.

### Export

| Botón | Output |
|-------|--------|
| ⬇ Guardar | JSON del estado completo con campo `v:1` |
| ⬆ Cargar | Input file oculto. Lee JSON, valida, restaura estado |
| ↓ SVG | Descarga SVG recortado al contenido (viewBox al bounding box de nodos), independiente del zoom/pan activo |
| ↓ SVG Secuencia | Descarga SVG del diagrama de secuencia |
| ⬡ Texto / Miro | Modal con Markdown de arquitectura + bloque PlantUML |
| ⬡ Prompt n8n | Prompt completo para pegar en Claude (incluye arquitectura + contratos) |
| 📋 SQL | Guía de meta-prompting SQL |

---

## 9. STACK TÉCNICO DEL PROGRAMA (M4)

Lo que los participantes tienen instalado y que la herramienta refleja:

```
n8n:        v2.1.4 (Docker, self-hosted, Windows)
BD:         Supabase / PostgreSQL
Vector DB:  Qdrant v1.14.1
LLM:        OpenRouter → lmChatOpenRouter (nodo n8n)
Modelo:     google/gemini-2.5-flash (principal), gemini-2.5-pro, GPT-4o, Claude, Llama
Embeddings: gemini-embedding-001 (3072 dimensiones)
Canal:      Telegram Bot (principal)
Obs:        Langfuse (integración via REST API: Set → Code → HTTP Request con Basic Auth)
Timezone:   America/Mexico_City — UTC-6 permanente sin horario de verano (México)
```

### Patrones de n8n documentados en la herramienta

- **SELECT EXISTS** para verificar usuarios (siempre devuelve 1 fila boolean)
- **UPSERT registro_usuarios** (INSERT ... ON CONFLICT DO UPDATE)
- **Clasificador-first** — el agente clasifica intención ANTES de verificar usuario (routing vs personalización)
- **No Switch node** — usar IF encadenados (Switch causa error de importación en v2.1.4)
- **Postgres Chat Memory** — subnodo automático, tabla s2_test, session_key = chat.id
- **Qdrant retrieve-as-tool** — topK: 5, metadata.* prefix, gemini-embedding-001
- **Splitter chunkSize 10000** — evita doble-splitting en ingesta RAG
- **Subworkflow trigger** — executeWorkflowTrigger + executeWorkflow + Set (contrato)
- **Langfuse REST API** — 3 nodos: Set → Code (base64) → HTTP Request (Basic Auth)
- **Re-engagement** — Schedule → query inactivos → Loop (batch 1) → Code → Telegram → UPDATE recordatorio
- **information_schema** — siempre antes de meta-prompting SQL
- **TIMESTAMPTZ** — para todos los campos de tiempo; México = UTC-6 permanente

---

## 10. CONVENCIONES DE CÓDIGO

- Funciones JS en camelCase. Estado global en `S`. Componentes en `C`. Patrones en `PAT`.
- SVG generado con `document.createElementNS('http://www.w3.org/2000/svg', tag)` (helper: `svgEl(tag)`).
- Render siempre redibuja todo desde estado (innerHTML = '' + reconstrucción). No hay diff.
- IDs de nodos: `n1`, `n2`, ... IDs de conexiones: `c1`, `c2`, ...
- CSS con variables en `:root` (`--bg`, `--sur`, `--sur2`, `--bdr`, `--txt`, `--muted`, `--acc`, `--accl`, `--entrada`, `--proceso`, `--mem`, `--rag`, `--modelo`, `--salida`, `--obs`, `--datos`).
- `escA(s)` — usar siempre para interpolar valores de usuario en atributos `value="..."` de plantillas de string. Evita corrupción del markup si un nombre de campo contiene comillas.
- Inputs en modales (contratos, subworkflow) **deben** usar `escA()` en `value="${escA(f.n)}"`.

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
  <div id="seq-wrap">                ← tab de secuencia UML
    <svg id="seq-cv">
</div>

<div id="node-tip">                  ← tooltip de hover (position:fixed, fuera de #app)
  <div class="tt-l">                 ← label del componente
  <div class="tt-d">                 ← descripción del componente

<!-- Modales: ov-wiz, ov-pric, ov-exp, ov-sw, ov-n8n, ov-contract, ov-sql -->

<script>
  const C = { ...componentes... }         // catálogo
  const PAT = [ ...plantillas... ]        // 7 plantillas
  const LLMS = [ ...pricing... ]          // tabla de pricing
  const MSG_MAP = { ...etiquetas seq... } // 39 pares de etiquetas de secuencia
  const S = { ...estado global... }       // estado mutable

  // helpers: escA, svgEl, lodBucket, svgPt, applyView, zoom, resetView, contentBounds
  // render: render, renderNodes, renderConns, renderStatus, renderMinimap, renderSeq
  // viewport: applyView, zoom, resetView
  // events: onNodeMD, onMM, onMU, onNodeClick, onNodeDbl, onConnClick, tipEnter, tipLeave, hideTip
  // operations: addNode, addConn, delSel, clearAll, toggleConn
  // connections: connGeom, bezMid
  // minimap: toggleMinimap, renderMinimap
  // palette: renderPal, addFromPal
  // wizard: showWiz, renderWiz, buildFromWiz, toggleCh, toggleWB, toggleIntent, addCustomIntent
  // export svg: doExpSVG, doExpSeqSVG
  // export text: genMD, showExpMD, copyExp, genPlantUML
  // n8n: showN8N, genN8NPrompt, copyN8N
  // save/load: saveDiagram, loadDiagram
  // contract: openContractEditor, renderContractFields, renderContractPreview, addContractField, saveContract
  // memory: showMemPanel, hideMemPanel, MEM_TABLES
  // subworkflow: openSW, renderSW, renderSWPre, saveSW, addSWF
  // sequence: renderSeq, topoSort, genSeqMsgs, doExpSeqSVG
  // pricing: showPric
  // tabs: setTab
  // keyboard: document keydown listener
  // zoom/pan: initZoomPan, onPanMove, onPanEnd
  // init: renderPal, renderWiz, show('ov-wiz'), initZoomPan, applyView
</script>
```

---

## 13. CÓMO USAR ESTE CONTEXTO EN CLAUDE CODE

Al iniciar una sesión en Claude Code:

1. Este archivo es el contexto completo del proyecto.
2. El único archivo fuente es `arquitectura-agentes7.html`.
3. Para probar cambios: abrir el archivo directamente en Chrome (no requiere servidor).
4. Workflow típico:
   - Leer este contexto y el HTML completo antes de cualquier modificación.
   - Identificar la sección del código a modificar.
   - Hacer cambios quirúrgicos con `str_replace` / targeted edits.
   - Verificar sintaxis con `node --check`.
   - Ejecutar smoke tests con jsdom si el cambio afecta lógica de estado.
5. Siempre mantener el archivo como single-file.
6. Las IDs de los elementos HTML son semánticas y se referencian desde JS — no renombrar sin actualizar JS también.

**Para tests rápidos con jsdom:**
- Las variables `const` globales no se exponen directamente en `window`; acceder con `w.eval('S')`, `w.eval('PAT')`, `w.eval('C')`, etc.
- `w.applyPat(i)`, `w.buildFromWiz()`, `w.genN8NPrompt()`, etc. sí son accesibles directamente.
- Antes de ejecutar, definir stubs: `w.confirm = () => true`, `w.prompt = () => 'valor'`, `w.URL.createObjectURL = () => 'blob:x'`.

---

## 14. PENDIENTES / MEJORAS IDENTIFICADAS

### Funcionalidad faltante
- [ ] **Modo touch/tablet** — zoom/pan con pinch y dos dedos (el pinch de trackpad ya funciona via `ctrlKey+wheel`; falta touch real)
- [ ] **Undo/redo** — historial de estados (snapshot de `S.nodes` + `S.conns` antes de cada mutación)
- [ ] **Snap to grid** — alinear nodos a una cuadrícula de 10px al soltar
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
| 6 | Fix sensibilidad de zoom (1.12 → 1.06) |
| 7 | Auditoría completa + fixes P0/P1 + 5 features nuevas: save/load JSON, rename por dblclick, contratos en prompt n8n, tooltip, mini-mapa. Fixes: stopPropagation en clic de conexiones (P0-1), llm_chain en catálogo (P0-2), clic fantasma post-drag (P1-1), export SVG independiente de zoom/pan (P1-2), clearAll y delSel limpian cFrom y mem-panel (P1-3/P1-5), midpoint bezier real al t=0.5 (Fix 1), LOD adaptativo por zoom (Fix 2), overflow:hidden + rueda=pan + Ctrl+rueda=zoom (Fix 3/4), escA() para inputs con valores de usuario (P2) |

---

*Generado en julio 2026 para uso en Claude Code. Actualizar cuando cambien features o convenciones.*
