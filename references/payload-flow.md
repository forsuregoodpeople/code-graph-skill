# Payload / Contract Flow Reference

Render rules for the **payload-centric** lens in the Code Graph skill: not "which
function calls which" (that is reachability), but "how the SHAPE of the data changes
from the request coming in to the response going out". Use the same entry points and
lanes as the reachability graph -- this is a second lens, not a second trace. ASCII only.

---

## Entry point for the payload lens (IMPORTANT -- altitude)

Payload lives at the **route / handler / command / consumer** level, NOT the process /
binary level. If the reachability graph is grouped coarsely (e.g. one node "cmd/net (API)"
standing for hundreds of routes), the payload will not be visible. So for the payload lens,
drop the altitude:

```txt
Payload entry point =
  HTTP/RPC   -> each concrete route: METHOD + path (POST /v1/customers), not "the API binary"
  CLI        -> each subcommand + the flags/args it parses
  Worker/job -> each consumer + the shape of the message/job payload it decodes
  Event      -> each subscriber + the shape of the event it listens for
  WebSocket  -> each message type + the shape of the inbound/outbound frame
```

Many routes -> do not silently pick one. Render a payload column for EVERY reachable route
(merge routes whose shape is identical and note "routes X, Y, Z share this shape"). The
reachability graph may stay at module altitude; the payload lens MUST be at route altitude.
If there are genuinely too many for one response, render all mutation routes
(POST/PUT/PATCH/DELETE -- the ones that carry a body and write) in full, then list the
GET/read routes briefly, and state explicitly which were summarized and why.

---

## What is tracked

For each entry point (route): the payload shape at each hop, and the DIFF between hops.

```txt
request shape (entry)
   -> validate / DTO / parse
   -> service transform
   -> entity / persist shape (write)
   -> response DTO / serializer
   -> response shape (terminal)
```

Each arrow carries a diff block: which fields changed at that hop.

---

## Target function -- REQUIRED on every node (where the payload is sent)

A node is not just a layer/module ("service", "repo") -- it MUST name the **concrete target
function** that receives the payload: qualified name + `file:line`. Without this the reader
knows "the payload goes into the service" but not which function exactly.

```txt
Node format:
  [LAYER: pkg.Type.Method(params)]  file/path.go:line

Edge format (arrow):
  -> targetFn(argShape)            // function being called + argument shape

Example:
  [SERVICE: CustomerService.Isolate(ctx, cmd)]  app/customer/isolflow.go:88
       | -> repo.SetIsolation(ctx, routerID, on)   { routerID, on: bool }
       v
  [REPO: CustomerRepo.SetIsolation(...)]  app/customer/repo.go:212  *
```

Rules:
- Function visible in code -> full name + file:line. Required.
- Dynamic dispatch / interface -> `[DYNAMIC: ~target]` or `[INTERFACE: Port.Method()]`,
  dashed edge, never invent the implementor.
- Function not visible (code not provided) -> `[PARTIAL: ~targetFn()]`, do not guess the signature.
- External lib -> `[EXTERNAL: pkg.Call()]`, not traced into.

---

## Per-hop diff notation

```txt
+field           field added at this hop
-field           field dropped / not forwarded
~old->new        field renamed
field: T1=>T2    field type changed (e.g. string=>Date, raw=>hashed)
field?           field optional / nullable
[SENSITIVE]      tag on a secret field (password, hash, token, secret, ssn, card)
~{ ... }         shape uncertain / inferred -- do not invent field names
```

Rule: if the shape is not visible in the provided code -> mark `~unknown`/`[PARTIAL]`,
do NOT guess field names.

---

## Request shape sources per stack

Record where the INITIAL shape comes from (it becomes the first row of the payload column):

```txt
Express/Koa   req.body / req.query / req.params / req.headers
NestJS        DTO class + @Body() @Query() @Param(); class-validator decorators
FastAPI       Pydantic model param; Query()/Path()/Header(); request.json()
Django/DRF    serializer.validated_data; request.data; request.GET
Laravel       FormRequest->validated(); $request->only([...]); $request->input()
Rails         strong params: params.require(:x).permit(:a,:b)
Go net/http   struct + json.NewDecoder(r.Body).Decode(&dto); r.URL.Query()
Spring Boot   @RequestBody DTO; @RequestParam; @PathVariable
GraphQL       input type / args object
```

Validation is detected (set `validated:true`) when present: schema validator (zod/joi/yup/
class-validator), Pydantic model, FormRequest rules, strong params permit, struct tag +
explicit validate. Without any of these -> `validated:false` -> UNVALIDATED_PAYLOAD candidate.

---

## Rendering the payload column (full example)

One payload column per entry point. Shape on the left of each node, diff on the arrow,
target function named on every node.

```txt
PAYLOAD FLOW -- POST /users (register)  ->  UserController.register()  user_controller.ts:24

  [REQUEST]  { name, email, password, isAdmin }            <- req.body (no validation) [!] UNVALIDATED_PAYLOAD
       |  -> userService.create(body)        { name, email, password, isAdmin }
       v
  [SERVICE: UserService.create(dto)]  user_service.ts:41
       |  +passwordHash: string  (bcrypt)   password: raw=>hashed [SENSITIVE]
       |  -> usersRepo.insert(entity)        (whole body spread into entity)
       v
  [PERSIST: UsersRepo.insert(row)]  users_repo.ts:88  *   [!!] OVERPOSTING -- isAdmin from body persisted
       |  shape: { name, email, password[SENSITIVE], passwordHash[SENSITIVE], isAdmin }
       |  -> res.status(201).json(user)
       v
  [RESPONSE: 201 user]  user_controller.ts:33        [!!] LEAKED_FIELD -- password, passwordHash in body
       shape: { id, name, email, password[SENSITIVE], passwordHash[SENSITIVE], isAdmin }
       declared: UserDTO { id, name, email }  => [!] CONTRACT_BREAK (extra fields)

  Legend: + add  - drop  ~ rename  T1=>T2 retype  [SENSITIVE] secret  * write
```

Compare with a healthy version (for contrast, not required to render):

```txt
  [REQUEST] { name, email, password }  <- CreateUserDto (validated)
       | password: raw=>hashed [SENSITIVE]  +passwordHash  -password
       v
  [PERSIST] { name, email, passwordHash[SENSITIVE] }  *
       | -passwordHash  +id  +createdAt
       v
  [RESPONSE 201] { id, name, email, createdAt }  == UserDTO  [OK]
```

---

## Inline detection rules (4 contract patterns)

Flag only when there is evidence in the provided code. Name the pattern on the node.

```txt
LEAKED_FIELD        [!!]
  Signal: a [SENSITIVE] field (password, *hash, token, secret, apiKey, ssn, card, *_key)
          is still present in the shape at the [RESPONSE] node, or a full entity is returned
          with no select/pick/serializer that strips the secret fields.
  Mark on: the [RESPONSE] node. Name the field.

OVERPOSTING         [!!]
  Signal: req.body / the whole input object is spread or passed intact into create/save/
          insert/update with no whitelist (no pick/only/permit/DTO field list). Privileged
          fields (isAdmin, role, ownerId, verified) can be set by the client.
  Mark on: the [PERSIST]/[SERVICE] node where the body enters the write.

UNVALIDATED_PAYLOAD [!]
  Signal: the payload is used/persisted with no schema validation (validated:false).
          A sharpening of UNVALIDATED_INPUT to the shape level -- focused on the payload
          shape, not just "raw input used".
  Mark on: the [REQUEST] node or the first node that consumes it.

CONTRACT_BREAK      [!]
  Signal: the shape at [RESPONSE] != the declared type/DTO/return type -- missing, extra,
          typo'd, or differently-typed field vs the contract (OpenAPI/DTO/interface/struct
          json tag).
  Mark on: the [RESPONSE] node. Show declared vs actual.
```

Relation to existing patterns (do not duplicate):
```txt
SHAPE_DRIFT    -> downstream reads the PRE-transform version of a field. CONTRACT_BREAK
                  focuses on response-vs-declared-contract mismatch.
TYPE_MISMATCH  -> producer type != consumer type. Here it shows as "T1=>T2" on a hop.
KEY_MISMATCH   -> publish key != listen key (event/queue). A different lens from HTTP payload.
```

---

## Multi-entry payload view

Same as reachability: one payload column per entry point, each labeled with its entry.
A transform node used by many entries (e.g. a shared serializer) is tagged "shared by N" --
useful for spotting one serializer that leaks a field into several responses at once.

---

## Unknown / partial shape

```txt
- Shape not visible in the code -> ~unknown, do NOT invent fields.
- File truncated -> show only the visible fields, mark the node [PARTIAL].
- Response type declared but body not visible -> show the declared shape, mark the actual
  as ~unknown, do not claim CONTRACT_BREAK without evidence.
```
