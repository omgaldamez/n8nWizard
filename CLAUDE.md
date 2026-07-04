# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**n8nWizard** is a single-file visual architecture designer for AI agents, built entirely in vanilla HTML/CSS/JS with no build step and no dependencies. It targets non-technical creative professionals (CENTRO Educación Continua, PCIA Module 4 program in Mexico City) who need to visually design AI agent architectures before implementing them in n8n.

## Running the App

There is no build step. Open `index.html` directly in a browser:
```
# Any of these work:
open index.html
# Or double-click it in the file system
```

There are no npm packages, no package.json, no transpilation, no linter config.

## Single-File Constraint

**All code lives in `index.html`** (~2100 lines of inline HTML + CSS + JS). Do not split into separate files — the single-file constraint is intentional (zero friction to open/share). CONTEXT.md is the project specification bible; consult it for detailed domain logic.

## Architecture

### Global State

Everything lives in `const S`:
```js
S.nodes   // {id → {id, type, x, y, label?, swData?}}
S.conns   // {id → {id, from, to, fields?}}
S.sel     // currently selected node/conn id
S.vx, S.vy, S.vs  // viewport pan X/Y and zoom scale
S.wizStep, S.wd   // wizard step and wizard form data
S.nc, S.cc        // node/connection ID counters
```

### Render Pipeline

The app uses **immediate-mode rendering**: every state mutation calls `render()`, which clears the SVG innerHTML and rebuilds from scratch. There is no diffing or virtual DOM.

```
User Action → mutate S.nodes/S.conns → render()
  ├─ renderConns()   bezier paths + contract badges
  ├─ renderNodes()   LOD-adaptive rectangles + icons + labels
  ├─ renderStatus()  bottom status bar
  └─ renderMinimap() 160×100px navigation widget
```

### SVG Canvas Structure
```html
<svg id="cv">
  <defs><marker id="ah">...</marker></defs>  <!-- arrowhead -->
  <g id="canvas-root" transform="translate(vx,vy) scale(vs)">
    <g id="cl"></g>  <!-- connections -->
    <g id="nl"></g>  <!-- nodes -->
  </g>
</svg>
```

### Key Data Objects

- **`C`** — Component registry: 35 component types across 8 categories (entrada, proceso, fuente, modelo, salida, obs, datos, custom). Each has `{label, cat, hex, icon, desc}`.
- **`PAT`** — 7 predefined pattern templates. Applied via `applyPat(idx)`.
- **`LLMS`** — 17 LLM models with pricing, used in wizard step 6.

### LOD (Level of Detail)

Nodes adapt based on `NH * zoom`:
- `< 10px`: rect only
- `< 20px`: rect + centered icon
- `≥ 20px`: full label + sublabel + icon

### Connections

Cubic Bezier paths. Bidirectional pairs are detected and offset by 12px. `connGeom()` calculates control points; `bezMid()` finds the real midpoint at t=0.5 (not centroid). Contracts (JSON schemas) can be attached per connection.

### ID Conventions

- Node IDs: `n1`, `n2`, ...  (counter: `S.nc`)
- Connection IDs: `c1`, `c2`, ...  (counter: `S.cc`)

### Wizard (6 Steps)

Steps 0–6 guide users from pattern selection → input channels → classifier → memory → RAG → LLM. Data accumulates in `S.wd`; `buildFromWiz()` constructs the canvas from it. Step 0 shows 7 template cards (shortcuts to `applyPat(idx)`).

### Export Paths

| Output | Function |
|--------|----------|
| SVG download | `doExpSVG()` |
| JSON save (v1 format) | `saveDiagram()` |
| Markdown + PlantUML | `genMD()` / `genPlantUML()` |
| Claude/n8n prompt | `genN8NPrompt()` |
| Sequence diagram SVG | `doExpSeqSVG()` → `renderSeq()` |

## Security

- **Always use `escA()`** to escape user input before inserting into SVG attribute strings. It escapes `&`, `"`, `<`.
- SVG elements are created via `createElementNS()` (not `innerHTML`) to avoid XSS.
- No backend, no external API calls. All processing is client-side.

## Tech Stack Context (What the Diagrams Represent)

The tool generates prompts and diagrams targeting this specific stack:
- **n8n** v2.1.4 (self-hosted, Docker, Windows)
- **Database:** Supabase / PostgreSQL
- **Vector DB:** Qdrant v1.14.1
- **LLM via:** OpenRouter (lmChatOpenRouter node), default: Gemini 2.5 Flash
- **Embeddings:** gemini-embedding-001 (3072 dims)
- **Chat:** Telegram Bot
- **Observability:** Langfuse (REST/Basic Auth)
- **Timezone:** America/Mexico_City (UTC-6, no DST)
