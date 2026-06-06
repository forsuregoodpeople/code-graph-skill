# Payload / Contract Flow Reference

Render rules untuk lensa **payload-centric** di Code Graph skill: bukan "fungsi apa
memanggil fungsi apa" (itu reachability), tapi "bagaimana BENTUK data berubah dari
request masuk sampai response keluar". Pakai entry point dan lane yang SAMA dengan
reachability graph -- ini lensa kedua, bukan trace kedua. ASCII only.

---

## Entry point untuk lensa payload (PENTING -- altitude)

Payload hidup di level **route/handler/command/consumer**, BUKAN di level proses/binary.
Jika graph reachability dikelompokkan kasar (mis. satu node "cmd/net (API)" mewakili
ratusan route), payload tidak akan terlihat. Maka untuk lensa payload, turunkan altitude:

```txt
Entry point payload =
  HTTP/RPC   -> tiap route konkret: METHOD + path (POST /v1/customers), bukan "the API binary"
  CLI        -> tiap subcommand + flags/args yang diparse
  Worker/job -> tiap consumer + bentuk message/job payload yang di-decode
  Event      -> tiap subscriber + bentuk event yang didengar
  WebSocket  -> tiap message type + bentuk frame masuk/keluar
```

Banyak route -> jangan diam-diam pilih satu. Render kolom payload untuk SETIAP route
yang dapat dijangkau (gabungkan yang shape-nya identik dan sebut "route X, Y, Z share
this shape"). Reachability boleh tetap altitude modul; lensa payload WAJIB altitude route.
Jika benar-benar terlalu banyak untuk satu response, render semua route mutasi
(POST/PUT/PATCH/DELETE -- yang membawa body & menulis) penuh, lalu daftar route GET/baca
secara ringkas, dan nyatakan eksplisit mana yang diringkas dan kenapa.

---

## Apa yang dilacak

Untuk setiap entry point (route): bentuk (shape) payload pada tiap hop, dan DIFF antar hop.

```txt
request shape (entry)
   -> validate / DTO / parse
   -> service transform
   -> entity / persist shape (write)
   -> response DTO / serializer
   -> response shape (terminal)
```

Tiap panah membawa blok diff: field apa yang berubah di hop itu.

---

## Target function -- WAJIB di tiap node (ke mana payload ditujukan)

Tiap node bukan sekadar layer/modul ("service", "repo") -- ia HARUS menyebut **fungsi
tujuan konkret** yang menerima payload itu: nama berkualifikasi + `file:line`. Tanpa ini
pembaca tahu "payload masuk ke service" tapi tidak tahu fungsi mana persisnya.

```txt
Format node:
  [LAYER: pkg.Type.Method(params)]  file/path.go:line

Format edge (panah):
  -> targetFn(argShape)            // fungsi yang dipanggil + bentuk argumen

Contoh:
  [SERVICE: CustomerService.Isolate(ctx, cmd)]  app/customer/isolflow.go:88
       | -> repo.SetIsolation(ctx, routerID, on)   { routerID, on: bool }
       v
  [REPO: CustomerRepo.SetIsolation(...)]  app/customer/repo.go:212  *
```

Aturan:
- Fungsi terlihat di kode -> nama lengkap + file:line. Wajib.
- Dynamic dispatch / interface -> `[DYNAMIC: ~target]` atau `[INTERFACE: Port.Method()]`,
  edge dashed, jangan mengarang implementor.
- Fungsi tak terlihat (kode tak diberikan) -> `[PARTIAL: ~targetFn()]`, jangan menebak signature.
- External lib -> `[EXTERNAL: pkg.Call()]`, tidak ditelusuri ke dalam.

---

## Notasi diff per hop

```txt
+field           field ditambahkan di hop ini
-field           field dibuang / tidak diteruskan
~old->new        field di-rename
field: T1=>T2    tipe field berubah (mis. string=>Date, raw=>hashed)
field?           field opsional / nullable
[SENSITIVE]      tag pada field rahasia (password, hash, token, secret, ssn, card)
~{ ... }         shape tidak pasti / disimpulkan -- jangan mengarang field
```

Aturan: jika shape tidak terlihat di kode yang diberikan -> tandai `~unknown`/`[PARTIAL]`,
JANGAN menebak nama field.

---

## Sumber request shape per stack

Catat dari mana shape AWAL diambil (jadi baris pertama kolom payload):

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

Validasi terdeteksi (set `validated:true`) bila ada: schema validator (zod/joi/yup/
class-validator), Pydantic model, FormRequest rules, strong params permit, struct tag
+ explicit validate. Tanpa salah satu -> `validated:false` -> kandidat UNVALIDATED_PAYLOAD.

---

## Render kolom payload (contoh penuh)

Satu kolom payload per entry point. Shape di kiri tiap node, diff di panah.

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

Bandingkan dengan versi sehat (untuk kontras, tidak wajib dirender):

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

## Aturan deteksi inline (4 pola kontrak)

Flag hanya bila ada bukti di kode yang diberikan. Sebut nama pola di node.

```txt
LEAKED_FIELD        [!!]
  Sinyal: field [SENSITIVE] (password, *hash, token, secret, apiKey, ssn, card, *_key)
          masih ada di shape pada node [RESPONSE], atau entity utuh dikembalikan tanpa
          select/pick/serializer yang membuang field rahasia.
  Tandai di: node [RESPONSE]. Sebut field-nya.

OVERPOSTING         [!!]
  Sinyal: req.body / seluruh input object di-spread atau diteruskan utuh ke create/
          save/insert/update tanpa whitelist (no pick/only/permit/DTO field list).
          Field privileged (isAdmin, role, ownerId, verified) bisa ikut terpersist.
  Tandai di: node [PERSIST]/[SERVICE] tempat body masuk ke write.

UNVALIDATED_PAYLOAD [!]
  Sinyal: payload dipakai/dipersist tanpa schema validation (validated:false).
          Pertajaman UNVALIDATED_INPUT ke level shape -- fokus pada bentuk payload,
          bukan sekadar "input mentah dipakai".
  Tandai di: node [REQUEST] atau hop pertama yang mengonsumsinya.

CONTRACT_BREAK      [!]
  Sinyal: shape pada [RESPONSE] != tipe/DTO/return type yang dideklarasikan --
          field hilang, ekstra, typo, atau tipe beda dari kontrak (OpenAPI/DTO/
          interface/struct json tag).
  Tandai di: node [RESPONSE]. Tunjukkan declared vs actual.
```

Hubungan dengan pola yang sudah ada (jangan diduplikasi):
```txt
SHAPE_DRIFT    -> downstream baca field versi PRA-transform. CONTRACT_BREAK fokus ke
                  ketidakcocokan response vs kontrak yang dideklarasikan.
TYPE_MISMATCH  -> tipe produsen != tipe konsumen. Di sini muncul sebagai "T1=>T2" pada hop.
KEY_MISMATCH   -> key publish != key listen (event/queue). Lensa berbeda dari payload HTTP.
```

---

## Multi-entry payload view

Sama seperti reachability: satu kolom payload per entry point, beri label entry.
Node transform yang dipakai banyak entry (mis. serializer bersama) ditandai
"shared by N" -- berguna untuk melihat satu serializer yang membocorkan field ke
beberapa response sekaligus.

---

## Unknown / partial shape

```txt
- Shape tidak terlihat di kode -> ~unknown, JANGAN mengarang field.
- File terpotong -> field yang terlihat saja, node ditandai [PARTIAL].
- Tipe respons dideklarasikan tapi body tak terlihat -> tampilkan declared shape,
  tandai actual sebagai ~unknown, jangan klaim CONTRACT_BREAK tanpa bukti.
```
