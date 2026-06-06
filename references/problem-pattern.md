# Problem Patterns -- Code Graph Skill

Catalog of structural anti-patterns detectable without running the code.

## Detection Rules

### 1. Circular Dependency
```
Pattern: A imports B, B imports C, C imports A
Signal:  Any import cycle of any length
Render:  ASCII dashed cycle edge (<-O->) between all nodes in the cycle
Badge:   [!!] Circular
Impact:  Build failures, unpredictable initialization order
```

### 2. God File / Module
```
Pattern: Single file > 300 LOC with multiple unrelated responsibilities
Signal:  File has exports covering >3 distinct concerns
Render:  Node tagged [!!]; append (XXX LOC)
Badge:   [!!] God File (XXX LOC)
Impact:  Hard to test, violates SRP, becomes a merge conflict magnet
```

### 3. Deep Nesting
```
Pattern: Function body indented > 4 levels consistently
Signal:  Count leading whitespace of deepest line relative to function start
Render:  Node tagged [!]; append (N levels)
Badge:   [!] Deep nest (N levels)
Impact:  Cognitive load, hard to extract, usually signals missing early returns
```

### 4. Dead Code / Unused Export
```
Pattern: Symbol exported but never imported in any other file
Signal:  export X in file A, no `import { X }` or `require X` anywhere else
Render:  Node tagged [DEAD]
Badge:   [!] Unused export
Impact:  Dead weight, confuses future readers, may hide stale logic
```

### 5. Missing Error Boundary
```
Pattern: async function with no try/catch and no .catch() chain
Signal:  async keyword present, no error handling visible
Render:  Node tagged [!]
Badge:   [!] Unhandled async
Impact:  Silent failures in production
```

### 6. Implicit Global Mutation
```
Pattern: Variable defined outside function, modified inside
Signal:  `let` / `var` at module scope + assignment inside function body
Render:  Edge tagged [!] from function to module scope node
Badge:   [!] Shared mutable state
Impact:  Race conditions, testing nightmare, hidden coupling
```

### 7. Fan-Out Explosion
```
Pattern: One file imports from >8 distinct modules
Signal:  Import count > 8 at top of file
Render:  Many edges radiating from one node; node gets "hub" label
Badge:   [!] High fan-out (N imports)
Impact:  Indicates too many responsibilities in one place
```

### 8. Fan-In Collapse
```
Pattern: One utility file imported by >80% of all other files
Signal:  Incoming edge count > 0.8 * total_nodes
Render:  Node marked "Core", or "Risk" [!!] if >150 LOC
Badge:   [i] Core module (N dependents)
Impact:  Any change here ripples everywhere -- treat as infrastructure
```

---

## Severity Scale

| Badge | Meaning |
|---|---|
| [!!] Critical | Fix before shipping |
| [!] Warning | Fix in next refactor sprint |
| [i] Info | Worth knowing, not urgent |
| [OK] OK | Intentionally good pattern |

---

## Language-Specific Signals

### JavaScript / TypeScript (generic)
- `require()` in ES module context -> CommonJS/ESM mismatch risk
- `any` type used > 5 times -> type safety erosion
- `console.log` in non-debug files -> leftover debug code
- `var` in modern codebase -> hoisting bug risk, use `const`/`let`
- Nested callbacks > 3 levels -> callback hell, promisify
- `==` instead of `===` -> type coercion bugs

### Vanilla JS / HTML / CSS
- Inline `onclick=` in HTML -> mixing concerns, hard to maintain
- `document.write()` -> overwrites entire document if called after load
- Global variables on `window` -> namespace collision risk
- `!important` overuse in CSS -> specificity war, hard to override
- CSS without reset/normalize -> cross-browser inconsistency
- Synchronous `XMLHttpRequest` -> blocks UI thread
- Missing `alt` on `<img>` -> accessibility + SEO problem

### jQuery
- Chained `.ajax()` without `.fail()` -> silent network errors
- `$('#id')` inside loops -> repeated DOM query, cache the selector
- `.html(userInput)` -> XSS vulnerability
- Mixing jQuery and vanilla DOM manipulation -> unpredictable state
- `.live()` / `.die()` -> removed in jQuery 3, use `.on()`/`.off()`
- God `$(document).ready()` block -> everything in one place, untestable

### Vue 2 (Options API)
- Mutating props directly -> one-way data flow violation
- `this.$parent` access -> tight coupling between components
- Large `data()` object > 20 keys -> split into composables/child components
- Missing `key` on `v-for` -> inefficient re-render, potential state bugs
- `v-if` + `v-for` on same element -> v-for always wins, confusing precedence

### Vue 3 (Composition API)
- `reactive()` on primitives -> use `ref()` instead
- Destructuring reactive object without `toRefs()` -> loses reactivity
- Watchers without cleanup -> memory leak on unmount
- `defineExpose()` overuse -> breaks encapsulation
- Missing `onUnmounted()` cleanup for timers/subscriptions -> memory leak

### Svelte
- Large inline `<script>` block > 100 lines -> extract to .ts module
- Stores mutated directly without `update()` -> unpredictable reactivity
- Missing `onDestroy()` for subscriptions -> memory leak
- `bind:` on non-primitive -> deep binding side effects hard to trace

### SvelteKit
- Business logic in `+page.svelte` -> move to `+page.server.ts` load()
- Fetching in `onMount()` -> use `load()` for SSR compatibility
- Missing `handle` hook for auth -> unprotected routes

### Angular
- Logic in template expressions -> hard to test, move to component class
- Direct DOM manipulation via `ElementRef` -> bypasses change detection
- Missing `async` pipe -> manual subscription without unsubscribe = leak
- `ngOnChanges` without `SimpleChanges` type -> brittle change detection

### React (if target codebase)
- `useEffect` missing dependency array -> infinite render loop
- State mutation directly (`state.x = 1`) -> bypasses re-render
- Key on array index -> unstable keys, broken animation/focus
- `useEffect` doing data fetch without cleanup -> race condition on unmount
- Prop drilling > 3 levels -> use Context or state manager

### Alpine.js
- Business logic in `x-data` inline string -> extract to `Alpine.data()`
- Missing `$destroy` cleanup for watchers -> memory leak
- `x-html` with user content -> XSS risk

### HTMX
- No `hx-boost` on full page -> missing progressive enhancement
- Missing CSRF token in `hx-headers` -> security vulnerability
- `hx-swap: outerHTML` on elements with listeners -> event listeners lost

### Python
- `import *` -> namespace pollution
- Bare `except:` -> swallows all exceptions
- Mutable default arguments (`def f(x=[]):`) -> classic bug
- `time.sleep()` in async function -> blocks event loop, use `asyncio.sleep()`

### PHP / Laravel
- Business logic in Controller -> should be in Service/Action
- Raw `$_GET`/`$_POST` without validation -> security smell
- `dd()` / `var_dump()` in committed code -> debug leftover
- N+1 query in loop without eager loading -> performance problem
- Missing `->authorize()` in FormRequest -> unprotected endpoint

### Go
- `err` ignored with `_` -> silent failure
- Goroutine without WaitGroup or channel -> leak risk
- `interface{}` / `any` overuse -> loses type safety

### Kotlin / Android
- `!!` operator -> NullPointerException waiting to happen
- `lateinit var` without initialization check -> crash risk
- Network call on main thread -> ANR (Application Not Responding)
- Missing `lifecycleScope` for coroutines -> leak on fragment destroy

### Ruby / Rails
- Logic in views (`.erb`) -> move to helper or presenter
- `User.all` without pagination -> memory explosion on large tables
- Missing `before_action` for auth -> unprotected controller actions

### CSS (any framework)
- Specificity > 0-3-0 -> hard to override without `!important`
- Magic numbers without variables/custom properties -> unmaintainable
- Unused CSS selectors > 20% of stylesheet -> dead weight