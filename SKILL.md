---
name: code-graph
description: >
  Visual codebase review and structural analysis. Follows the edges -- every call,
  data pass, and import -- from each entry point to cover all connected, reachable
  code in one pass (nothing skipped, dead code flagged). Produces a reachability/edge
  graph, dependency graphs, module/architecture maps, complexity heatmaps, and
  structural problem detection (circular deps, god files, dead code, misplaced logic,
  missing error boundaries) as ASCII-only directed graphs. Use this skill whenever
  the user shares code, a file, or a folder and wants to understand or review
  its structure end-to-end: "review my code", "analyze codebase", "show dependencies",
  "map the modules", "where is the complexity", "find structural problems", "what
  touches this", "end-to-end analysis", "is this well organized", "architecture
  review", "dependency graph", "complexity heatmap", "request/response shape",
  "payload flow", "data contract", "where does this field come from", "what shape
  comes back", "is a field leaking". Always output an ASCII graph first.
  Never explain structure in prose alone.
---

# Code Graph Skill

Review codebase structure visually -- dependencies, layering, complexity, and structural problems.
Graph first, always. Prose is secondary. There is one mode: REVIEW. No execution tracing, no zoom levels.

## Operating principle -- total, in one command

One command = one complete review. Deliver the FULL analysis in this single response:
every file, every entry point, every reachable node, every edge, all problems -- done.
No staging, no "phase 1 of N", no "want me to continue?", no waiting for a second prompt,
no drill-down menu. The user should never have to ask twice to get the whole picture.

If the codebase is genuinely too large to fit one response, do NOT silently truncate or defer.
Cover as much as possible, then state plainly and specifically what was not covered and why
(e.g. "files X, Y not analyzed -- exceeded response limit"). Honesty about a hard limit is allowed;
a voluntary partial answer with an offer to continue is not.

> **IMPORTANT -- trace 100% end to end.**
> Follow every edge from each entry point all the way to its terminal (response / return /
> DB-file-cache write / emitted event-job-webhook-email / handled error). Do not stop at the
> first handler, service, or repository. Every reachable node -- every function, method, helper,
> mapper, serializer, branch, and side effect that the provided code can hit -- MUST appear and be
> analyzed. Both arms of every reachable branch. One iteration plus exit for loops. Self-loop plus
> base case for recursion. 100% means zero "continues to ...", zero summarized branches whose code
> was provided, zero skipped paths. The only nodes allowed to be incomplete are [PARTIAL] (code not
> provided) and [EXTERNAL] (third-party internals). An incomplete trace is worse than none: it hides
> the exact edge where the bug lives and gives false confidence that the path was checked.

---

## Phase 0 -- Mental Model (runs before every graph, silent)

Build a symbol table from all provided files. Never show it raw -- it only drives rendering.

```txt
FILE:       path, language, LOC, imports[], exports[]
FUNCTION:   name, file, line_start/end, params, returns, calls[], reads[], writes[],
            side_effects[], throws[]
VARIABLE:   name, file, line, declared_in, assigned_from, type_inferred, read_by[], mutated_by[]
```

**Build rules:**
```txt
- Partial/truncated file -> mark symbols [PARTIAL], never guess
- Unknown type -> prefix "~unknown", never fabricate
- External lib -> [EXTERNAL] node, do not inspect internals
- Dynamic dispatch -> [DYNAMIC] edge, dashed
- Recursive call -> self-loop, label depth limit if known
```

---

## REVIEW MODE (the only mode)

Always review. Given any code, file, or folder, render the structure as graphs -- never as prose first.

**Produce, in this order:**
```txt
1. Reachability / edge graph -- follow the edges, hit everything connected.
   Nodes = functions/methods (and the files that hold them). Edges = calls, data passes,
   imports, and side-effect writes (DB/file/queue/event). Label each edge with what moves
   across it (params, return type, payload), e.g. "{ email, password }", "User | null".
   From every entry point, walk EVERY in-repo edge to every connected node:
     handler -> middleware/guard -> service -> repository -> model -> DB/store -> response,
     plus every helper, mapper, serializer, and constructor touched along the way.
   Every node that is reachable (can be hit) must appear and be analyzed -- nothing skipped.
   Nodes with no in-repo caller and not an entry point -> mark UNREACHABLE / dead with [DEAD].
   External libraries are [EXTERNAL] terminals; do not walk into their internals.
2. Payload / Contract Flow -- one payload column per ROUTE/handler/command/consumer
                       (NOT per binary/process), from the request shape at the entry to the
                       response shape at the terminal. Payload lives at route altitude: if graph 1
                       grouped many routes under one coarse node (e.g. "the API binary"), drill
                       down here and enumerate the concrete routes (METHOD + path) with their
                       shapes -- never collapse payloads to the process level. At each hop show the
                       field-level DIFF: +field (added)  -field (dropped)  ~old->new (renamed)
                       field: T1=>T2 (retyped)  [SENSITIVE] (secret field). Flag sensitive
                       fields that survive to the response, request bodies persisted without a
                       whitelist, payloads consumed without schema validation, and responses that
                       diverge from the declared DTO/return type. See `references/payload-flow.md`.
3. Dependency graph  -- nodes = files/modules, edges = imports.
                       Circular dependency cycles highlighted with ASCII dashed edges + [!!] Circular badge.
                       Dead exports (never imported in-repo) -> [DEAD] node.
4. Complexity heatmap -- node size = LOC, risk label = [low] [medium] [high].
                       God files (>300 LOC) and deeply nested functions (>4 levels) flagged [high].
5. Architecture/layer map -- ONLY when a framework/arch is detected (see Architecture Detection).
                       Group nodes by layer; flag business logic sitting in the wrong layer.
```

**Structural problem checklist (run inline as observations):**
```txt
[ ] Circular imports            [ ] God files (>300 LOC)
[ ] Business logic in wrong layer   [ ] Missing error boundaries
[ ] Dead exports                [ ] Deep nesting (>4 levels)
[ ] Leaked field in response    [ ] Over-posting / mass-assign
[ ] Contract break (resp vs DTO)    [ ] Unvalidated payload
```
Plus the full pattern catalog below.

**Completeness contract -- full edge coverage in ONE response:**
```txt
Cover ALL provided code in a single pass. Never stop at the first file/module/entry point.
- From every entry point, follow EVERY in-repo edge until each branch reaches a terminal
  (response / return / DB-file-cache write / emitted event-job-webhook-email / handled error).
- Every reachable node (every piece of code that can be hit) must be analyzed and rendered --
  no "continues to ...", no summarizing a branch whose code was provided.
- The dependency graph includes every file and every import edge -- not a sample.
- The complexity heatmap ranks every module.
- If multiple entry points exist (routes, main, CLI, workers), walk all of them.
- For conditional branches, cover both arms if both are reachable. For loops, one iteration
  plus exit condition. For recursion, self-loop plus base case.
Do NOT defer modules, entry points, or branches to a follow-up. Do NOT reply with a partial
view plus an offer to "drill into the rest". A partial answer is a failed review.
Only mark a node [PARTIAL]/[EXTERNAL] when the code for it was genuinely not provided.
```

---

## Structural Problem Detection (run before rendering, evidence-based only)

Scan the symbol table for patterns. Flag a node/edge ONLY where a pattern is actually found.
No pattern -> no annotation. State the pattern name explicitly on every flagged node.

**SECURITY + FATAL -> [!!] node (named pattern)**
```txt
SECURITY:
  SQL_INJECTION       raw user input into DB query, no parameterize/ORM
  AUTH_BYPASS         handler reachable without required auth guard
  EXPOSED_SECRET      API key / token hardcoded or passed to logger
  UNVALIDATED_INPUT   req.body / args -> service/DB with no schema check
  IDOR                resource fetched by user-supplied ID, no ownership check
  LEAKED_FIELD        sensitive field (password/*hash/token/secret) survives to response payload
  OVERPOSTING         whole req.body persisted with no whitelist -- privileged field can be set

FATAL:
  NIL_DEREF           nullable accessed before nil/null check
  UNHANDLED_PROMISE   async call, no await AND no .catch() -- silent crash
  MISSING_RETURN      typed function has code path with no return value
  DEADLOCK_RISK       mutex locked, no guaranteed unlock / circular goroutine wait
  INFINITE_LOOP       loop condition cannot become false given data in scope
```

**WARNING -> [!] node annotation (named pattern)**
```txt
NULL_UNGUARDED      returns T | null, consumer accesses without guard
SWALLOWED_ERROR     error caught but not propagated or logged
MISSING_AWAIT       async called without await, result is Promise<T> not T
SHAPE_DRIFT         object transformed, downstream still reads pre-transform fields
TYPE_MISMATCH       upstream produces type X, downstream expects type Y
KEY_MISMATCH        publish key and consumer listen key differ (typo / case / prefix)
UNCLOSED_RESOURCE   DB connection / file handle opened, no close/defer path
CONTRACT_BREAK      response shape != declared DTO/return type (missing/extra/typo field)
UNVALIDATED_PAYLOAD payload consumed/persisted with no schema validation (shape-level)
```

See `references/problem-pattern.md` for the full structural anti-pattern catalog and render rules.

---

## Architecture Detection

Detect architecture pattern AND framework/stack to label and group nodes in the dependency/layer map.
Show in graph title, e.g. `Review -- NestJS Clean Arch`. The arrows below are layer ordering for grouping, not execution traces.

**Backend:**
```txt
Laravel MVC    routes/web.php, routes/api.php, Http/Controllers/  -> Route/Controller/Model/DB
Express/Koa    router.get/post, app.use(), req/res params          -> Router/Middleware/Handler
Fastify        fastify.register(), reply.send()                    -> Plugin/Route/Handler
NestJS         @Module @Controller @Injectable @InjectRepository   -> Module/Controller/Service/Repo
Django         urls.py, views.py, serializers.py, models.py        -> URL/View/Serializer/Model/DB
FastAPI        @app.get/post, Depends(), async def, SQLAlchemy     -> Router/Endpoint/Dependency/DB
Go net/http    http.HandleFunc, ServeHTTP                          -> Mux/Handler/Service/DB
Go Gin/Echo    gin.Default(), r.GET/POST, c.JSON()                 -> Engine/RouteGroup/Handler
Rails          routes.rb, *_controller.rb, ApplicationRecord       -> Route/Controller/Model/DB
Spring Boot    @RestController @Service @Repository @Autowired      -> Controller/Service/Repo/DB
Generic MVC    controllers/, models/, views/ or *Controller.*      -> Model/View/Controller
Clean Arch     domain/, application/, infrastructure/ or *UseCase  -> Entity/UseCase/Interface/Infra
Hexagonal      ports/, adapters/, core/                            -> Core/Port/Adapter
Layered        services/, repositories/, handlers/                 -> Presentation/Business/Data
```

**Frontend:**
```txt
Vanilla JS  addEventListener, querySelector -- no framework   React     .jsx/.tsx, useState, useEffect
jQuery      $(fn), .on(), $.ajax                              Next.js   pages/api/, app/api/, route.ts
Vue 2/3     new Vue() / setup(), ref(), reactive()            Astro     .astro, ---frontmatter---
Nuxt.js     pages/, useAsyncData(), useFetch()                Alpine.js x-data, x-on, x-bind, x-model
Svelte(Kit) .svelte, $: reactive, +page.server.ts            HTMX      hx-get/post, hx-trigger, hx-swap
Angular     @NgModule @Component @Injectable, ngOnInit()
```

**Mobile:**
```txt
React Native  .tsx + react-native      Flutter        .dart, StatefulWidget, build()
Kotlin Android .kt, ViewModel, Flow    Swift iOS      .swift, viewDidLoad, URLSession
```

**Tiebreaker:** framework-specific signals > generic folder naming > file naming > most matches > tied -> `Layered (detected)`.
**Unknown fallback:** folder/file heuristics + import patterns -> label modules `[INPUT]/[PROCESS]/[OUTPUT]`.

---

## Output Format

```txt
1. Graph first, always -- before any prose.
2. ASCII ONLY. Render the full review in ASCII in every context -- terminal, chat, or agent.
   Never generate HTML, SVG, JavaScript, interactive artifacts, clickable graphs, or visual
   artifacts, even if the user explicitly asks for them. If asked for HTML/interactive output,
   refuse that format briefly and provide the ASCII graph instead.
3. Render in this order: reachability/edge graph, payload/contract flow, dependency graph,
   complexity heatmap, architecture/layer map when detected, then problems and observations.
4. Severity is shown with ASCII badges only ([!!] fatal, [!] warning, [DEAD] dead, [OK] safe).
   No colors, no Unicode box-drawing, no emoji -- 7-bit ASCII characters only.
```

---

## ASCII Reachability Map (the default primary output)

This is the main deliverable, not a companion. Render the FULL review here in ASCII: every entry
point traced through every edge to its terminal, every reachable node shown, problems flagged inline.

**Symbols (pure ASCII only -- never use Unicode box-drawing or emoji):**
```txt
Edges:   -->  call / depends on        <-O->  circular dependency
Flow:    |  vertical edge    v  arrowhead    + - corners & connectors    label = data passed
Badges:  [!!] security/fatal   [!] warning   * side-effect   (+) god file (>300 LOC)   [DEAD] dead/unreachable   [OK] safe
Nodes:   [RESPONSE] terminal   [EXTERNAL] third-party   [PARTIAL] code not provided
```

**Structure -- boxed, entry at top, terminal at bottom, one lane per entry point:**
```
  ENTRY /login          ENTRY /users/:id --auth-->    ENTRY /admin/reset [!!] AUTH_BYPASS
       | {email,pwd}          | id                          |
       v                      v                            v
 +--------------+      +--------------+             +--------------+
 | controller   |      | controller   |             | controller   |
 +------+-------+      +------+-------+             +------+-------+
        v                    v                            v
 +--------------+<-O->+--------------+             +--------------+
 | service *    |     | service [!]  |             | service      |
 +------+-------+     +------+-------+             +------+-------+
        +-------------------+--------------------------+
                            v
              +-----------------------------+
              | repository [!!] SQL_INJECTION |
              +--------------+--------------+
                             v
                      [EXTERNAL: db]  -->  [RESPONSE]

 DISCONNECTED:  utils.legacyFormat()  [DEAD] UNREACHABLE -- no caller, not an entry point

 Legend: --> call  <-O-> circular  * side-effect  [!] warning  [!!] security  [DEAD] dead  label = data
```

**ASCII rules:**
```txt
- One lane (vertical column) per entry point; trace each lane top->terminal. Lanes may merge at shared nodes.
- Every reachable node appears in a box. Nothing summarized as "continues to ..." if code was provided.
- Label edges with the data that moves. Flag nodes inline with badge + named pattern.
- Circular deps -> <-O-> between the two nodes. Dead/unreachable nodes listed under DISCONNECTED with [DEAD].
- Terminals explicit: [RESPONSE] / [EXTERNAL] / final write. Legend always at bottom.
- Boxes in the same column share width (pad with spaces). Keep it copy-paste clean.
```

---

## Observation Format (below graph)

```txt
[OK] [Strong point]                     max 2
[!]  [Warning -- PATTERN_NAME: detail]   one per WARNING found, max 3
[!!] [Security/Fatal -- PATTERN_NAME]    one per SECURITY/FATAL found
```
Max 6 bullets. No paragraphs. Zero problems found -> omit [!!]/[!] entirely.

---

## Ending the response -- STOP after observations

By the completeness contract, the review already covers the entire codebase. There is nothing
left to defer, so there is nothing to offer.

After the graph(s), the ASCII map, and the observation bullets: **STOP. End the response.**

Do NOT append any of the following -- they all imply the review was partial, which it is not:
```txt
x a drill-down menu (A/B/C/D options)
x "want me to go deeper / drill into X?"
x a list of other entry points / modules "to trace next"
x "switch to ... mode" / "I can also ..."
x any closing question that offers further analysis
```
If the user wants more, they will ask. The correct ending is the last observation bullet.

---

## Quick Checklist

```txt
MENTAL MODEL   symbol table built? partials marked? externals marked? dynamic edges dashed?
PROBLEMS       all patterns scanned? flagged ONLY where evidence found? pattern named on node?
ARCHITECTURE   framework/arch detected and labeled? nodes grouped by layer?
GRAPH          ASCII reachability graph rendered first? NO HTML/SVG/JS ever? lanes per entry point?
               circular deps badged [!!]? dead code badged [DEAD]? legend shown? edge labels visible?
PAYLOAD        payload/contract flow rendered per entry? field diffs per hop (+/-/~/T1=>T2)?
               sensitive fields tagged? LEAKED_FIELD / OVERPOSTING / CONTRACT_BREAK / UNVALIDATED_PAYLOAD flagged on evidence?
ASCII          one lane per entry point? every reachable node boxed? terminals explicit? legend at bottom?
OBSERVATIONS   [OK] max 2, [!] max 3, [!!] per pattern. zero problems -> omit [!!]/[!].
COMPLETENESS   ALL files/modules/entry points covered in one pass? nothing deferred to follow-up?
EDGE COVERAGE  every edge followed? every reachable node hit & analyzed? unreachable flagged dead?
ENDING         response stops after observations? NO drill-down menu, offers, or follow-up questions?
```