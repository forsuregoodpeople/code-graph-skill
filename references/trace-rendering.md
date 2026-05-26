# Trace Rendering Reference

Detail rules untuk render execution trace graph di Code Graph skill.

---

## Happy Path vs Error Path

```txt
Happy path  → solid edges, normal colors
Error path  → dashed red edges, [ERROR] node at terminus

Always render both in same graph. Do not hide error path.
```

```
[ENTRY: POST /login]
        │
        ▼ { email, password }
[MIDDLEWARE: AuthMiddleware]
        │                    ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌▶ [ERROR: 429 Rate Limited]
        ▼ req.body validated
[CONTROLLER: AuthController.login()]
        │                    ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌▶ [ERROR: 422 Validation]
        ▼ { email, password }
[SERVICE: AuthService.authenticate()] ★⟳
        │                    ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌▶ [ERROR: 401 Invalid creds]
        ▼ email
[REPOSITORY: UserRepository.findByEmail()] ★
        │
        ▼ User | null
[SERVICE: AuthService — bcrypt.compare()] ⊕
        │
        ▼ { user, token }
[RESPONSE: 200 { token, user }]
```

---

## Edge Label Rules

Every edge in a trace graph must be labeled. Format:

```txt
Short type hint  →  { shape }   or   primitiveType: value_hint

Examples:
  → { userId: string, role: string }
  → string | null
  → Error: 401
  → void  (for fire-and-forget)
  → Promise<User>  (async, add ⟳ on edge)
  → event: 'user.created'  (for event emitters)
```

When exact type unknown → infer from context, prefix with `~`:
```txt
  → ~{ user object }
  → ~array of results
```

---

## Node Badge Rules

```txt
★  Side effect
     → node writes to DB, calls external API, writes file, emits event, sends email
     → place on the node that performs the side effect
     → NOT on nodes that just read/query

⟳  Async boundary
     → function is async / returns Promise / uses await
     → place on BOTH the node and the edge going into it

⊕  Data transform
     → node changes the shape/structure of data (not just passes through)
     → e.g. serializer, mapper, formatter, reducer
     → if both async AND transform apply: ⟳⊕
     → if side effect AND async: ★⟳
     → if all three: ★⟳⊕

?  Conditional branch
     → node has if/switch that sends execution different directions
     → show as fork: two edges out, each labeled with condition
```

---

## Conditional Branch Rendering

```txt
[SERVICE: OrderService.create()]  ?
        │
        ├─ [user.role === 'admin'] ──▶ [SERVICE: applyAdminDiscount()]
        │                                      │
        └─ [user.role !== 'admin'] ─────────────┘
                                               │
                                               ▼
                                    [REPOSITORY: OrderRepository.save()] ★
```

Do not collapse branches. Always show the fork explicitly.

---

## Async Chain Rendering

```txt
[CONTROLLER: upload()] ⟳
        │
        ▼ ⟳ Promise<Buffer>
[SERVICE: FileService.process()] ⟳⊕★
        │
        ├─ ▶ [EXTERNAL: S3.putObject()] ★  ⟳
        │         │
        │         ▼ { url: string }
        │    [back to FileService]
        │
        ▼ { fileUrl, metadata }
[REPOSITORY: FileRepository.save()] ★⟳
        │
        ▼ FileRecord
[RESPONSE: 201 { id, url }]
```

---

## Architecture Layer Color Bands

When rendering layered architecture, group nodes into color bands:

```txt
Layer band colors (background strip behind nodes):
  Presentation / Route / HTTP    → rgba(102, 141, 184, 0.15)  blue tint
  Middleware / Guard              → rgba(196, 154, 60, 0.15)   amber tint
  Controller / Handler            → rgba(102, 141, 184, 0.20)  blue tint
  Service / Use Case / Business   → rgba(58, 170, 140, 0.15)   green tint
  Repository / Data Access        → rgba(240, 165, 0, 0.15)    gold tint
  Database / External / Cache     → rgba(240, 165, 0, 0.25)    gold tint
```

Label each band on left side with layer name.

---

## Node Type Color Reference

Used in HTML interactive graph artifacts and ASCII badges. Consistent across all outputs.

```txt
TRACE_NODE_TYPES = {
  entry:      color #3aaa8c   label "ENTRY"       — where execution begins
  router:     color #668db8   label "ROUTER"      — route matching
  middleware: color #c49a3c   label "MIDDLEWARE"  — auth, rate limit, validation
  controller: color #668db8   label "CONTROLLER"  — request handler
  service:    color #668db8   label "SERVICE"     — business logic
  repository: color #668db8   label "REPO"        — data access
  db:         color #F0A500   label "DB"          — database, cache, file system
  external:   color #F0A500   label "EXTERNAL"    — third-party API, queue
  response:   color #3aaa8c   label "RESPONSE"    — final output
  error:      color #e05252   label "ERROR"       — exception / error response
  mock:       color #888888   label "MOCK"        — test mock boundary
  assert:     color #9b59b6   label "ASSERT"      — test assertion node
}

EDGE_STYLES = {
  happy:  solid line      color #aaaaaa   — happy path
  error:  dashed line     color #e05252   — error/exception path
  async:  solid line      color #668db8   — async boundary edge
  return: spaced dashes   color #aaaaaa   — return value path
}

BADGES (inline text after node label) = {
  ⟳   async boundary
  ★   side effect
  ⊕   data transform
  ?   conditional branch
}
```

---

## Trace Depth Limit

```txt
Default trace depth: unlimited (follow all edges to terminus)

If codebase is large (>30 files) and trace would exceed ~15 nodes:
  → show first 10 hops fully
  → collapse remainder into [... N more steps] node
  → offer "expand" on that node
```

---

## Multi-Entry Trace (batch)

When user wants to trace all routes or all entry points at once:

```txt
- Render each entry as a separate color-coded starting node
- Show shared nodes (e.g., same middleware, same service) as shared nodes
  with multiple incoming edges from different traces
- Use edge color to distinguish which trace each edge belongs to
- Shared nodes get a "shared by N traces" badge
```

This reveals which layers are shared bottlenecks vs isolated paths.

---

## Cypress / Playwright Trace Specifics

```txt
cy.visit('/dashboard')
  → Entry: GET /dashboard
  → if app code provided: trace into route handler
  → if not: show as [ENTRY] → [APP: /dashboard] → [ASSERT: ...] chain

cy.intercept('POST', '/api/auth', { fixture: 'user.json' })
  → creates [MOCK] node replacing real handler
  → dashed border, gray fill
  → label: "mocked — returns fixture"

page.waitForResponse('/api/data')
  → async boundary on that edge ⟳
  → shows response shape if fixture/assertion provides it

expect(response.status()).toBe(200)
  → [ASSERT] node at end of chain
  → purple, shows assertion condition as label
```