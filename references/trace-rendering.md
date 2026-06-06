# Trace Rendering Reference

Detail rules untuk render execution trace graph di Code Graph skill.

---

## Happy Path vs Error Path

```txt
Happy path  -> solid edges (-->)
Expected error response -> dashed edge (-.->), [ERROR] node at terminus, labeled by status/condition
Anomaly path -> dashed edge (-.->), [ERROR] node at terminus, labeled by PATTERN_NAME

Always render expected error branches present in code. Do not hide them just
because they are normal application control-flow. Security/fatal anomaly paths
are additional evidence-backed branches, not a replacement for returned 4xx/5xx
responses.
```

```
[ENTRY: POST /login]
        |
        v { email, password }
[MIDDLEWARE: AuthMiddleware]
        |                    ------------------------> [ERROR: 429 Rate Limited]
        v req.body validated
[CONTROLLER: AuthController.login()]
        |                    ------------------------> [ERROR: 422 Validation]
        v { email, password }
[SERVICE: AuthService.authenticate()] *(~)
        |                    ------------------------> [ERROR: 401 Invalid creds]
        v email
[REPOSITORY: UserRepository.findByEmail()] *
        |
        v User | null
[SERVICE: AuthService -- bcrypt.compare()] (+)
        |
        v { user, token }
[RESPONSE: 200 { token, user }]
```

For route handlers, returned application errors must point to the local response
builder or status mapper when present:

```txt
[CONTROLLER: CustomerHandler.GetAll()] ?
        |
        +- [router_id parse fails] ---------> [ERROR: 400 invalid router_id]
        |
        v { routerID, search, active, limit, offset, principal }
[SERVICE: CustomerService.GetAllForUser()] ?
        |
        +- [scope denied] ------------------> [HELPER: customerErrorStatus()] (+)
        |                                             |
        |                                             v status:int
        |                                      [ERROR: 403 forbidden]
        v query params + scope
[REPOSITORY: CustomerRepo.GetAllScoped()] *
        |
        +- [db returns error] ---------------> [HELPER: customerErrorStatus()] (+)
        |                                             |
        |                                             v status:int
        |                                      [ERROR: 500 db/query error]
        v []Customer
[RESPONSE: 200 pkg.Response{ data: customers }]
```

---

## Edge Label Rules

Every edge in a trace graph must be labeled. Format:

```txt
Short type hint  ->  { shape }   or   primitiveType: value_hint

Examples:
  -> { userId: string, role: string }
  -> string | null
  -> Error: 401
  -> void  (for fire-and-forget)
  -> Promise<User>  (async, add (~) on edge)
  -> event: 'user.created'  (for event emitters)
```

When exact type unknown -> infer from context, prefix with `~`:
```txt
  -> ~{ user object }
  -> ~array of results
```

---

## Node Badge Rules

```txt
*  Side effect
     -> node writes to DB, calls external API, writes file, emits event, sends email
     -> place on the node that performs the side effect or externally visible I/O
     -> DB/cache reads are * when they cross a process boundary; pure in-memory reads are not

(~)  Async boundary
     -> function is async / returns Promise / uses await
     -> place on BOTH the node and the edge going into it

(+)  Data transform
     -> node changes the shape/structure of data (not just passes through)
     -> e.g. serializer, mapper, formatter, reducer
     -> if both async AND transform apply: (~)(+)
     -> if side effect AND async: *(~)
     -> if all three: *(~)(+)

?  Conditional branch
     -> node has if/switch that sends execution different directions
     -> show as fork: two edges out, each labeled with condition
```

---

## Conditional Branch Rendering

```txt
[SERVICE: OrderService.create()]  ?
        |
        +- [user.role === 'admin'] --> [SERVICE: applyAdminDiscount()]
        |                                      |
        +- [user.role !== 'admin'] -------------+
                                               |
                                               v
                                    [REPOSITORY: OrderRepository.save()] *
```

Do not collapse branches. Always show the fork explicitly.

Branches that terminate early in HTTP handlers still count as full trace
branches. Show the condition, status code, response helper, and terminal output.
If the exact helper is not available in provided code, use a [PARTIAL] helper
node rather than jumping directly from condition to response.

---

## Async Chain Rendering

```txt
[CONTROLLER: upload()] (~)
        |
        v (~) Promise<Buffer>
[SERVICE: FileService.process()] (~)(+)*
        |
        +- > [EXTERNAL: S3.putObject()] *  (~)
        |         |
        |         v { url: string }
        |    [back to FileService]
        |
        v { fileUrl, metadata }
[REPOSITORY: FileRepository.save()] *(~)
        |
        v FileRecord
[RESPONSE: 201 { id, url }]
```

---

## Architecture Layer Grouping

When rendering layered architecture, group nodes into labeled bands (ASCII only, no colors):

```txt
Layer bands (label each band on the left with its layer name):
  Presentation / Route / HTTP
  Middleware / Guard
  Controller / Handler
  Service / Use Case / Business
  Repository / Data Access
  Database / External / Cache
```

---

## Node Type Reference

ASCII labels only. No colors, no HTML. Consistent across all outputs.

```txt
TRACE_NODE_TYPES = {
  entry:      [ENTRY]       -- where execution begins
  router:     [ROUTER]      -- route matching
  middleware: [MIDDLEWARE]  -- auth, rate limit, validation
  controller: [CONTROLLER]  -- request handler
  service:    [SERVICE]     -- business logic
  repository: [REPO]        -- data access
  db:         [DB]          -- database, cache, file system
  external:   [EXTERNAL]    -- third-party API, queue
  response:   [RESPONSE]    -- final output
  error:      [ERROR]       -- exception / error response
  mock:       [MOCK]        -- test mock boundary
  assert:     [ASSERT]      -- test assertion node
}

EDGE_STYLES = {
  happy:  -->    solid line     -- happy path
  error:  -.->   dashed line    -- error/exception path
  async:  ~~>    async line      -- async boundary edge
  return: - ->   spaced dashes  -- return value path
}

BADGES (inline text after node label) = {
  (~)   async boundary
  *   side effect
  (+)   data transform
  ?   conditional branch
}
```

---

## Trace Depth Limit

```txt
Default trace depth: unlimited (follow all edges to terminus)

If codebase is large (>30 files) and trace would exceed ~15 nodes:
  -> show first 10 hops fully
  -> collapse remainder into [... N more steps] node
  -> offer "expand" on that node
```

---

## Multi-Entry Trace (batch)

When user wants to trace all routes or all entry points at once:

```txt
- Render each entry as a separate labeled starting node
- Show shared nodes (e.g., same middleware, same service) as shared nodes
  with multiple incoming edges from different traces
- Tag each edge with its entry-point label to show which trace it belongs to
- Shared nodes get a "shared by N traces" badge
```

This reveals which layers are shared bottlenecks vs isolated paths.

---

## Cypress / Playwright Trace Specifics

```txt
cy.visit('/dashboard')
  -> Entry: GET /dashboard
  -> if app code provided: trace into route handler
  -> if not: show as [ENTRY] -> [APP: /dashboard] -> [ASSERT: ...] chain

cy.intercept('POST', '/api/auth', { fixture: 'user.json' })
  -> creates [MOCK] node replacing real handler
  -> dashed connector (-.->), tagged [MOCK]
  -> label: "mocked -- returns fixture"

page.waitForResponse('/api/data')
  -> async boundary on that edge (~)
  -> shows response shape if fixture/assertion provides it

expect(response.status()).toBe(200)
  -> [ASSERT] node at end of chain
  -> shows assertion condition as label
```
