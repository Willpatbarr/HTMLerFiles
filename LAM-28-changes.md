# LAM-28 — Serve the Built Frontend from the Go Backend (Same-Origin)

**Backend:** `LAM-28-B` → `E-LAM-0001-B` · PR [#7](https://github.com/Willpatbarr/LaminarFlow-Backend/pull/7) · **CI green**
**Frontend:** `LAM-28-F` → `E-LAM-0001-F` · PR [#2](https://github.com/Willpatbarr/LaminarFlow-Frontend/pull/2) · **CI green**

Backend: 11 files, +413 / −1 · Frontend: 2 files, +39 / −3

**Decisions made before building:** embed the bundle with a `FRONTEND_DIR` override · touch both repos · build script now, container in LAM-29.

---

## Backend — LaminarFlow-Backend

#### web/embed.go **(new file)**

### Change #0001
:18-19
```go
+ //go:embed all:dist
+ var Dist embed.FS
```

- **What:**
  - embed the built bundle, using the `all:` prefix
- **Why:**
  - **B** — the binary is the whole deployment; LAM-29's image ships one file
  - **C** — without `all:` the directive skips dotfiles, and on a fresh clone `.gitkeep` is the only file present — an empty match is a compile error

---

#### .gitignore

### Change #0002
:4-9
```gitignore
+ /web/dist/*
+ !/web/dist/.gitkeep
```

- **What:**
  - ignore the build output but keep the directory marker
- **Why:**
  - **B** — a 193 KB bundle does not belong in git
  - **C** — `go:embed` fails to compile if the directory does not exist, so *something* must be committed

---

#### internal/frontend/frontend.go **(new file)**

### Change #0003
:43-49
```go
+ func New(fsys fs.FS) http.Handler {
+ 	return &handler{files: fsys}
+ }
```

- **What:**
  - take an `fs.FS` rather than a concrete type
- **Why:**
  - **C** — the same handler then serves the embedded bundle and an `os.DirFS` override with no branching

### Change #0004
:51-73
```go
+ name := strings.TrimPrefix(path.Clean(r.URL.Path), "/")
+ if name == "" { name = indexFile }
+ if !fs.ValidPath(name) {
+ 	http.NotFound(w, r)
+ 	return
+ }
```

- **What:**
  - normalise the path and reject anything escaping the bundle
- **Why:**
  - **C** — `http.FileServer` does this for you; doing the lookup by hand means doing this by hand too

### Change #0005 ← **caught a real bug**
:77-80
```go
+ // A missing file that looks like an asset is a 404, not a fallback.
+ if path.Ext(name) != "" && name != indexFile {
+ 	http.NotFound(w, r)
+ 	return
+ }
```

- **What:**
  - restrict the SPA fallback to extensionless paths
- **Why:**
  - **B** — `TestMissingAssetDoesNotFallBack` **failed** on the first implementation, where the fallback was unconditional
  - **C** — serving HTML in answer to a `.js` request makes the browser report `Unexpected token '<'`, which points nowhere near the stale asset hash that caused it

### Change #0006
:121-127
```go
+ if hashed {
+ 	w.Header().Set("Cache-Control", "public, max-age=31536000, immutable")
+ } else {
+ 	w.Header().Set("Cache-Control", "no-cache")
+ }
```

- **What:**
  - cache hashed assets forever, never cache the shell
- **Why:**
  - **C** — a change renames a hashed asset, but `index.html` keeps its name and is what points at the new hashes

### Change #0007
:24-38
```go
+ const notBuilt = `<!doctype html>
+ <h1>Frontend not built</h1>
+ <pre>./scripts/build-frontend.sh &amp;&amp; go build ./...</pre>`
```

- **What:**
  - serve an explanatory 503 when no bundle is embedded
- **Why:**
  - **B** — a fresh clone has never run the build script
  - **C** — a blank 404 on a healthy server with a working API is the confusing case; saying so costs one const

---

#### main.go

### Change #0008
:72-78
```go
+ mux.HandleFunc("/api/", func(w http.ResponseWriter, r *http.Request) {
+ 	w.Header().Set("Content-Type", "application/json")
+ 	w.WriteHeader(http.StatusNotFound)
+ 	w.Write([]byte(`{"error":"not found"}`))
+ })
```

- **What:**
  - reserve the `/api/` prefix although no API exists yet
- **Why:**
  - **B** — ticket step 2: API and static routing must not collide
  - **C** — without it Go's mux sends `/api/anything` to the catch-all, so a typo'd endpoint returns the app shell with status 200 — which `fetch()` surfaces as a JSON parse error

### Change #0009
:80-84
```go
+ bundle, err := frontendFS(cfg.FrontendDir)
+ if err != nil { log.Fatalf("frontend: %v", err) }
+ mux.Handle("/", frontend.New(bundle))
```

- **What:**
  - mount the SPA handler as the catch-all
- **Why:**
  - **C** — registered last conceptually; Go's mux prefers the more specific `/api/` and `/healthz` patterns regardless of order

### Change #0010
:129-145
```go
+ func frontendFS(dir string) (fs.FS, error) {
+ 	if dir != "" { return os.DirFS(dir), nil }
+ 	return fs.Sub(web.Dist, "dist")
+ }
```

- **What:**
  - choose the disk directory when set, else the embedded bundle
- **Why:**
  - **B** — deployed binaries carry their own bundle; dev wants a rebuild visible without rebuilding Go
  - **C** — `fs.Sub` strips `dist/` so `index.html` sits at the root the handler expects

---

#### internal/config/config.go

### Change #0011
:22-26, :40
```go
+ 	// FrontendDir overrides the embedded bundle with a directory on disk.
+ 	FrontendDir string
  ...
+ 		FrontendDir:      os.Getenv("FRONTEND_DIR"),
```

- **What:**
  - read `FRONTEND_DIR`, empty meaning "use the embedded bundle"
- **Why:**
  - **C** — empty-as-default keeps deployment the zero-config path

---

#### scripts/build-frontend.sh **(new file)**

### Change #0012
:14-22
```sh
+ FRONTEND_REPO="${FRONTEND_REPO:-../LaminarFlow-Frontend}"
+ if [ ! -d "$FRONTEND_REPO" ]; then
+ 	echo "✗ No frontend checkout at $FRONTEND_REPO" >&2
+ 	exit 1
+ fi
```

- **What:**
  - locate the frontend checkout, defaulting to the sibling layout
- **Why:**
  - **B** — the two repos stay separate; this only moves a build artifact
  - **C** — a default, not an assumption — nothing in the Go code depends on the layout

### Change #0013
:38-44
```sh
+ rm -rf web/dist
+ mkdir -p web/dist
+ touch web/dist/.gitkeep
+ cp -R "$FRONTEND_REPO/dist/." web/dist/
```

- **What:**
  - clear before copying, and restore the marker
- **Why:**
  - **C** — without the clear, an asset deleted from the frontend survives here forever; without the marker, a clean checkout will not compile

---

#### internal/frontend/frontend_test.go **(new file)**

### Change #0014
:76-88
```go
+ func TestMissingAssetDoesNotFallBack(t *testing.T) {
+ 	rec := get(t, h, "/assets/index-OldHash.js")
+ 	if strings.Contains(rec.Body.String(), "<div id=root>") {
+ 		t.Error("a missing asset returned the app shell")
+ 	}
+ }
```

- **What:**
  - assert a stale asset hash 404s instead of returning HTML
- **Why:**
  - **B** — **this test failed first and drove Change #0005**
  - **C** — it is the difference between a useful 404 and a misleading JS syntax error

### Change #0015
:90-104
```go
+ func TestCacheHeaders(t *testing.T) {
+ 	// asset -> immutable ; shell and client routes -> no-cache
+ }
```

- **What:**
  - lock the caching contract for both file classes
- **Why:**
  - **C** — a cached shell keeps requesting asset hashes that no longer exist

---

#### COMMANDS.md · README.md · .env.example

### Change #0016
COMMANDS.md :15-32
```diff
+ ## Build the frontend into this repo, then embed it
+     ./scripts/build-frontend.sh && go build ./...
+ ## Run the server against a frontend bundle on disk
+     FRONTEND_DIR=../LaminarFlow-Frontend/dist go run .
```

- **What:**
  - document both the embed path and the dev override
- **Why:**
  - **C** — the embed step is invisible otherwise and will silently serve a stale bundle

---

## Frontend — LaminarFlow-Frontend

#### vite.config.ts

### Change #0017
:13
```ts
+ const apiTarget = process.env.VITE_API_TARGET ?? 'http://localhost:8080'
```

- **What:**
  - make the backend address configurable with a working default
- **Why:**
  - **C** — hardcoding 8080 breaks anyone running the backend on another port

### Change #0018
:17-24
```ts
+ server: {
+   proxy: {
+     '/api': { target: apiTarget, changeOrigin: true },
+     '/healthz': { target: apiTarget, changeOrigin: true },
+   },
+ },
```

- **What:**
  - proxy the backend's paths from the Vite dev server
- **Why:**
  - **B** — same-origin holds in production but not under `npm run dev`; without this, dev needs absolute URLs and CORS — the split-origin setup the decision exists to remove
  - **C** — `/healthz` needs its own entry because health endpoints sit at the root, not under the API prefix

---

#### README.md

### Change #0019
:1-18
```diff
- # React + TypeScript + Vite
- This template provides a minimal setup...
+ # LaminarFlow — Frontend
+ ## Same-origin deployment
+ Application code should always use relative URLs (`fetch('/api/...')`),
+ never an absolute backend URL.
```

- **What:**
  - replace the template preamble with the same-origin contract
- **Why:**
  - **B** — the rule is easy to break and invisible until auth fails cross-site
  - **C** — template notes kept below, marked as inherited

---

## Verification

### Backend gate
`./scripts/test.sh` green — gofmt, build, vet, staticcheck, and all tests against a throwaway Postgres.

> **staticcheck caught one:** a doc line beginning `// go:embed` reads as a malformed compiler directive (SA9009). Reworded. This is the second time the lint-before-test ordering in `scripts/test.sh` masked the test run — documented in `docs/adr/0001`.

### End-to-end, real bundle, one process, one port

| Request to `:8099` | Result |
|---|---|
| `/` | app shell, `Cache-Control: no-cache` |
| `/assets/index-CP6jzYRJ.js` | 200, `text/javascript`, `immutable` |
| `/projects/42` | 200 `text/html` — SPA fallback |
| `/assets/index-STALE.js` | **404**, not HTML |
| `/api/nope` | `{"error":"not found"}`, 404 |
| `/healthz`, `/healthz/db` | `{"status":"ok"}` |

The same run exercised `MIGRATE_ON_STARTUP=true` from LAM-10, applying all four migrations at boot.

### Vite proxy, both servers running

| Request to `:5173` | Result |
|---|---|
| `/` | Vite dev server, app HTML |
| `/healthz` | `{"status":"ok"}` — reached the backend |
| `/healthz/db` | `{"status":"ok"}` — reached Postgres |
| `/api/nope` | `{"error":"not found"}`, 404 — backend's JSON, not Vite's HTML |

That last row confirms the proxy takes precedence over Vite's own SPA fallback, so a real API 404 is not masked by an HTML page.

---

## Follow-up actions

1. **Merge both PRs** — backend [#7](https://github.com/Willpatbarr/LaminarFlow-Backend/pull/7), frontend [#2](https://github.com/Willpatbarr/LaminarFlow-Frontend/pull/2). Both CI green.
2. **Set LAM-28 → Done** once merged.
3. **Adopt the dev database** — still outstanding from LAM-10; the server refuses to start against it until:
   ```bash
   set -a && source .env && set +a && go run ./cmd/migrate baseline && go run ./cmd/migrate up
   ```
4. **Next ticket: LAM-29** (containerize) — LAM-28 was blocking it. It owns the multi-stage build this ticket deliberately left out.
5. **Switch gh back** — `gh auth switch --user willbarr_church`

## Deliberately out of scope

**React Router.** The backend's SPA fallback is in place and tested, but the frontend is still the unmodified Vite template — there is no route to add yet. Worth its own ticket once there is a real screen.
