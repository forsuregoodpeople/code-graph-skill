---
name: code-graph
description: >
  Full codebase review through visual execution tracing and flow analysis.
  Core capability: trace where data/requests/files/function calls GO — from
  entry point to final response/output — as an interactive directed graph.
  Use this skill whenever the user shares code and wants to understand flow:
  "trace this request", "where does data go", "what happens after X", "show
  execution path", "where does data go", "trace this function", "request flow".
  Also trigger for: dependency graphs, module maps, architecture review,
  complexity heatmaps, "review my code", "analyze codebase", "map the flow".
  Supports Cypress and Playwright test files as trace entry points.
  Always output a visual graph first. Never explain structure in prose alone.
---

# Code Graph Skill

Trace execution flow — where does a request/data/function start, what does it touch, where does it end.
Graph first, always. Prose is secondary.

---

## Phase 0 — Mental Model (runs before every graph, silent)

Build a symbol table from all provided files. Never show it raw — it only drives rendering.

```txt
FILE:       path, language, LOC, imports[], exports[]
FUNCTION:   name, file, line_start/end, params, returns, calls[], reads[], writes[],
            awaits, throws[], side_effects[]
VARIABLE:   name, file, line, declared_in, assigned_from, type_inferred, read_by[], mutated_by[]
EXPRESSION: (Level C–E, built on demand) text, kind, parent, line, input_symbols[],
            output_symbol, output_type, side_effects[], is_async
```

**Zoom levels** — default Level A, user drills deeper on demand:
```txt
A  Function   node=function/method,      edge=function calls function
B  Statement  node=meaningful statement, edge=control flow
C  Expression node=single expression,    edge=data dependency
D  Variable   node=variable/binding,     edge=assignment / read
E  Property   node=property access,      edge=access chain
```

**Build rules:**
```txt
- Partial/truncated file → mark symbols [PARTIAL], never guess
- Unknown type → prefix "~unknown", never fabricate
- External lib call → [EXTERNAL] node, do not trace into internals
- Dynamic dispatch → [DYNAMIC] edge, dashed
- Recursive call → self-loop, label depth limit if known
```

---

## Mode Selection

```txt
"trace X" / "flow of X" / "where does X go"    → TRACE MODE (primary)
  + multi-file present                           → still TRACE MODE, skip overview
multiple files / folder tree, no trace keyword  → OVERVIEW MODE → AUTO-PROCEED to full trace
Cypress / Playwright test file                  → TEST TRACE MODE
"review" / "complexity" / "dependencies"        → REVIEW MODE

Tiebreakers: trace > review > overview. Ambiguous + zero detectable entry points → ask one question.
```

---

## TRACE MODE

Given any starting point, draw the full directed path to final output. Level A default.

**Entry point types:** HTTP route, function call, file operation, event/webhook/cron, variable, expression, test action (`cy.request`, `page.goto`)

**Node types:**
```txt
[ENTRY]      green   — execution start
[ROUTER]     blue    — route matching / dispatch
[MIDDLEWARE] amber   — auth, validation, rate limit, logging
[CONTROLLER] blue    — request handler
[SERVICE]    blue    — business logic
[REPOSITORY] blue    — data access
[DB/STORE]   gold    — database, cache, filesystem
[EXTERNAL]   gold    — third-party API, queue, email
[RESPONSE]   green   — final output
[ERROR]      red     — exception / error response
[PARTIAL]    dashed border — truncated file symbol
[DYNAMIC]    dashed edge  — runtime dispatch target unknown
```

**Trace rules:**
```txt
1. Happy path always shown — solid edges, top to bottom
2. Error/warning paths ONLY if anomaly pattern detected (see below)
3. Every edge labeled with what moves (e.g. "{ email, password }", "User | null", "Error: 401")
4. Data transform → ⊕ badge on node (never ⟳ — ⟳ is async only)
5. Side effect → ★ badge on node
6. Async boundary → ⟳ badge on node and edge (await, Promise, goroutine, callback, queue)
7. Conditional branch → fork explicitly, label each arm with condition
8. End node always explicit (res.json / return value / file written / event emitted)
9. Zoom control [A][B][C][D][E] always visible in graph UI (TRACE MODE only)
10. "Drill into" any node → expand to Level+1 (max 2 levels below active); "Collapse" to return
```

---

## Anomaly Detection (run before rendering, evidence-based only)

Scan symbol table for patterns. Draw error/warning paths ONLY where a pattern is found.
No pattern → no annotation. State pattern name explicitly on every flagged edge.

**SECURITY + FATAL → 🔴 error path (╌╌ PATTERN_NAME ╌╌▶)**
```txt
SECURITY:
  SQL_INJECTION       raw user input into DB query, no parameterize/ORM
  AUTH_BYPASS         route reachable without required auth middleware
  EXPOSED_SECRET      API key / token hardcoded or passed to logger
  UNVALIDATED_INPUT   req.body / args → service/DB with no schema check
  IDOR                resource fetched by user-supplied ID, no ownership check

FATAL:
  NIL_DEREF           nullable accessed before nil/null check
  UNHANDLED_PROMISE   async call, no await AND no .catch() — silent crash
  MISSING_RETURN      typed function has code path with no return value
  DEADLOCK_RISK       mutex locked, no guaranteed unlock path / circular goroutine wait
  INFINITE_LOOP       loop condition cannot become false given data in scope
```

**WARNING → 🟡 inline annotation (~~▶ ⚠ PATTERN_NAME)**
```txt
NULL_UNGUARDED      returns T | null, consumer accesses without guard
SWALLOWED_ERROR     error caught but not propagated or logged
MISSING_AWAIT       async called without await, result is Promise<T> not T
SHAPE_DRIFT         object transformed, downstream still accesses pre-transform fields
TYPE_MISMATCH       upstream produces type X, downstream expects type Y
KEY_MISMATCH        publish key and consumer listen key differ (typo / case / prefix)
UNCLOSED_RESOURCE   DB connection / file handle opened, no close/defer path
```

---

## Architecture Detection

Detect architecture pattern AND framework/stack before tracing. Apply labels to nodes automatically.
Show in graph title: `Trace — NestJS Clean Arch`.

**Backend:**
```txt
Laravel MVC    routes/web.php, routes/api.php, Http/Controllers/  → Route→Controller→Model→DB
Express/Koa    router.get/post, app.use(), req/res params          → Router→Middleware→Handler→Response
Fastify        fastify.register(), reply.send()                    → Plugin→Route→Handler→Reply
NestJS         @Module @Controller @Injectable @InjectRepository   → Module→Controller→Service→Repo
Django         urls.py, views.py, serializers.py, models.py       → URL→View→Serializer→Model→DB
FastAPI        @app.get/post, Depends(), async def, SQLAlchemy     → Router→Endpoint→Dependency→DB
Go net/http    http.HandleFunc, ServeHTTP                          → Mux→Handler→Service→DB
Go Gin/Echo    gin.Default(), r.GET/POST, c.JSON()                 → Engine→RouteGroup→Handler→Response
Rails          routes.rb, *_controller.rb, ApplicationRecord       → Route→Controller→Model→DB
Spring Boot    @RestController @Service @Repository @Autowired     → Controller→Service→Repo→DB
Generic MVC    controllers/, models/, views/ or *Controller.*      → Model/View/Controller
Clean Arch     domain/, application/, infrastructure/ or *UseCase  → Entity→UseCase→Interface→Infra
Hexagonal      ports/, adapters/, core/                            → Core→Port→Adapter
Layered        services/, repositories/, handlers/                 → Presentation→Business→Data
```

**Frontend:**
```txt
Vanilla JS     addEventListener, querySelector — no framework      → DOMReady→EventListener→Handler→DOM
jQuery         $(fn), .on(), $.ajax                                → DOMReady→.on()→callback→$.ajax→DOM
Vue 2          new Vue(), export default { data(), methods:{} }    → created/mounted→method→data→render
Vue 3          setup(), ref(), reactive(), onMounted()             → setup→ref→computed→template→DOM
Nuxt.js        pages/, useAsyncData(), useFetch()                  → page→asyncData→store→component
Svelte         .svelte, $: reactive, writable store                → onMount→$:→store.update→DOM
SvelteKit      +page.svelte, +page.server.ts, load()              → load→fetch→render
Angular        @NgModule @Component @Injectable, ngOnInit()        → Module→Component→Service→Template
React          .jsx/.tsx, useState, useEffect                      → Component→state→effect→render
Next.js        pages/api/, app/api/, route.ts                     → Page/APIRoute→ServerComp→fetch→Response
Astro          .astro, ---frontmatter---, getStaticPaths()         → frontmatter→fetch→staticHTML
Alpine.js      x-data, x-on, x-bind, x-model                     → x-data→x-on→method→DOM
HTMX           hx-get/post, hx-trigger, hx-target, hx-swap       → trigger→hx-request→serverHTML→swap
```

**Mobile:**
```txt
React Native   .tsx + react-native, StyleSheet                     → Component→state→NativeBridge→UI
Flutter        .dart, StatefulWidget, setState(), build()          → Widget→build→setState→render
Kotlin Android .kt, ViewModel, LiveData/Flow                       → Activity→ViewModel→Repository→DB
Swift iOS      .swift, viewDidLoad, URLSession                     → ViewController→Model→URLSession→UI
```

**Tiebreaker:** framework-specific signals > generic folder naming > file naming convention > most matches > tied → "Layered (detected)"

**Unknown fallback:** folder/file heuristics + import patterns → label `[INPUT]→[PROCESS]→[OUTPUT]`

---

## OVERVIEW MODE

Runs when multiple files shared with no trace keyword. Do NOT stop — auto-proceed.

```txt
Step 1 — Inventory: file count, language, architecture, entry points, external deps
Step 2 — Architecture Layer Graph (HTML artifact: files as nodes by layer, import edges,
          node size ∝ LOC, clickable highlight; + static ASCII map below)
Step 3 — Problem Detection:
          [ ] Circular imports
          [ ] God files (>300 LOC)
          [ ] Business logic in wrong layer (DB query in Controller, HTTP call in Model)
          [ ] Missing error boundaries
          [ ] Dead exports (exported, never imported)
          [ ] Deep nesting (>4 levels)
Step 4 — AUTO-PROCEED to full TRACE MODE:
          Priority: HTTP route > main()/index > CLI > event consumer > cron > other
          State entry point chosen: "Continuing trace from: [entry_point] — [reason]"
          Trace top-to-bottom, all the way to final output. Do NOT stop mid-trace.
          Show Drill-Down Offer after trace completes (include other entry points as options).
```

---

## TEST TRACE MODE

Triggered by Cypress / Playwright files (`.cy.js`, `.cy.ts`, `.spec.*`, `.test.*`, `*.e2e.*`).

```txt
Entry point signals:
  Cypress:    cy.visit() → GET, cy.request('POST') → POST, cy.intercept() → [MOCK]
  Playwright: page.goto() → GET, page.request.post() → POST, page.route() → [MOCK]

Node types: [DESCRIBE] [IT/TEST] [ACTION] [ASSERT](purple) [MOCK](dashed) [COVERAGE]

Steps:
  1. Extract all entry points from test file
  2. For each: trace which route/handler it hits (if app code provided)
     If no app code: render test flow only (describe→it→action→assert chain)
  3. Coverage gap detection (if app code also provided):
     → show routes/functions with NO test coverage (gray + ⚠️)
     → overlay coverage on architecture layer graph

Output: HTML artifact — test flow graph + coverage overlay if applicable + ASCII test map below
```

---

## REVIEW MODE

Triggered by "review" / "complexity" / "dependencies". No zoom levels.

```txt
Graphs: Dependency graph (circular deps highlighted) + Complexity heatmap (size=LOC, color=complexity)
Problems: same checklist as OVERVIEW Step 3
Output: HTML artifact, nodes clickable, problem nodes red, dead code gray,
        heatmap green→yellow→red; + static ASCII dependency map below
```

---

## Output Format

```txt
1. Graph first, always — before any prose
2. HTML artifact: plain HTML + inline SVG + vanilla JS (no framework)
   - Nodes clickable → highlight connected path
   - Hover tooltip → file path, LOC, function signature, layer
   - "Trace from here" button on every node
   - Zoom control [A][B][C][D][E] visible (TRACE MODE only)
   - ASCII panel below, synced to active zoom level
3. ASCII-only when: terminal/plain-text context, ≤8 nodes, or user asks
4. Colors (consistent per session):
   Entry/Response #3aaa8c  Internal #668db8  External/DB #F0A500
   Error #e05252  Dead/Mock #888888  Middleware #c49a3c  Assert #9b59b6
5. Legend always shown. Edge labels always visible (not hover-only).
```

---

## ASCII Flowchart (mandatory with every trace)

Portable, copy-pasteable companion to the interactive graph. Not optional.

**Symbols:**
```txt
Direction:  │ ▼ ▶ ─ ┌┐└┘ ├┤┬┴┼
Paths:      ───  happy path     ╌╌╌  SECURITY/FATAL error (+ pattern name)
            ~~~  WARNING         ─ ─  return / response
Badges:     ★ side-effect  ⟳ async  ? branch  ⊕ transform  ⚠ warning anomaly
Edge label: inline between nodes  e.g. │ { email, password }  or  ╌╌ NIL_DEREF ╌╌▶
```

**Structure (Level A):**
```
[ENTRY: trigger]
        │
        ▼ { edge label }
┌───────────────────────┐
│  FunctionName()       │ [badges]
└──────────┬────────────┘
           │              ╌╌ PATTERN ╌╌╌╌╌╌╌╌╌▶ [ERROR: description]
           ▼ { return }   ~~ PATTERN ~~~~~~~~~~▶ ⚠ warning annotation
┌───────────────────────┐
│  NextFunction()       │ [badges]
└──────────┬────────────┘
           ▼
     [RESPONSE: final output]
Legend: ─── happy path  ╌╌╌ security/fatal  ~~~ warning  ─ ─ return  ★ side-effect  ⟳ async  ⊕ transform  ⚠ anomaly
```

**ASCII rules:**
```txt
- All boxes in same column → same width (pad with spaces)
- Error/warning paths → RIGHT side with ╌╌▶ / ~~▶ + pattern name
- Conditional fork → BOTH arms labeled with condition; rejoin or terminate separately
- Side effects → ├─ ★ sideEffect() ─▶ [EXTERNAL]
- Return paths → ─ ─ ─ label inline
- Legend always at bottom
- Syncs to active zoom level in interactive graph (TRACE MODE only)
```

---

## Observation Format (below graph)

```txt
✅ [Strong point]                    max 2
⚠️  [Warning — PATTERN_NAME: detail]  one per WARNING found, max 3
🔴 [Security/Fatal — PATTERN_NAME]   one per SECURITY/FATAL found
```
Max 6 bullets. No paragraphs. Zero anomalies → omit 🔴/⚠️ entirely.

---

## Drill-Down Offer (end of every response)

```txt
TRACE:    A) Zoom Level B — per statement in [node]
          B) Error/warning detail at [flagged node]
          C) Trace another entry point: [list]
          D) Switch to REVIEW MODE

OVERVIEW: A) Trace flow from [entry point 1]   C) Zoom into subsystem [layer]
          B) Trace flow from [entry point 2]   D) View dependency graph detail

TEST:     A) Trace app code hit by [test]
          B) Coverage gaps — untested routes/functions
          C) Drill into mock boundary [MOCK]

REVIEW:   A) Trace flow from [entry point]
          B) Detail problem node: [list]
          C) View circular dependency chain
```

---

## Quick Checklist

```txt
MENTAL MODEL       symbol table built? partials marked? externals marked? dynamic edges dashed?
ANOMALY DETECTION  all 3 phases scanned? error paths only where pattern found? pattern named on edge?
MODE               correct mode selected? architecture detected and labeled?
GRAPH              graph rendered first? zoom control visible (TRACE only)? happy path solid top-to-bottom?
                   badges ★⟳⊕⚠ correct? edge labels present? legend shown?
                   OVERVIEW: auto-proceeded without stopping? entry point stated?
ASCII              present for every trace? error RIGHT with ╌╌▶+pattern? warning RIGHT with ~~▶+pattern?
                   happy path center │▼┌─┐? forks both arms labeled? side-effects as ├─★? legend at bottom?
OBSERVATIONS       ✅ max 2, ⚠️ max 3, 🔴 per pattern. zero anomalies → omit 🔴/⚠️.
DRILL-DOWN         offer at end, tailored to active mode?
```