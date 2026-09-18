# LAM-39 — Gaps Found Auditing E-LAM-0001

**Backend:** `LAM-39-B` → `LAM-29-B` · PR #9 · commit `1d7a27b` · 7 files, +511 / −39
**Frontend:** `LAM-39-F` → `E-LAM-0001-F` · PR [#3](https://github.com/Willpatbarr/LaminarFlow-Frontend/pull/3) · 2 files
**Ticket:** [LAM-39](https://linear.app/willieworkspace/issue/LAM-39/close-the-gaps-found-auditing-e-lam-0001) (created by this audit)

> ### The merges still have not landed
>
> You said you merged all three. GitHub reports backend [#7](https://github.com/Willpatbarr/LaminarFlow-Backend/pull/7), backend [#8](https://github.com/Willpatbarr/LaminarFlow-Backend/pull/8), and frontend [#2](https://github.com/Willpatbarr/LaminarFlow-Frontend/pull/2) as `state=OPEN, merged=null`, and both epic branches are unchanged at `4cc7cb7` and `e00db35`. The last successful merges were backend #6 at 00:49 and frontend #1 at 00:58.
>
> That is twice in a row. Worth checking for a branch protection rule or an unsaved merge dialog. This branch stacks on `LAM-29-B` because that is the only branch holding the full state.

## What the audit covered

Every ticket in E-LAM-0001 (LAM-1, 2, 3, 4, 5, 28, 29, 38) step by step against the code, plus the Architecture & Tech Stack, Testing Strategy, and Master Spec notes. Four gaps — and writing the fix for the first one uncovered a fifth.

---

## Backend — LaminarFlow-Backend

#### main.go

### Change #0001
:56-60
```go
+ bundle, err := frontendFS(cfg.FrontendDir)
+ if err != nil {
+ 	log.Fatalf("frontend: %v", err)
+ }
+
+ mux := newMux(pool, bundle)
```

- **What:**
  - replace ~40 lines of inline mux construction with one call
- **Why:**
  - **B** — the routing had no test coverage at all; only LAM-29's container smoke test in CI touched it
  - **C** — `main()` needs a real database URL and a listening socket, so nothing in it was reachable from a test

### Change #0002
:153-194
```go
+ // Deleting the /api/ registration below breaks every unknown API endpoint
+ // and, before this function existed, broke no test.
+ func newMux(pool *pgxpool.Pool, bundle fs.FS) *http.ServeMux {
+ 	mux := http.NewServeMux()
+ 	...
+ 	return mux
+ }
```

- **What:**
  - move every route registration into a testable function, comments unchanged
- **Why:**
  - **B** — a local `./scripts/test.sh` passed with the routing completely broken, which is the exact failure LAM-38 exists to remove
  - **C** — returning the mux rather than a `*http.Server` keeps the test free of ports and lets `httptest` drive it directly

---

#### internal/frontend/frontend.go

### Change #0003 ← **a real bug, found by Change #0004**
:57-65
```go
+ if r.Method != http.MethodGet && r.Method != http.MethodHead {
+ 	w.Header().Set("Allow", "GET, HEAD")
+ 	http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
+ 	return
+ }
```

- **What:**
  - answer GET and HEAD only; 405 with an `Allow` header otherwise
- **Why:**
  - **B** — `GET /healthz` only matches GET, so **every other method fell through to the catch-all and got the app shell with status 200**. `POST /healthz`, and any write to any unrouted path, looked like it had succeeded
  - **C** — the guard belongs on the static handler, not each route: it is a property of serving files, not of any one path
  - **C** — HEAD stays allowed; it is how a client checks an asset without fetching it

---

#### main_test.go **(new file)**

### Change #0004 ← **found Change #0003**
:186-199
```go
+ func TestHealthzIsNotShadowedByTheCatchAll(t *testing.T) {
+ 	mux.ServeHTTP(rec, httptest.NewRequest(http.MethodPost, "/healthz", nil))
+ 	if strings.Contains(rec.Body.String(), "<div id=root>") {
+ 		t.Error("POST /healthz fell through to the frontend handler")
+ 	}
```

- **What:**
  - assert a non-GET request to a health path is not swallowed by the catch-all
- **Why:**
  - **B** — this failed on first run with `status = 200, want 405`, which is how the method bug was found
  - **C** — Go's method-scoped patterns do not exclude a broader pattern from matching other methods

### Change #0005
:139-160
```go
+ func TestUnknownAPIRouteReturnsJSONNotTheAppShell(t *testing.T) {
+ 	for _, path := range []string{"/api/", "/api/nope", "/api/documents/42"} {
+ 		if strings.Contains(rec.Body.String(), "<div id=root>") {
+ 			t.Error("an unknown API route returned the app shell")
```

- **What:**
  - lock the `/api/` reservation LAM-28 added
- **Why:**
  - **B** — delete that handler and, before this test, every Go test still passed while every unknown endpoint silently returned HTML with status 200
  - **C** — `fetch()` reports that as a JSON parse error, three layers from the cause

### Change #0006
:62-91, :106-137
```go
+ // Close the pool out from under the mux, then ask for liveness anyway.
+ pool.Close()
+ rec := get(t, mux, "/healthz")
+ if rec.Code != http.StatusOK { ... }
```

- **What:**
  - prove liveness survives a dead database, and readiness does not
- **Why:**
  - **B** — the comment in `newMux` claims `/healthz` deliberately avoids Postgres; nothing checked it
  - **C** — closing the pool is the cheapest honest way to simulate the outage

---

#### internal/config/config_test.go **(new file)**

### Change #0007
:80-98
```go
+ for _, value := range []string{"", "false", "1", "yes", "TRUE", "True", "on"} {
+ 	if cfg.MigrateOnStartup {
+ 		t.Errorf("MIGRATE_ON_STARTUP=%q enabled migrations", value)
+ 	}
+ }
```

- **What:**
  - assert only exactly `"true"` enables startup migrations
- **Why:**
  - **B** — Testing Strategy §1 names this exact case: pure logic, no database, unit test it
  - **C** — `TRUE` and `yes` reading as false is the safe direction, and worth pinning so nobody "fixes" it into a loose parse

### Change #0008
:117-130
```go
+ func TestFailedLoadReturnsZeroConfig(t *testing.T) {
+ 	if cfg != (Config{}) {
+ 		t.Errorf("Load returned %+v on error, want the zero Config", cfg)
+ 	}
+ }
```

- **What:**
  - assert a failed `Load` hands back nothing usable
- **Why:**
  - **C** — a half-populated Config alongside an error is the kind of thing a caller ignores the error for

---

#### internal/frontend/frontend_test.go

### Change #0009
:143-163, :165-176
```go
+ func TestRejectsNonReadMethods(t *testing.T) {
+ 	for _, method := range []string{POST, PUT, PATCH, DELETE} { ... }
+ }
+ func TestHeadIsAllowed(t *testing.T) { ... }
```

- **What:**
  - cover the new method guard in both directions
- **Why:**
  - **C** — a guard that rejected HEAD too would break asset probing without any test noticing

---

#### docs/adr/0002-deployment-packaging.md **(new file)**

### Change #0010
:1-84
```txt
+ # 2. Deployment packaging: one image, no shell, frontend embedded
+ Status: Accepted — 2026-09-04 (LAM-29, recorded by LAM-39)
```

- **What:**
  - record LAM-29's still-in-force decisions as an ADR
- **Why:**
  - **B** — `docs/adr/README.md` says records hold decisions still in force, and that change documents describe a ticket and are **not maintained**
  - **C** — leaving the distroless base, the named build context, and the embedded bundle only in `docs/changes/LAM-29-changes.md` contradicted our own stated convention

### Change #0011
docs/adr/README.md, records table
```txt
+ | [2](0002-deployment-packaging.md) | Deployment packaging: one image, no shell, frontend embedded | Accepted |
```

- **What:**
  - add the row
- **Why:**
  - **C** — the table is the index; an ADR missing from it is an ADR nobody finds

---

## Frontend — LaminarFlow-Frontend

#### scripts/check-no-db-driver.mjs **(new file)**

### Change #0012
:20-30
```js
+ const BANNED_DEPS = ['pg', 'postgres', 'prisma', 'drizzle', 'knex', ...]
+ const BANNED_SOURCE = /postgres(?:ql)?:\/\/|mysql:\/\/|DATABASE_URL/
```

- **What:**
  - fail if a database driver or a connection string appears
- **Why:**
  - **B** — LAM-2 step 4 said *"Confirm the frontend has no Postgres driver or connection string."* It was confirmed once, by reading; nothing has stopped `npm install pg` since
  - **C** — same shape as LAM-5: structure makes the right path easy, not the wrong path impossible
  - **C** — substring matching catches `pg-promise` and `@prisma/client`

### Change #0013
package.json scripts
```json
+ "check:boundary": "node scripts/check-no-db-driver.mjs",
+ "lint": "oxlint && npm run check:boundary"
```

- **What:**
  - wire the check into `npm run lint`
- **Why:**
  - **C** — CI already runs `npm run lint`, so no workflow change is needed and the check cannot be forgotten

---

## Decisions I made

| # | Decision | Alternative |
|---|----------|-------------|
| 1 | **Fix the method bug rather than assert the old behaviour** — the test failed, and returning HTML 200 for `POST /anything` is wrong, not merely surprising | Change the test to document current behaviour. Rejected: a write that appears to succeed is the worst kind of harmless |
| 2 | **Guard on the static handler**, not per-route | Register each route for all methods. Rejected: `POST /healthz` returning `{"status":"ok"}` would imply POST is supported |
| 3 | **Created LAM-39** rather than folding fixes into an existing ticket | Amend LAM-29. Rejected: those tickets are Done, and an audit finding deserves its own record |
| 4 | **Branched off `LAM-29-B`** | Off the epic branch — but it lacks LAM-28 and LAM-29 entirely |
| 5 | **Substring-matched banned deps** | Exact names only. Rejected: `pg-promise` would slip through |
| 6 | **ADR 0002 covers all of LAM-29's decisions in one record** | One ADR per decision. Rejected: they are one coherent packaging choice, and splitting them would obscure that |
| 7 | **Playwright still deferred** | Add it now. Testing Strategy §4 asks for early scaffolding, but the frontend remains the unmodified Vite template — there is no flow to drive. LAM-38's reasoning still holds |

---

## Verification

`./scripts/test.sh` green — every package now has tests except `cmd/*`, `internal/db`, `internal/dbtest`, `migrations`, and `web`, which are thin wrappers or embed declarations.

**The method bug was found by a failing test, not by inspection:**

```
main_test.go:193: POST /healthz fell through to the frontend handler
main_test.go:196: status = 200, want 405
--- FAIL: TestHealthzIsNotShadowedByTheCatchAll
```

**The frontend guard was confirmed to fail, not just to pass:**

| Injected | Exit | Message |
|---|---|---|
| `"pg": "^8.0.0"` in dependencies | 1 | `package.json dependencies: pg` |
| `postgres://u:p@h:5432/db` in `src/scratch.ts` | 1 | `src/scratch.ts:1` |
| Clean tree | 0 | ✓ |

Exit codes were checked directly — a guard that prints a complaint but exits 0 would let CI pass.

---

## What the audit found clean

* Every LAM-3 step — blob, index, single write path, rebuild, drift test
* Every LAM-4 step — workspace table, no single-workspace assumptions, cross-workspace test
* LAM-29's portability audit — no host-specific values in the image or config
* `.env.example` covers every variable the code reads
* The frontend genuinely has no database driver or connection string today

## Follow-up actions

1. **Work out why the merges are not landing.** Three PRs are open that you believe are merged.
2. **Merge in order:** backend #7 → #8 → #9, and frontend #2 → #3.
3. **Set LAM-29 → Done and LAM-39 → Done** once merged.
4. **Build the image once yourself** — still never run outside CI, and never on Apple Silicon.
5. **Adopt the dev database** — outstanding since LAM-10: `migrate baseline && migrate up`.
6. `gh auth switch --user willbarr_church`
