# LAM-29 — Containerize for Portable Deployment

**Branch:** `LAM-29-B` → **`LAM-28-B`** (not the epic branch — see below) · PR [#8](https://github.com/Willpatbarr/LaminarFlow-Backend/pull/8)
**Commit:** `071d017` · 11 files, +668 / −2

> ### Two things to know first
>
> **1. This is based on `LAM-28-B`, not `E-LAM-0001-B`.** You said you'd merged both LAM-28 PRs, but GitHub still shows backend [#7](https://github.com/Willpatbarr/LaminarFlow-Backend/pull/7) and frontend [#2](https://github.com/Willpatbarr/LaminarFlow-Frontend/pull/2) as `state=OPEN, merged=null`, and neither epic branch contains them. LAM-29 needs LAM-28's `web/dist` embedding to build at all, so I stacked on `LAM-28-B` rather than merging your PRs for you. **Merge #7 and #2 first**; GitHub will retarget #8 to the epic branch.
>
> **2. Docker is not installed on this machine.** `docker: command not found`. The image was never built or run locally. CI is the verification — both repos are public, so it checks out the frontend, builds both architectures, and runs the image against real Postgres. The Verification section says exactly what is and isn't proven.

---

## Backend — LaminarFlow-Backend

#### Dockerfile **(new file)**

### Change #0001
:17-18
```dockerfile
+ ARG NODE_VERSION=26
+ ARG GO_VERSION=1.27
```

- **What:**
  - take the toolchain versions as build arguments
- **Why:**
  - **B** — `.nvmrc` and `go.mod` already own these versions
  - **C** — hardcoding them here would make the Dockerfile a third place a version has to be kept in step; the wrapper script and CI both read the real sources

### Change #0002
:27-38
```dockerfile
+ FROM --platform=$BUILDPLATFORM node:${NODE_VERSION}-alpine AS frontend
+ WORKDIR /src
+ COPY --from=frontendsrc . .
+ RUN rm -rf node_modules dist \
+     && npm ci \
+     && npm run build
```

- **What:**
  - build the frontend from a BuildKit **named context**, not a clone
  - delete `node_modules` and `dist` before installing
- **Why:**
  - **B** — the frontend is a separate repo; a named context keeps them independent with no network or credentials at build time
  - **C** — copying the checkout wholesale drags in a `node_modules` built for the host's OS and architecture, and the named context has no `.dockerignore` of its own to rely on
  - **C** — `$BUILDPLATFORM` pins this to the build machine: the output is plain JS/CSS with no architecture, so emulating the target would cost minutes and buy nothing

### Change #0003
:46-71
```dockerfile
+ FROM --platform=$BUILDPLATFORM golang:${GO_VERSION}-alpine AS build
+ COPY go.mod go.sum ./
+ RUN go mod download
+ COPY . .
+ COPY --from=frontend /src/dist ./web/dist
+ ARG TARGETARCH
+ RUN CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} \
+         go build -trimpath -ldflags="-s -w" -o /out/laminarflow .
```

- **What:**
  - cross-compile for `$TARGETARCH` from a native-platform builder
  - place the bundle where `web/embed.go` expects it
- **Why:**
  - **B** — step 2 wants amd64 and arm64 from one build
  - **C** — Go cross-compiles for free, so **nothing runs under QEMU** and the arm64 build costs what amd64 does
  - **C** — `CGO_ENABLED=0` is what makes the binary static, which is what lets the final stage have no libc at all

### Change #0004
:81-100
```dockerfile
+ FROM gcr.io/distroless/static-debian12:nonroot
+ COPY --from=build /out/laminarflow /laminarflow
+ COPY --from=build /out/migrate /migrate
+ COPY --from=build /out/reindex /reindex
+ USER nonroot:nonroot
+ HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
+     CMD ["/laminarflow", "healthcheck"]
```

- **What:**
  - ship on distroless, as nonroot, with the operator tools alongside the server
- **Why:**
  - **B** — a service exposed to a tailnet should have no shell and no package manager
  - **C** — `/migrate` is **not optional**: a database built before LAM-10 needs `migrate baseline` before the server will start, and there is no shell here to fetch a binary with
  - **C** — the `HEALTHCHECK` runs the binary because distroless has no curl

---

#### .dockerignore **(new file)**

### Change #0005
:10-13
```gitignore
+ web/dist/*
+ !web/dist/.gitkeep
```

- **What:**
  - exclude the local bundle but keep the directory marker
- **Why:**
  - **C** — the frontend stage supplies the real bundle; the marker stays so `web/embed.go`'s `go:embed` still has a directory if that stage is ever skipped

---

#### main.go

### Change #0006
:34-36
```go
+ if len(os.Args) > 1 && os.Args[1] == "healthcheck" {
+ 	os.Exit(healthcheck())
+ }
```

- **What:**
  - dispatch the healthcheck before anything else in `main`
- **Why:**
  - **C** — handled first so it never opens a database connection or reads config it has no business requiring

### Change #0007
:158-183
```go
+ func healthcheck() int {
+ 	port := os.Getenv("PORT")
+ 	if port == "" { port = config.DefaultPort }
+ 	client := &http.Client{Timeout: healthcheckTimeout}
+ 	resp, err := client.Get("http://127.0.0.1:" + port + "/healthz")
+ 	...
+ }
```

- **What:**
  - ask the running server over loopback and return a process exit code
- **Why:**
  - **B** — a distroless image has no curl to run a `HEALTHCHECK` with
  - **C** — hits `/healthz`, not `/healthz/db`: a database blip should not make a supervisor restart a healthy process

---

#### internal/config/config.go

### Change #0008
:29-32
```go
+ // DefaultPort is the port used when PORT is unset.
+ const DefaultPort = "8080"
```

- **What:**
  - export the port default
- **Why:**
  - **C** — the healthcheck needs it without calling `config.Load()`, which would demand a `DATABASE_URL` it does not use

---

#### scripts/build-image.sh **(new file)**

### Change #0009
:35-45
```sh
+ NODE_VERSION=$(tr -d ' \t\nv' < "$FRONTEND_REPO/.nvmrc")
+ GO_VERSION=$(awk '/^go /{split($2,v,"."); print v[1]"."v[2]; exit}' go.mod)
```

- **What:**
  - read the toolchain versions from the files that declare them
- **Why:**
  - **B** — `.nvmrc` is already the single source of truth CI uses
  - **C** — `go.mod` says `go 1.27.0` but the golang image tag wants `1.27`

### Change #0010
:56-66
```sh
+ case "$PLATFORMS" in
+     *,*)
+         echo "==> multi-platform build (${PLATFORMS}) - not loaded locally"
+         ;;
+     *) set -- "$@" --load ;;
+ esac
```

- **What:**
  - skip `--load` when more than one platform is requested
- **Why:**
  - **C** — the local image list holds one architecture per tag, so a multi-platform `--load` fails; without this the script would break on exactly the command the ticket asks for

---

#### deploy/docker-compose.yml **(new file)**

### Change #0011
:21, :26-31
```yaml
+ depends_on:
+   db:
+     condition: service_healthy
+ environment:
+   DATABASE_URL: postgres://...@db:5432/...
+   MIGRATE_ON_STARTUP: ${MIGRATE_ON_STARTUP:-true}
```

- **What:**
  - wait for the database's healthcheck, not just its container
  - default migrations to on in the deployed stack only
- **Why:**
  - **C** — the server refuses to start against an unreachable database, so starting it first would crash-loop until Postgres caught up
  - **B** — an operator pulling a new image expects the schema to follow; the binary still defaults to off, so nothing else inherits this

### Change #0012
:36 and the `db` service
```yaml
+ - "${APP_BIND:-127.0.0.1}:${APP_PORT:-8080}:${APP_PORT:-8080}"
```

- **What:**
  - publish the app on loopback by default; publish no database port at all
- **Why:**
  - **B** — Tailscale routes to loopback, a LAN does not, so this is reachable over the tailnet without being exposed locally
  - **C** — there is no reason to put Postgres on the tailnet; `docker compose exec db psql` covers access

---

#### deploy/.env.example · docs/deploy.md **(new files)**

### Change #0013
docs/deploy.md :150-177
```txt
+ ## Portability audit
+ | Value | Where it comes from | Default |
+ | Listen port | PORT | 8080 |
+ | Database connection | DATABASE_URL | none — refuses to start |
+ ...
+ Not present, and deliberately so: no BASE_URL, no storage path.
```

- **What:**
  - the step 5 audit, plus what is deliberately absent
- **Why:**
  - **B** — ticket step 5 asks to confirm nothing assumes this machine
  - **C** — naming the *absent* settings matters as much as the present ones, so the next reader does not add them speculatively

---

#### .github/workflows/ci.yml

### Change #0014
:54-72
```yaml
+ image:
+   env:
+     FRONTEND_REF: E-LAM-0001-F
+   steps:
+     - uses: actions/checkout@v7
+       with: { path: backend }
+     - uses: actions/checkout@v7
+       with:
+         repository: Willpatbarr/LaminarFlow-Frontend
+         ref: ${{ env.FRONTEND_REF }}
+         path: frontend
```

- **What:**
  - check out both repos so the named context has a source
- **Why:**
  - **B** — **this is the only verification available**, since Docker is not installed locally
  - **C** — both repos are public, so the default token suffices — no PAT needed

### Change #0015
:99, :116-117
```yaml
+ platforms: linux/amd64,linux/arm64
+ push: false
  ...
+ platforms: linux/amd64
+ load: true
```

- **What:**
  - build both architectures, then a loadable single-arch image
- **Why:**
  - **C** — multi-platform images cannot be loaded, so proving they build and proving one runs are necessarily two steps

### Change #0016
:124-180
```sh
+ curl -sf localhost:8080/healthz | grep -q '"status":"ok"'
+ curl -sf localhost:8080/ | grep -qi '<!doctype html'
+ test "$(curl -s -o /dev/null -w '%{http_code}' localhost:8080/assets/nope-STALE.js)" = "404"
+ docker exec lf-app /laminarflow healthcheck
+ test "$(docker inspect -f '{{.Config.User}}' laminarflow:ci)" = "nonroot:nonroot"
```

- **What:**
  - run the built image against real Postgres and assert its behaviour
- **Why:**
  - **B** — a Dockerfile that builds but produces a broken image would otherwise pass
  - **C** — covers the LAM-28 behaviours too, so the container path and the local path cannot diverge

---

#### COMMANDS.md · README.md

### Change #0017
COMMANDS.md :23-38
```sh
+ ## Build the container image
+     ./scripts/build-image.sh
+     PLATFORMS=linux/amd64,linux/arm64 ./scripts/build-image.sh
+ ## Adopt a hand-built database inside the container
+     docker compose -f deploy/docker-compose.yml run --rm app /migrate baseline
```

- **What:**
  - document the image build and the in-container adoption step
- **Why:**
  - **C** — a bare `docker build` here fails on the named context with a message that does not explain itself

---

## Decisions I made while you were away

Each with the alternative, in case you want to change one.

| # | Decision | Why | Alternative if you disagree |
|---|----------|-----|------------------------------|
| 1 | **Branch off `LAM-28-B`**, not the epic | LAM-28's PRs are still open and LAM-29 cannot build without them | Merge #7 first, then rebase this onto `E-LAM-0001-B` |
| 2 | **Distroless base** (`static-debian12:nonroot`) | No shell or package manager on a tailnet-exposed service; CA certs and nonroot included | `alpine:3` for `docker exec sh` — one line in the final stage. `docs/deploy.md` documents debugging without one |
| 3 | **BuildKit named context** for the frontend | Repos stay independent; no network or credentials at build time | Clone in-build (needs auth for a future private repo), or require `build-frontend.sh` first (build no longer self-contained) |
| 4 | **Ship `/migrate` and `/reindex`** in the image | Adopting a pre-LAM-10 database is impossible from a shell-less container otherwise | Server-only image, and run migrations from outside |
| 5 | **`healthcheck` subcommand** on the binary | Distroless has no curl | No `HEALTHCHECK` at all, or an Alpine base |
| 6 | **Separate `deploy/docker-compose.yml`** | A developer running `docker compose up` should not start a server | Extend the root file with compose profiles |
| 7 | **`APP_BIND` defaults to `127.0.0.1`** | Tailscale routes to loopback; a LAN does not | `0.0.0.0` if you want LAN access by default |
| 8 | **No published Postgres port** in deploy | No reason to put a database on the tailnet | Publish 5432 if you want external psql |
| 9 | **`MIGRATE_ON_STARTUP=true` in deploy only** | Pulling a new image should bring the schema; the binary still defaults to off | Keep it off and run `/migrate up` as a deliberate step |
| 10 | **CI pins `FRONTEND_REF: E-LAM-0001-F`** | `main` is still the initial commit with no `package.json` | Change to `main` once the epic lands — it is one env line |
| 11 | **No `BASE_URL` or storage-path config** | Nothing builds absolute URLs and there are no uploads; unused config is one more thing that can be wrong | Add them now if you'd rather have the slots reserved |
| 12 | **`rm -rf node_modules dist` then `npm ci`** | Named contexts have no `.dockerignore`, so a host `node_modules` would poison the build | Add a `.dockerignore` to the frontend repo and copy selectively — better caching, second PR |

---

## Verification

### What I could prove locally (no Docker on this machine)

| Check | Result |
|---|---|
| `./scripts/test.sh` — full gate | green |
| Static cross-compile `linux/amd64` | `ELF 64-bit LSB executable, x86-64` |
| Static cross-compile `linux/arm64` | `ELF 64-bit LSB executable, ARM aarch64` |
| `healthcheck` with no server | exit 1, names the refused connection |
| `healthcheck` against a live server | exit 0 |
| `healthcheck` with `DATABASE_URL` unset | exit 0 — confirms it needs no database config |
| `healthcheck` port default | falls back to 8080 |
| Version parsing → real image tags | `node:26-alpine`, `golang:1.27-alpine` |
| CI workflow YAML | parses, 2 jobs, 7 steps in `image` |

### What only CI could prove — **all 4 checks passed**

The Dockerfile itself was never executed here. CI ran it, and the log confirms:

```
node 26, go 1.27
#15 [linux/amd64 stage-2 1/4] FROM gcr.io/distroless/static-debian12:nonroot
#17 [linux/arm64 stage-2 1/4] FROM gcr.io/distroless/static-debian12:nonroot
```

Then, inside the running container against real Postgres:

```
2026/09/04 01:28:49 applied migration 0002_search_index
2026/09/04 01:28:49 applied migration 0003_workspace
2026/09/04 01:28:49 applied migration 0004_table_comments
2026/09/04 01:28:49 laminarflow backend listening on :8080
--- healthz ---
--- healthz/db (migrations applied on startup) ---
--- embedded frontend is served ---
--- SPA fallback ---
--- missing asset 404s rather than returning HTML ---
--- unknown API route returns JSON, not the app shell ---
--- the image's own healthcheck command ---
--- operator tools are present ---
applied  0001_document
applied  0002_search_index
applied  0003_workspace
applied  0004_table_comments
--- runs as nonroot ---
✓ image smoke test passed
```

Every assertion is a hard failure if it does not hold — `grep -q` or `test`, under `set -e`.

### Still not covered

* **The arm64 image was built but never run.** CI has no arm64 runner, and running it under emulation would prove less than it costs. The Pi is the first place that image executes.
* **The Apple Silicon build path is untested.** CI builds on amd64, so `scripts/build-image.sh` on your Mac is a genuinely new path.

---

## Follow-up actions

1. **Merge LAM-28 first** — backend [#7](https://github.com/Willpatbarr/LaminarFlow-Backend/pull/7), frontend [#2](https://github.com/Willpatbarr/LaminarFlow-Frontend/pull/2). Both were green and are still open.
2. **Then merge [#8](https://github.com/Willpatbarr/LaminarFlow-Backend/pull/8)** and set LAM-29 → Done.
3. **Build the image yourself once** — I never ran `docker build`. `./scripts/build-image.sh` on your Mac is the first real test on Apple Silicon.
4. **Adopt the dev database** — still outstanding from LAM-10:
   ```bash
   set -a && source .env && set +a && go run ./cmd/migrate baseline && go run ./cmd/migrate up
   ```
5. **Change `FRONTEND_REF` to `main`** in `.github/workflows/ci.yml` once E-LAM-0001-F lands on main.
6. **Switch gh back** — `gh auth switch --user willbarr_church`
