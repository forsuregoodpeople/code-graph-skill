---
name: code-graph
description: >
  Full codebase review through visual execution tracing and flow analysis.
  Core capability: trace where data/requests/files/function calls GO — from
  entry point to final response/output — as an interactive directed graph.
  Use this skill whenever the user shares code and wants to understand flow:
  "trace this request", "where does data go", "what happens after X", "show
  execution path", "arah data kemana", "trace function ini", "alur request".
  Also trigger for: dependency graphs, module maps, architecture review,
  complexity heatmaps, "review my code", "analyze codebase", "map the flow".
  Supports Cypress and Playwright test files as trace entry points.
  Always output a visual graph first. Never explain structure in prose alone.
---

# Code Graph Skill

Code review, fully visual. Core job: **trace execution flow** — where does a
request / data / file / function call start, what does it touch, where does it end.

```txt
INPUT:  code files / folder tree / test files / single function
OUTPUT: directed flow graph + minimal prose (graph first, always)
```

---

## Phase 0 — Build Internal Mental Model  ← runs before EVERY graph

Before rendering anything, build a structured internal representation of the code.
This is what enables tracing to any zoom level — from architecture down to a single expression.

### The Symbol Table

Parse every provided file and extract:

```txt
For each FILE:
  path, language, LOC, imports[], exports[]

For each FUNCTION / METHOD:
  name, file, line_start, line_end
  params: [ { name, type? } ]
  returns: type?
  calls: [ { target_name, args_shape, line } ]
  reads: [ variable_name ]       ← variables read from outer scope
  writes: [ variable_name ]      ← variables assigned / mutated
  awaits: boolean
  throws: [ error_type ]
  side_effects: [ "db.write" | "http.call" | "file.write" | "event.emit" | ... ]

For each VARIABLE / BINDING:
  name, file, line
  declared_in: function_name | "module_scope"
  assigned_from: expression_text   ← right-hand side of assignment
  type_inferred: string?
  read_by: [ function_name ]
  mutated_by: [ function_name ]

For each EXPRESSION (Level C–E, built on demand when user drills):
  text: string                  ← exact source text, max 120 chars
  kind: "call" | "access" | "assign" | "await" | "return" | "throw"
  parent: function_name
  line: number
  input_symbols: [ symbol_name ]
  output_symbol: symbol_name?
  output_type: string?          ← ~unknown if unclear
  side_effects: SideEffect[]    ← drives ★ badge at Level C/D/E
  is_async: boolean             ← drives ⟳ badge at Level C/D/E
```

This symbol table is the **mental model**. It is never shown raw to the user —
it only drives what gets rendered in the graph.

### Zoom Levels

```txt
Level A  — Function       default start
           node = function/method block
           edge = function calls another function

Level B  — Statement      one step deeper
           node = meaningful statement (assignment, return, throw, if-branch)
           edge = control flow between statements

Level C  — Expression     one step deeper
           node = single expression (the call itself, the await itself)
           edge = data dependency (output of expr → input of next)

Level D  — Variable       one step deeper
           node = variable / binding
           edge = assignment ("x gets result of"), read ("y reads x")

Level E  — Property       deepest
           node = property access / field read (user.role, req.body.email)
           edge = access chain
```

**Default:** render at Level A. User can say "drill into X" / "zoom ke X" / "lebih detail" to go deeper.

**Level selection in graph UI:**
Render a zoom control: `[A] [B] [C] [D] [E]` — clicking re-renders selected subgraph at that level.
Highlight the active level. Only show levels that have extractable data.

### Build Rules

```txt
1. Build symbol table FIRST, silently — do not narrate this step to user
2. If file is partial / truncated → mark affected symbols as [PARTIAL] — do not guess
3. If type cannot be inferred → use "~unknown" prefix, never fabricate
4. External library calls → mark as [EXTERNAL] node, do not trace into library internals
5. Dynamic dispatch (reflection, eval, dynamic require) → mark edge as [DYNAMIC] — dashed
6. Recursive calls → show as self-loop on node, label with depth limit if known
```

---

## Mode Selection

```txt
  User input
      │
      ├─ "trace X" / "alur X" / "kemana X" / "flow dari X"
      │    → TRACE MODE  ← primary mode
      │    NOTE: if multi-file also present, still use TRACE MODE
      │          Skip OVERVIEW — go straight to trace from named entry point
      │
      ├─ shares multiple files / folder tree  (no trace keyword)
      │    → OVERVIEW MODE → then offer trace on any node
      │
      ├─ Cypress / Playwright test file
      │    → TEST TRACE MODE → extract entry points → trace each
      │
      └─ "review" / "complexity" / "dependencies"
           → REVIEW MODE (structure + problems)

TIEBREAKER RULES (when signals overlap):
  trace keyword + multi-file         → TRACE MODE wins
  trace keyword + test file          → TEST TRACE MODE wins (use test as entry)
  review keyword + trace keyword     → TRACE MODE wins
  multi-file + no keyword            → OVERVIEW MODE, offer trace at end
  ambiguous / no clear signal        → OVERVIEW MODE, ask one question: entry point?
```

---

## TRACE MODE  ← Core Feature

**Goal:** given any starting point, draw the full directed path to the final output.
Uses the mental model built in Phase 0. Default zoom = Level A. User drills deeper on demand.

### What counts as a starting point
```txt
HTTP Request    → route definition (GET /api/users, POST /login, etc.)
Function call   → any named function the user points to
File operation  → read/write/upload trigger
Event           → click handler, webhook, queue consumer, cron job
Variable        → trace where this variable comes from and where it goes
Expression      → trace what this expression produces and who consumes it
Test action     → cy.request(), page.goto(), cy.visit()
```

### Trace Graph Structure
```txt
Each node = one unit at the current zoom level (function / statement / expression / variable / property)
Each edge = direction of control or data flow, labeled with what passes through

Node types (apply at any zoom level):
  [ENTRY]      → where execution begins (green)
  [ROUTER]     → route matching / middleware dispatch (blue)
  [MIDDLEWARE] → auth, validation, rate limit, logging (amber)
  [CONTROLLER] → request handler (blue)
  [SERVICE]    → business logic (blue)
  [REPOSITORY] → data access layer (blue)
  [DB/STORE]   → database, cache, file system (gold)
  [EXTERNAL]   → third-party API, queue, email service (gold)
  [RESPONSE]   → final output — HTTP response, return value, file write (green)
  [ERROR]      → error path — exception thrown, error response (red)
  [PARTIAL]    → symbol extracted from truncated/incomplete file (dashed border)
  [DYNAMIC]    → dynamic dispatch — runtime target unknown (dashed edge)

Edge labels:
  → params / payload    (what data moves)
  → return value        (what comes back)
  → error thrown        (when error path diverges)
  → variable: type      (at Level D/E)
```

### Trace Graph Rules
```txt
1. ALWAYS show both happy path AND error path
   → happy path = solid edges
   → error path = dashed red edges

2. Label every edge with what moves through it
   (e.g., "userId: string", "{ user, token }", "Error: 401")

3. Show WHERE data transforms (shape changes)
   → node gets ⊕ badge  (NOT ⟳ — ⟳ is async only)

4. Show WHERE side effects happen
   → node gets ★ badge

5. Conditional branches → show fork explicitly
   → if (user.role === 'admin') → two edges out, each labeled with condition

6. Async boundaries → ⟳ badge on edge and node
   → await, Promise, callback, queue
   → ⟳ = async ONLY. Never use ⟳ for data transform.

7. End node always explicit
   → res.json() / return value / file written / event emitted

8. Zoom level control always visible in graph UI — TRACE MODE only
   → [A] [B] [C] [D] [E] — active level highlighted
   → clicking level re-renders current trace at that depth
   → only enable levels with extractable data from symbol table
   → OVERVIEW and REVIEW MODE do not show zoom control

9. "Drill into" on any node → re-render that node expanded at Level+1
   → rest of graph stays at current level (mixed-level view allowed)
   → MAX DRILL DEPTH: 2 levels below current active zoom
     e.g. active = A → can drill to B or C, not D/E directly
   → "Collapse" button on expanded node → returns to parent zoom level
```

### Zoom Level Rendering per Level

```txt
Level A — Function (default)
  Node label:   functionName()
  Node tooltip: file path, LOC, params, return type
  Edge label:   what args pass, what returns

Level B — Statement
  Node label:   abbreviated statement text (max 40 chars)
                e.g., "const user = await repo.find()"
  Node tooltip: full statement, line number
  Edge label:   variable name that flows out

Level C — Expression
  Node label:   expression text
                e.g., "repo.find(email)"
  Node tooltip: full call signature
  Edge label:   output type / value

Level D — Variable
  Node label:   varName: ~type
  Node tooltip: declared at line X, assigned from Y, read by [...]
  Edge label:   "assigned from" / "read by" / "mutated by"

Level E — Property
  Node label:   object.property
  Node tooltip: access at line X, type inferred: ~T
  Edge label:   "accessed from" / "passed into"
```

---

## Graph Renderer vs Target Codebase

```txt
The interactive graph is rendered as a plain HTML artifact (inline SVG + vanilla JS).
No React, no framework — just HTML, CSS, SVG, and vanilla JS in one file.

The TARGET CODEBASE being analyzed can be anything:
  Vue, Svelte, Angular, jQuery, vanilla JS/HTML/CSS, Alpine.js, Astro,
  Laravel, Express, Django, Go, Python, Kotlin, PHP, Ruby — anything.
```

---

## Architecture Detection for Trace

Before tracing, detect which architecture pattern AND which framework/stack is used.
These are two separate detections — architecture shapes the layers, framework shapes the signals.

### Backend Architectures

```txt
MVC (generic)   → Model / View / Controller
                  signals: controllers/, models/, views/ folders
                  or: *Controller.*, *Model.* naming

Clean Arch      → Entity / Use Case / Interface / Infrastructure
                  signals: domain/, application/, infrastructure/ folders
                  or: *UseCase.ts, *Repository.ts, *Entity.ts naming

Hexagonal       → Core / Port / Adapter
                  signals: ports/, adapters/, core/ folders

Layered         → Presentation / Business / Data
                  signals: services/, repositories/, handlers/ folders

Laravel MVC     → Route → Controller → Model → DB
                  signals: routes/web.php, routes/api.php, app/Http/Controllers/
                  trace: web.php route → Controller@method → Model::query() → DB

Express / Koa   → Router → Middleware chain → Handler → Response
                  signals: router.get/post, app.use(), middleware files, req/res params
                  trace: app.use() → router.METHOD() → handler fn → res.json()

Fastify         → Plugin → Route → Handler → Reply
                  signals: fastify.register(), fastify.get/post(), reply.send()

NestJS          → Module → Controller → Service → Repository
                  signals: @Module, @Controller, @Injectable, @InjectRepository

Django          → URL → View → Serializer → Model → DB
                  signals: urls.py, views.py, serializers.py, models.py
                  trace: urlpatterns → View.get/post() → Serializer → Model.objects

FastAPI         → Router → Endpoint → Dependency → DB
                  signals: @app.get/post, Depends(), async def, SQLAlchemy

Go (net/http)   → Mux → Handler → Service → DB
                  signals: http.HandleFunc, mux.Handle, ServeHTTP

Go (Gin/Echo)   → Engine → Route group → Handler → Response
                  signals: gin.Default(), r.GET/POST, c.JSON()

Rails           → Route → Controller → Model → DB
                  signals: routes.rb, *_controller.rb, ApplicationRecord

Spring Boot     → Controller → Service → Repository → DB
                  signals: @RestController, @Service, @Repository, @Autowired
```

### Frontend Architectures

```txt
Vanilla JS/HTML/CSS  → DOM ready → Event listener → Handler → DOM mutation
                        signals: document.addEventListener, querySelector, innerHTML
                        no framework imports, plain .js + .html + .css files
                        trace: DOMContentLoaded → addEventListener → fn() → DOM update

jQuery              → DOM ready → .on() handler → AJAX / DOM mutation
                      signals: $(document).ready / $(fn), $.ajax / $.get / $.post,
                               .on(), .click(), .val(), .html(), .append()
                      trace: $(document).ready → $('sel').on('event') → callback → $.ajax → DOM

Vue 2 (Options API) → Component → data/computed/methods → Template → DOM
                      signals: new Vue(), Vue.component(), export default { data(), methods: {} }
                      trace: created/mounted → method() → data mutation → re-render

Vue 3 (Composition API) → setup() → ref/reactive → watchEffect → Template
                           signals: defineComponent, setup(), ref(), reactive(),
                                    computed(), watch(), onMounted()
                           trace: setup() → ref() → computed() → template binding → DOM

Nuxt.js             → pages/ → asyncData/fetch → store → component
                      signals: pages/, layouts/, store/, nuxt.config.js
                      useAsyncData(), useFetch(), definePageMeta()

Svelte              → Component → reactive declarations → store → DOM
                      signals: .svelte files, $: reactive, import { writable } from 'svelte/store'
                      trace: onMount → $: statement → store.update() → reactive DOM

SvelteKit           → +page.svelte → load() → fetch → render
                      signals: +page.svelte, +page.server.ts, +layout.svelte, load()

Angular             → Module → Component → Service → Template
                      signals: @NgModule, @Component, @Injectable, @Input/@Output,
                               ngOnInit(), HttpClient

React (if target)   → Component → useState/useEffect → fetch → re-render
                      signals: .jsx/.tsx, useState, useEffect, import React

Next.js             → Page/API route → Server component → fetch → Response
                      signals: pages/api/, app/api/, route.ts, getServerSideProps

Astro               → .astro files → frontmatter → fetch → static HTML
                      signals: .astro, ---frontmatter---, Astro.props, getStaticPaths()

Alpine.js           → x-data → x-on / x-bind → x-effect → DOM
                      signals: x-data, x-on:click, x-bind, x-show, x-model, Alpine.data()
                      trace: x-data init → x-on:event → method() → reactive DOM update

HTMX                → HTML attr → hx-get/post → server response → DOM swap
                      signals: hx-get, hx-post, hx-trigger, hx-target, hx-swap
                      trace: user trigger → hx-get request → server HTML → hx-swap into DOM

Vanilla CSS only    → Selector → Property → Computed style → Paint
                      signals: .css files only, no JS
                      trace: specificity chain → cascade → computed value → visual output
```

### Mobile / Cross-platform

```txt
React Native        → Component → state → Native bridge → Native UI
                      signals: .tsx + react-native imports, StyleSheet, Platform
                      trace: useState → setState → bridge → native render

Flutter (Dart)      → Widget → build() → setState → re-render
                      signals: .dart, extends StatefulWidget, setState(), build()

Kotlin Android      → Activity/Fragment → ViewModel → Repository → DB/API
                      signals: .kt, override fun onCreate, ViewModel, LiveData/Flow

Swift iOS           → ViewController → Model → URLSession → UI update
                      signals: .swift, viewDidLoad, @IBAction, URLSession.shared
```

### Unknown / Generic fallback

```txt
Unknown             → use folder/file naming heuristics + import patterns
                      label layers generically: [INPUT] → [PROCESS] → [OUTPUT]
                      note detected language in graph title
```

Apply detected architecture labels to trace nodes automatically.
Show architecture name in graph title: `Trace — Laravel MVC` etc.

**Architecture detection tiebreaker:**
```txt
If signals from multiple patterns are present, apply this priority order:
  1. Framework-specific signals win over generic folder naming
     (routes/web.php → Laravel MVC beats generic services/ folder)
  2. More specific pattern wins over generic
     (Clean Arch beats Layered if domain/ + application/ + infrastructure/ all present)
  3. File naming convention wins over folder naming alone
     (*UseCase.ts → Clean Arch even if folders are generic)
  4. If still ambiguous → pick the pattern with the most matching signals
     count matches per pattern, use highest score
  5. If tied → label as "Layered (detected)" and note ambiguity in observation
```

---

## OVERVIEW MODE

Run when user shares multiple files without a specific trace target.

### Step 1 — Inventory
```txt
- File count + language detected
- Architecture pattern detected (see above)
- Entry points found (routes, main, index, CLI args, cron, events)
- External dependencies (package.json, composer.json, go.mod, etc.)
```

### Step 2 — Architecture Layer Graph
Render as HTML artifact (inline SVG + vanilla JS):
- All files as nodes, grouped by detected layer
- Import/dependency edges between them
- Node size proportional to LOC
- Color by layer (use Architecture Layer Color Bands from trace-rendering.md)
- Nodes clickable → highlight all dependents/dependencies
- Follow with static ASCII architecture map below the graph

### Step 3 — Problem Detection
```txt
[ ] Circular imports
[ ] God files (>300 LOC)
[ ] Business logic leaking into wrong layer
    (e.g., DB query inside Controller, HTTP call inside Model)
[ ] Missing error boundaries
[ ] Dead exports (exported but never imported)
[ ] Deep nesting (>4 levels)
```

### Step 4 — Offer Trace
After overview graph, always offer:
```txt
"Mau trace alur eksekusi dari titik mana?
  → [list detected entry points]
  → atau ketik nama function / route spesifik"
```

---

## TEST TRACE MODE  (Cypress & Playwright)

When user shares test files with Cypress or Playwright imports:
```txt
File extensions recognized:
  .cy.js  .cy.ts                          ← Cypress
  .spec.js  .spec.ts                      ← Cypress or Playwright
  .test.js  .test.ts                      ← Playwright or Vitest/Jest with Playwright
  *.e2e.js  *.e2e.ts                      ← end-to-end tests (any runner)
```

### Step 1 — Extract Test Entry Points
```txt
Cypress signals:
  cy.visit('/path')          → HTTP GET entry
  cy.request('POST', '/api') → HTTP POST entry
  cy.intercept(...)          → mock boundary (mark as [MOCK] node)

Playwright signals:
  page.goto('/path')         → HTTP GET entry
  page.request.post('/api')  → HTTP POST entry
  page.route(...)            → mock boundary
```

### Step 2 — Map Test Coverage
For each entry point found in tests:
```txt
- Trace which route/handler it hits (if app code provided)
- If app code NOT provided: render test flow graph only
  (describe block → it block → actions → assertions)
- Mark assertions as [ASSERT] nodes (purple)
- Mark mocked boundaries as [MOCK] nodes (dashed border)
```

### Step 3 — Coverage Gap Detection
```txt
If both test files AND app code provided:
  → Show which routes/functions have NO test coverage (gray + ⚠️)
  → Show which tests cover which layers (overlay on architecture graph)
```

### Test Flow Graph Node Types
```txt
[DESCRIBE]  → test suite container
[IT/TEST]   → individual test case
[ACTION]    → user action (click, fill, visit, request)
[ASSERT]    → expect / should assertion
[MOCK]      → intercepted / mocked boundary
[COVERAGE]  → app node that this test reaches
```

### Output Format
Render as HTML artifact (inline SVG + vanilla JS):
- Test flow graph: DESCRIBE → IT blocks → ACTION → ASSERT chain
- If app code also provided: overlay coverage on architecture layer graph
- Nodes clickable → show which test file + line triggers this node
- Follow with static ASCII test flow map below the graph

---

## REVIEW MODE

For explicit code quality / dependency review requests.

### Graphs to render
```txt
- Dependency Graph   → who imports whom, circular deps highlighted
- Complexity Heatmap → node size = LOC, color = complexity
- Problem nodes      → red for issues, gray for dead code
```

### Output Format
Render as HTML artifact (inline SVG + vanilla JS):
- No zoom control (REVIEW MODE does not use zoom levels)
- Nodes clickable → highlight dependents/dependencies
- Problem nodes rendered in red, dead code in gray
- Complexity heatmap: green (simple) → yellow (medium) → red (complex)
- Follow with static ASCII dependency map below the graph

Problem detection same as OVERVIEW MODE Step 3.

---

## Output Format Rules

```txt
1. GRAPH FIRST — always render diagram before any prose

2. HTML artifact for interactive graphs
   → plain HTML + inline SVG + vanilla JS — no framework
   → nodes clickable → highlight connected path
   → hover tooltip → show: file path, LOC, function signature, layer
   → "Trace from here" button on every node → re-render trace from that node
   → zoom control [A][B][C][D][E] always visible in graph UI (TRACE MODE only)
   → ASCII flowchart panel BELOW the interactive graph, synced to active zoom level

3. ASCII-only output when:
   → user is in a terminal / plain text context
   → graph has ≤8 nodes (ASCII alone is sufficient)
   → user explicitly asks for ASCII only

4. Color coding (consistent per session):
   Entry/Response  = #3aaa8c  (Palm Green)
   Internal node   = #668db8  (Miracloud Blue)
   External/DB     = #F0A500  (Mango Gold)
   Error path      = #e05252  (Red)
   Dead/Mock       = #888888  (Gray)
   Middleware      = #c49a3c  (Amber)
   Assert (tests)  = #9b59b6  (Purple)

5. Always show legend
6. Edge labels always visible (not just on hover) for flow graphs
```

---

## ASCII Flowchart — Mandatory Companion Output

Every trace graph ships with an ASCII flowchart. It is not optional.
The ASCII version is the portable, copy-pasteable, terminal-friendly form of the same graph.

### When to render

```txt
HTML interactive graph    → ASCII panel embedded below graph, syncs with zoom level
ASCII-only response       → ASCII block as main output (no HTML artifact)
Any TRACE MODE response   → always, at every zoom level
OVERVIEW MODE             → architecture ASCII after the overview graph
TEST TRACE MODE           → test flow ASCII after the coverage graph
```

### ASCII Symbols (use consistently)

```txt
Flow direction:
  │   vertical flow (top-down)
  ▼   downward arrow
  ▶   rightward arrow (error paths, side branches)
  ─   horizontal connector
  ┌ ┐ └ ┘   box corners
  ├ ┤ ┬ ┴ ┼  box junctions

Paths:
  ───   happy path (solid)
  ╌╌╌   error / exception path (dashed)
  ─ ─   return / response path (spaced dashes)

Badges (inline, after node label):
  ★   side effect (db write, http call, file write, event emit)
  ⟳   async boundary (await, Promise, goroutine, coroutine)
  ?   conditional branch (if/switch that forks execution)
  ⊕   data transform (shape of data changes here)

Edge labels:
  written inline between nodes, e.g.:
  │ { email, password }
  ▼
  or on the same line as the connector:
  ╌╌ Error: 401 ╌╌╌╌╌╌╌╌╌╌╌╌╌╌▶ [ERROR node]
```

### ASCII Structure per Zoom Level

**Level A — Function**
```
[ENTRY: trigger]
      │
      ▼ { edge label }
┌─────────────────────┐
│  FunctionName()     │ [badges]
└──────────┬──────────┘
           │                   ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌▶ [ERROR: type status]
           ▼ { return label }
┌─────────────────────┐
│  NextFunction()     │ [badges]
└──────────┬──────────┘
           ▼
     [RESPONSE: final output]
```

**Level B — Statement**
```
FunctionName() — statements
  ┌──────────────────────────────────────┐
  │ const x = await repo.find(param) ⟳  │
  └──────────────┬───────────────────────┘
                 │ x: Type|null
                 ▼
  ┌──────────────────────────────────────┐
  │ if (!x)                          ?  │
  └──────┬──────────────────┬───────────┘
         │ x !== null        │ x === null
         ▼                   ▼
  ┌────────────┐     ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌
  │ next step  │     throw Error(code)
  └────────────┘     ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌
```

**Level C — Expression**
```
expr1(args)
      │ return: Type
      ▼
expr2(result_of_expr1)
      │ return: Type2
      ▼
expr3->prop
      │ return: FinalType
      ▼
[final value]
```

**Level D — Variable**
```
◇ varA: Type ──────────────────────┐
                                    │ passed into
◇ varB: Type ──────────────────┐   ▼
                                │  someFunction()
◇ result: ReturnType ◀─────────┘
    │ read_by: [fn1, fn2]
    │ mutated_by: none
    ▼
[next consumer]
```

**Level E — Property**
```
$obj
  └─.method(args)
       │ returns ComplexType
       └─.property
            │ returns string
            └─ [final value — passed into X]
```

### ASCII Rules

```txt
1. Box width: align all boxes in same column to same width
   → find longest label in column, pad others with spaces

2. Error paths always on the RIGHT side of the column
   → main flow: center column
   → errors: branch right with ╌╌╌╌╌╌╌╌╌╌╌▶

3. Return paths (dashed, going UP or sideways):
   → use ─ ─ ─ with label inline
   → or vertical return: line on right side of column going up

4. Conditional fork: show BOTH branches explicitly
   → label each branch arm with the condition
   → rejoin below if branches converge, or terminate separately

5. Side effects: show as indented branch off main flow
   → ├─ ★ sideEffect() ─▶ [EXTERNAL node]

6. Legend always at bottom of ASCII block:
   ─── happy path    ╌╌╌ error path    ─ ─ return value
   ★ side effect    ⟳ async    ? branch    ◇ variable
```

### Zoom Sync in Interactive Graph

When the HTML graph has zoom controls [A][B][C][D][E] (TRACE MODE only):
- The ASCII panel below the graph **updates when zoom level changes**
- Each zoom level has its own pre-rendered ASCII string
- ASCII is shown in a monospace code block with scroll if long
- Label above ASCII panel: `ASCII Flowchart — Level [X]`
- OVERVIEW and REVIEW MODE: render one static ASCII block, no zoom sync

---

## Observation Format (below graph)

```txt
✅ [Strong point]       → max 2 points
⚠️  [Worth watching]    → max 3 points
🔴 [Needs attention]    → one line per problem node
```

Max 4 bullets total. No paragraphs.

---

## Drill-Down Offer (end of every response)

Tailor options to the active mode — do not show irrelevant options.

```txt
TRACE MODE:
  "Mau lanjut ke mana?
    A) Zoom ke Level B — lihat per statement di [current node]
    B) Lihat error path dari [problem node]
    C) Trace entry point lain: [list other detected entry points]
    D) Switch ke REVIEW MODE — analisa struktur & complexity"

OVERVIEW MODE:
  "Mau lanjut ke mana?
    A) Trace alur dari [detected entry point 1]
    B) Trace alur dari [detected entry point 2]
    C) Zoom ke subsystem [problem layer]
    D) Lihat dependency graph detail"

TEST TRACE MODE:
  "Mau lanjut ke mana?
    A) Trace app code yang di-hit oleh [test name]
    B) Lihat coverage gap — route/function tanpa test
    C) Drill ke mock boundary [MOCK node]"

REVIEW MODE:
  "Mau lanjut ke mana?
    A) Trace alur dari [detected entry point]
    B) Detail problem node: [list red nodes]
    C) Lihat circular dependency chain"
```

---

## References

```txt
problem-patterns.md        Anti-pattern catalog per language/framework
trace-rendering.md         Happy/error path rules, edge labels, async chains, Cypress/Playwright
mental-model.md            Symbol table schema, zoom level rendering detail, partial file handling
```

---

## Quick Checklist

```txt
PHASE 0 — Mental Model
[ ] Symbol table built from all provided files?
[ ] Partial/truncated symbols marked [PARTIAL]?
[ ] External libs marked [EXTERNAL], not traced into?
[ ] Dynamic dispatch marked [DYNAMIC] with dashed edge?

GRAPH OUTPUT
[ ] Mode selected (TRACE / OVERVIEW / TEST TRACE / REVIEW)?
[ ] Architecture detected and labeled?
[ ] Graph rendered first?
[ ] Zoom level control [A][B][C][D][E] visible in graph UI? (TRACE MODE only — not OVERVIEW/REVIEW)
[ ] Default zoom = Level A unless user specified otherwise?
[ ] Both happy path AND error path shown (TRACE mode)?
[ ] Edge labels show what data moves?
[ ] Side effects marked ★, transforms marked ⊕, async marked ⟳?
[ ] Legend present?
[ ] Drill-down offer at end?
[ ] Max 4 prose bullets?

ASCII FLOWCHART
[ ] ASCII flowchart present for every trace response?
[ ] ASCII synced to active zoom level (all 5 levels pre-rendered)?
[ ] Error paths on RIGHT with ╌╌╌╌▶?
[ ] Happy path center column with │ ▼ ┌─┐?
[ ] Conditional forks show BOTH arms labeled?
[ ] Side effects as ├─ ★ branch?
[ ] Legend at bottom of ASCII block?
[ ] Box widths aligned within same column?
```