# Mental Model Reference

Detailed schema and rendering rules for Phase 0 (internal symbol table)
used by the Code Graph skill before rendering any graph.

---

## Symbol Table Schema

### FileSymbol
```txt
{
  path:      string          // relative path, e.g. "app/Services/AuthService.php"
  language:  string          // "php" | "ts" | "py" | "go" | "kt" | ...
  loc:       number          // lines of code
  imports:   ImportSymbol[]
  exports:   string[]        // exported names
  partial:   boolean         // true if file was truncated / incomplete
}
```

### ImportSymbol
```txt
{
  source:    string          // module path or package name
  names:     string[]        // what was imported ([] = wildcard / default)
  is_external: boolean       // true if from node_modules / vendor / stdlib
}
```

### FunctionSymbol
```txt
{
  name:        string
  file:        string
  line_start:  number
  line_end:    number
  kind:        "function" | "method" | "arrow" | "closure" | "lambda"
  class_name:  string?       // for methods
  params:      ParamSymbol[]
  returns:     string?       // inferred return type, ~unknown if unclear
  calls:       CallEdge[]
  reads:       string[]      // outer-scope variables read
  writes:      string[]      // variables assigned or mutated
  is_async:    boolean
  throws:      string[]      // error types thrown (inferred from throw/raise)
  side_effects: SideEffect[]
  partial:     boolean       // true if body was truncated
}
```

### ParamSymbol
```txt
{
  name:     string
  type:     string?          // ~unknown if not annotated
  default:  string?          // default value if any
  position: number           // 0-indexed
}
```

### CallEdge
```txt
{
  target:      string        // function name being called
  target_file: string?       // resolved file if deterministic
  args_shape:  string        // e.g. "(email, password)" or "(~dynamic)"
  line:        number
  is_await:    boolean
  is_dynamic:  boolean       // dynamic dispatch — target unknown at parse time
  is_external: boolean       // calling into external library
}
```

### SideEffect
```txt
{
  kind: "db.read" | "db.write" | "http.call" | "file.read" | "file.write"
      | "event.emit" | "cache.read" | "cache.write" | "email.send"
      | "queue.push" | "log.write" | "unknown"
  target: string?            // e.g. "users table", "S3", "EventBus"
  line:   number
}
```

### PayloadSymbol  (one per entry point -- drives Payload / Contract Flow graph)
```txt
{
  entry:          string         // entry point id, e.g. "POST /users"
  request_shape:  {              // where the inbound shape comes from
    body:    string?             // ~{ name, email, password } or ~unknown
    query:   string?
    params:  string?
    headers: string?
  }
  validated:      boolean        // true if a schema validator/DTO/permit guards it
  hops:           PayloadHop[]   // shape at each hop, in order, to the terminal
  response_shape: string?        // actual shape at [RESPONSE]
  declared_response_type: string? // DTO/interface/struct the response claims to be
  sensitive_fields: string[]     // fields matching password/*hash/token/secret/ssn/card/*_key
}
```

### PayloadHop
```txt
{
  node:    string                // node id this shape exists at (controller/service/persist/response)
  shape:   string                // shape here, ~unknown if not visible
  diff:    {                     // change vs previous hop
    added:   string[]            // +field
    dropped: string[]            // -field
    renamed: string[]            // ~old->new
    retyped: string[]            // field: T1=>T2
  }
  is_persist:  boolean           // hop writes to DB/store (drives OVERPOSTING check)
  is_response: boolean           // terminal response hop (drives LEAKED_FIELD / CONTRACT_BREAK)
}
```

**Payload shape extraction (where the initial shape + validation come from per stack)**
is tabulated in `references/payload-flow.md` (Express req.body, NestJS DTO/@Body, FastAPI
Pydantic, Laravel FormRequest, Go json.Decode struct, Rails strong params, ...). Build
`request_shape` and `validated` from those signals; never fabricate fields not in the code.

### VariableSymbol
```txt
{
  name:          string
  file:          string
  line:          number
  declared_in:   string      // function name or "module_scope"
  kind:          "const" | "let" | "var" | "param" | "property" | "unknown"
  assigned_from: string      // right-hand side expression text (max 80 chars)
  type_inferred: string?     // ~unknown if unclear
  read_by:       string[]    // function names that read this variable
  mutated_by:    string[]    // function names that assign to this variable
  is_param:      boolean
  is_transform:  boolean     // true if assigned_from changes data shape
                             // e.g. mapper.toDTO(), JSON.parse(), array.map()
                             // drives ⊕ badge at Level D
}
```

### ExpressionSymbol  (built on demand at Level C+)
```txt
{
  text:           string     // exact source text, max 120 chars
  kind:           "call" | "await" | "access" | "assign" | "return"
                | "throw" | "ternary" | "binary" | "literal"
  parent_fn:      string     // containing function name
  line:           number
  input_symbols:  string[]   // variable/param names read by this expression
  output_symbol:  string?    // variable name this expression is assigned to
  output_type:    string?    // ~unknown if unclear
  side_effects:   SideEffect[]  // populated if expression itself causes side effect
                                // e.g. await db.save(), fetch(), fs.writeFile()
                                // drives ★ badge at Level C/D/E
  is_async:       boolean    // true if expression contains await or returns Promise
                             // drives ⟳ badge at Level C/D/E
}
```

---

## Extraction Heuristics by Language/Framework

### TypeScript / JavaScript (generic)
```txt
Functions:   function f(), const f = () => {}, class C { method() {} }
Imports:     import X from '...', import { X } from '...', require('...')
Exports:     export const, export default, module.exports
Async:       async keyword, .then(), new Promise()
Side effects: await prisma.*() → db, await axios.*() → http,
              fs.readFile → file.read, EventEmitter.emit → event.emit
Variables:   const/let/var declarations, destructuring
```

### Vanilla JS / HTML / CSS
```txt
Entry points: DOMContentLoaded, window.onload, <script> tag order
Functions:    function declarations, const fn = function(), arrow fn
Events:       addEventListener(event, handler), onclick=, oninput=
DOM reads:    querySelector, getElementById, getElementsBy*
DOM writes:   innerHTML, textContent, appendChild, classList.*
Side effects: fetch(), XMLHttpRequest, localStorage.*, sessionStorage.*
Async:        fetch().then(), async/await, setTimeout, setInterval
CSS signals:  @import, :root variables, media queries, keyframes
```

### jQuery
```txt
Entry points: $(document).ready(fn), $(fn), $(window).on('load')
Functions:    named functions passed to .on(), $.fn.pluginName
Events:       .on(event, selector, handler), .click(), .submit(), .change()
AJAX:         $.ajax(), $.get(), $.post(), $.getJSON()
DOM reads:    $('selector').val(), .text(), .attr(), .data()
DOM writes:   .html(), .append(), .prepend(), .css(), .addClass()
Side effects: $.ajax → http.call, .trigger() → event.emit
```

### Vue 2 (Options API)
```txt
Entry points: new Vue({ el, data, mounted }), Vue.component()
Functions:    methods: { fn() {} }, computed: { x() {} }, watch: { x(val) {} }
Lifecycle:    created, mounted, beforeDestroy
Events:       this.$emit('event', payload), v-on:event / @event
Data flow:    props → data → computed → template
Side effects: this.$http.* / axios.* → http.call, Vuex commit/dispatch → state
Async:        async methods, axios calls in mounted/created
```

### Vue 3 (Composition API)
```txt
Entry points: createApp(), defineComponent(), setup()
Functions:    const fn = () => {} inside setup(), composables (use*())
Reactive:     ref(), reactive(), computed(), readonly()
Lifecycle:    onMounted(), onUnmounted(), onBeforeMount()
Events:       emit('event', payload), defineEmits()
Watch:        watch(source, cb), watchEffect(cb)
Side effects: axios.* → http.call, store.commit/dispatch → state
Async:        async setup(), useFetch(), useAsyncData()
```

### Svelte
```txt
Entry points: .svelte file — script block runs on mount
Functions:    function declarations, arrow fns in <script>
Reactive:     $: label = reactive declaration
Stores:       import { writable, readable, derived } from 'svelte/store'
              $store auto-subscription in template
Lifecycle:    onMount(), onDestroy(), beforeUpdate(), afterUpdate()
Events:       on:click={handler}, createEventDispatcher(), dispatch()
Side effects: fetch → http.call, store.update/set → state.write
Async:        async function + await inside onMount or event handler
```

### SvelteKit
```txt
Entry points: +page.svelte, +layout.svelte, +page.server.ts, +server.ts
Load fn:      export async function load({ fetch, params })
Form actions: export const actions = { default: async ({ request }) }
API routes:   export async function GET/POST/PUT/DELETE({ request })
Side effects: same as Svelte + server-side DB calls in load()
```

### Angular
```txt
Entry points: @NgModule bootstrap, AppComponent, routing module
Functions:    component methods, service methods, pipe transforms
DI:           constructor(private svc: ServiceName)
Lifecycle:    ngOnInit(), ngOnDestroy(), ngOnChanges()
Events:       (click)="handler()", EventEmitter, @Output
Observables:  Observable, Subject, BehaviorSubject, async pipe
Side effects: HttpClient.get/post → http.call, Router.navigate → nav
Async:        Observable chains, async/await in services
```

### Alpine.js
```txt
Entry points: x-data on root element, Alpine.data() registration, Alpine.start()
Functions:    methods inside x-data object, Alpine.data('name', () => ({}))
Events:       x-on:click / @click, $dispatch('event')
Reactive:     x-bind:attr, x-model (two-way), $watch('prop', callback)
Side effects: fetch inside x-data methods → http.call
Async:        async methods in x-data, await fetch()
Magic props:  $el, $refs, $store, $dispatch, $nextTick, $watch
```

### HTMX
```txt
Entry points: hx-get/post/put/delete attributes on HTML elements
Triggers:     hx-trigger (click, submit, load, revealed, etc.)
Targets:      hx-target (CSS selector for where response goes)
Swap:         hx-swap (innerHTML, outerHTML, beforeend, etc.)
Side effects: every hx-* request → http.call → DOM swap
Events:       htmx:afterRequest, htmx:beforeRequest (JS event listeners)
Extensions:   hx-ext (ws for websockets, sse for server-sent events)
Note:         no JS functions to trace — trace HTTP endpoints instead
```

### PHP / Laravel
```txt
Functions:    function f(), public function f() in class
Imports:      use Namespace\Class, require/include
Exports:      public methods = implicit export
Async:        none native (mark queue jobs as async boundary)
Side effects: DB::, Model::, Mail::, Event::, Storage::, Http::
Variables:    $varName declarations and assignments
Routes:       routes/web.php, routes/api.php → Route::get/post/put/delete
```

### Python
```txt
Functions:    def f(), async def f(), lambda
Imports:      import X, from X import Y
Exports:      __all__, top-level names
Async:        async def, await, asyncio
Side effects: db.session.*, requests.*, open(), emit(), send()
Variables:    assignments (no declaration keyword)
```

### Go
```txt
Functions:    func f(), func (r Receiver) Method()
Imports:      import blocks
Exports:      Capitalized names
Async:        goroutines (go func()), channels
Side effects: db.Exec/Query, http.*, os.*, json.Marshal
Variables:    var x, x := assignment
```

### Kotlin
```txt
Functions:    fun f(), suspend fun f()
Imports:      import statements
Exports:      public top-level declarations
Async:        suspend, coroutineScope, launch, async
Side effects: db.*, retrofit.*, File(), EventBus.*
Variables:    val (immutable), var (mutable)
```

### Ruby / Rails
```txt
Functions:    def method_name, lambda, proc, block { }
Imports:      require, require_relative, include Module
Async:        Sidekiq jobs, ActiveJob, async gem
Side effects: ActiveRecord queries, ActionMailer, ActionCable
Variables:    local (lowercase), instance (@var), class (@@var)
Routes:       config/routes.rb → resources, get/post
```

---

## Zoom Level Rendering — Node Shape Guide

```txt
Level A (Function)
  Shape:    rounded rectangle, width ~120px
  Label:    functionName()   or   ClassName.method()
  Subtext:  file path, shortened
  Size:     scales with LOC (min 40px height, max 80px)
  Tooltip:  params list, return type, side effects, call count

Level B (Statement)
  Shape:    rectangle, width ~180px, height 36px fixed
  Label:    truncated statement text (max 40 chars, "..." if longer)
  Color:    inherits from parent function node type
  Tooltip:  full statement, line number, variables involved

Level C (Expression)
  Shape:    pill/capsule, compact
  Label:    expression text (max 30 chars)
  Color:    by kind:
              call    → blue
              await   → blue + ⟳
              assign  → green
              return  → green (darker)
              throw   → red
              access  → gray
  Tooltip:  full expression, input → output

Level D (Variable)
  Shape:    diamond (shows it's a value, not an action)
  Label:    varName
  Subtext:  ~type
  Color:    
              const/param → teal
              let/mutable → amber
              module-scope → gold
  Tooltip:  assigned from, read by, mutated by, declared at line

Level E (Property)
  Shape:    small circle (smallest unit)
  Label:    obj.prop  or  obj?.prop
  Color:    same as parent variable
  Tooltip:  access at line, inferred type, parent object
```

---

## Mixed-Level View Rules

When user drills into ONE node but keeps rest at Level A:

```txt
- Expanded node: render at Level B or C (one level deeper than current)
- Surrounding nodes: stay at Level A
- Edges between levels: use transition edge style
    → thick on Level A side, thin on Level B side
    → label shows "entry point" for drill boundary
- Collapsed indicator: nodes with drillable content show "▼ N symbols" badge
```

Example:
```
[AuthService.login()]  Level A
        │
        ▼ drilled ──────────────────────────────────┐
        │                                            │ Level B
        │  [const user = await repo.find(email)]     │
        │           │                                │
        │  [if (!user) throw 401]                    │
        │           │                                │
        │  [const match = await bcrypt.compare()]    │
        │           │                                │
        │  [return { user, token }]                  │
        └────────────────────────────────────────────┘
        │
        ▼  Level A resumes
[TokenService.sign()]
```

---

## Partial File Handling

When a file is truncated or only partially provided:

```txt
1. Extract what's visible — do not fabricate what's not
2. Mark all symbols from that file as partial: true
3. In graph: render with dashed border + ⋯ indicator
4. In tooltip: "Extracted from partial file — may be incomplete"
5. Edges going INTO or OUT OF partial symbols: dashed gray
6. Do not infer side effects or return types for partial functions
7. In observation bullets: note which symbols are partial
```

---

## Recursive Function Handling

```txt
- Detect: function A calls function A (direct recursion)
         or A→B→A (indirect recursion)
- Render: self-loop arrow on the node
- Label on self-loop: "recursive" + base condition if detectable
         e.g., "recursive (base: n === 0)"
- Indirect recursion: show cycle with all nodes in cycle,
         mark cycle edges in orange
- Do not expand recursive calls infinitely — cap at 1 level deep
```

---

## Dynamic Dispatch Handling

```txt
Signals:
  JavaScript:  obj[methodName](), eval(), require(variable)
  PHP:         $class->$method(), call_user_func()
  Python:      getattr(obj, name)(), __getattribute__
  Go:          interface dispatch (mark as [INTERFACE] not [DYNAMIC] if type-safe)

Render:
  - Edge drawn as dashed gray
  - Target node: [DYNAMIC: ~unknown target]
  - Label: "dynamic call — target resolved at runtime"
  - Do not attempt to trace further
  - In observation: flag as ⚠️ — harder to statically analyze
```

---

## Symbol Table — React State Shape

For the interactive graph UI, the symbol table drives these state structures:

```js
// Built once in Phase 0, drives all graph renders
const symbolTable = {
  files: Map<path, FileSymbol>,
  functions: Map<name, FunctionSymbol>,
  variables: Map<`${file}:${name}`, VariableSymbol>,
  expressions: Map<`${file}:${line}`, ExpressionSymbol>,  // built on Level C+ demand
  callGraph: Map<callerName, CallEdge[]>,    // who calls whom
  reverseCallGraph: Map<calleeName, string[]>, // who is called by whom
  architecture: {
    pattern: string,
    layers: Map<layerName, string[]>,  // layer → [file paths]
  }
};

// Zoom state per trace
const traceState = {
  entryPoint: string,         // starting node id
  activeLevel: "A"|"B"|"C"|"D"|"E",
  expandedNodes: Set<string>, // nodes drilled deeper
  selectedNode: string|null,
};
```