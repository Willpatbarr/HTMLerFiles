# LAM-10 — Versioned Migrations Pipeline

**Branch:** `LAM-10-B` → `E-LAM-0001-B` · **PR:** [#6](https://github.com/Willpatbarr/LaminarFlow-Backend/pull/6) · **Commit:** `93a4191`
**Status:** CI green, awaiting merge

18 files changed, 1287 insertions(+), 95 deletions(-)

---

#### migrations/embed.go **(new file)**

### Change #0001
:1-14
```go
+ package migrations
+
+ import "embed"
+
+ //go:embed *.sql
+ var FS embed.FS
```

- **What:**
  - make `migrations/` a Go package that embeds its own `.sql` files
- **Why:**
  - **B** — LAM-29 ships a container image; the binary must carry its schema history
  - **C** — `go:embed` cannot reference `../`, so the embed has to live in the migrations directory itself

---

#### internal/migrate/migrate.go **(new file)**

### Change #0002
:47-53
```go
+ type Migration struct {
+ 	Version int64
+ 	Name    string
+ 	Up      string
+ 	Down    string
+ }
```

- **What:**
  - one struct holding both halves of a reversible change
- **Why:**
  - **B** — ticket step 4 requires `up`/`down` to both work
  - **C** — one file per migration means one parse produces both directions

### Change #0003
:64-131
```go
+ func Load(fsys fs.FS) ([]Migration, error) {
+ 	...
+ 	return nil, fmt.Errorf("migration %q does not match NNNN_name.sql", e.Name())
+ 	...
+ 	return nil, fmt.Errorf("migrations %q and %q share version %d", prev, e.Name(), version)
+ }
```

- **What:**
  - fail on malformed filename, duplicate version, or missing marker
- **Why:**
  - **B** — a skipped migration reports "up to date" while a table is missing
  - **C** — silently ignoring an unreadable file is the worst available behavior

---

#### internal/migrate/runner.go **(new file)**

### Change #0004
:25-62
```go
+ func withLock(ctx, pool, fn) error {
+ 	conn.Exec(ctx, `SELECT pg_advisory_lock($1)`, lockID)
+ 	defer func() { conn.Exec(context.WithoutCancel(ctx), `SELECT pg_advisory_unlock($1)`, lockID) }()
+ 	conn.Exec(ctx, createTable)
+ 	return fn(ctx, conn)
+ }
```

- **What:**
  - take a Postgres advisory lock before any schema operation
  - `WithoutCancel` on unlock
- **Why:**
  - **B** — two servers can start at once on the self-hosted target
  - **C** — without it both read the same pending list and the loser fails with a confusing "table already exists"

### Change #0005
:186-219
```go
+ func apply(ctx, conn, m, sql string, up bool) error {
+ 	tx, _ := conn.Begin(ctx)
+ 	defer tx.Rollback(ctx)
+ 	tx.Exec(ctx, sql)                                    // no args -> simple protocol
+ 	tx.Exec(ctx, `INSERT INTO schema_migrations ...`)    // or DELETE on down
+ 	return tx.Commit(ctx)
+ }
```

- **What:**
  - run the migration SQL and its bookkeeping row in one transaction
- **Why:**
  - **B** — a half-applied migration is the worst state to debug on a home server
  - **C** — Postgres has transactional DDL, so schema and `schema_migrations` cannot disagree

### Change #0006
:139-166
```go
+ // Newest first: a migration can depend on an earlier one.
+ for i := len(migrations) - 1; i >= 0 && len(done) < n; i-- {
+ 	if m.Down == "" {
+ 		return fmt.Errorf("%04d_%s: %w", m.Version, m.Name, ErrIrreversible)
+ 	}
```

- **What:**
  - roll back newest-first; refuse a migration with an empty `Down`
- **Why:**
  - **B** — `down` must not report success having changed nothing
  - **C** — `child` references `parent`, so unwinding in application order would fail on the FK

### Change #0007
:247-274
```go
+ func Baseline(ctx, pool, migrations) ([]Migration, error) {
+ 	if len(applied) > 0 {
+ 		return errors.New("database already has recorded migrations - baseline is only for adopting a hand-built schema")
+ 	}
```

- **What:**
  - record all migrations as applied without running any
  - refuse once any history exists
- **Why:**
  - **B** — the dev database has tables built by hand across LAM-2 → LAM-5
  - **C** — `up` there would `CREATE TABLE document` a second time and fail

---

#### internal/config/config.go

### Change #0008
:15-21, :32-35
```go
  type Config struct {
  	Port        string
  	DatabaseURL string
+
+ 	// off by default: an unattended container restart must not silently
+ 	// reshape the database.
+ 	MigrateOnStartup bool
  }
  ...
- 		Port:        getenv("PORT", "8080"),
- 		DatabaseURL: os.Getenv("DATABASE_URL"),
+ 		Port:             getenv("PORT", "8080"),
+ 		DatabaseURL:      os.Getenv("DATABASE_URL"),
+ 		MigrateOnStartup: os.Getenv("MIGRATE_ON_STARTUP") == "true",
```

- **What:**
  - read `MIGRATE_ON_STARTUP`, defaulting to false
- **Why:**
  - **B** — chosen behavior: env-gated, default off
  - **C** — `== "true"` means any other value is off, which is the safe direction

---

#### main.go

### Change #0009
:34-37
```go
+ 	if err := ensureSchema(context.Background(), pool, cfg.MigrateOnStartup); err != nil {
+ 		log.Fatalf("schema: %v", err)
+ 	}
```

- **What:**
  - gate startup on schema state, after the pool connects
- **Why:**
  - **B** — a mismatched schema must not reach a request
  - **C** — placed after `db.Connect` because it needs the pool

### Change #0010
:68-103
```go
+ func ensureSchema(ctx, pool, autoApply bool) error {
+ 	if autoApply { ...migrate.Up... }
+ 	pending, err := migrate.Pending(ctx, pool, loaded)
+ 	if len(pending) > 0 {
+ 		return fmt.Errorf("%d migration(s) pending, oldest %04d_%s - run `migrate up`, or set MIGRATE_ON_STARTUP=true", ...)
+ 	}
```

- **What:**
  - apply when opted in; otherwise fail fast naming the fix
- **Why:**
  - **B** — ticket step 3 asks for a startup path
  - **C** — a missing column otherwise surfaces as a random 500, not a startup error

---

#### migrations/0001_document.sql · 0002_search_index.sql · 0003_workspace.sql

### Change #0011
:6-7, and section markers throughout
```sql
- -- Temporary raw SQL until E-LAM-0002 lands a migration runner.
+
+ -- +migrate Up
  CREATE TABLE document (...);
+
+ -- +migrate Down
+
+ DROP TABLE document;
```

- **What:**
  - add `Up`/`Down` markers to all three; drop the stale epic reference
- **Why:**
  - **B** — LAM-10 *is* that runner, and it lives in E-LAM-0003, not E-LAM-0002
  - **C** — the parser requires both markers

### Change #0012
0003 only, Down section
```sql
+ -- Order matters: the column references workspace, so it goes first.
+ ALTER TABLE document DROP COLUMN workspace_id;
+ DROP TABLE workspace;
```

- **What:**
  - drop the FK column before the table it points at
- **Why:**
  - **C** — reverse order fails on the foreign key

---

#### migrations/0004_table_comments.sql **(new file)**

### Change #0013
:16-24
```sql
+ -- +migrate Up
+ COMMENT ON TABLE document IS 'Source of truth for document field data...';
+ COMMENT ON TABLE search_index IS 'Derived from document.body and disposable...';
+
+ -- +migrate Down
+ COMMENT ON TABLE document IS NULL;
```

- **What:**
  - the trivial first migration the ticket asks for
- **Why:**
  - **B** — ticket step 4: prove `up`/`down` without risking anything
  - **C** — comments move no data and tighten no constraint, so the rollback is genuinely total

---

#### cmd/migrate/main.go **(new file)**

### Change #0014
:1-111
```go
+ //	migrate up          apply every pending migration
+ //	migrate down [n]    roll back the n most recent (default 1)
+ //	migrate status      list every migration and whether it has run
+ //	migrate baseline    record all as applied, running none
```

- **What:**
  - four subcommands over the runner
- **Why:**
  - **B** — ticket step 3 asks for a local dev command
  - **C** — migrations are embedded, so it needs a database URL and nothing else

---

#### internal/dbtest/dbtest.go **(new file)**

### Change #0015
:36-79
```go
+ func Create(ctx) (dsn string, cleanup func(), err error)
```

- **What:**
  - extract the throwaway-database bootstrap out of `main_test.go`
- **Why:**
  - **B** — `internal/migrate` needs a real database too
  - **C** — a second copy of that `TestMain` is the same duplication this ticket removes

---

#### internal/document/main_test.go

### Change #0016
:103-120 (was :95-127)
```go
- files, err := filepath.Glob(filepath.Join("..", "..", "migrations", "*.sql"))
- sort.Strings(files)
- for _, f := range files { conn.PgConn().Exec(ctx, string(sql)).ReadAll() }
+ loaded, err := migrate.Load(migrations.FS)
+ if _, err := migrate.Up(ctx, pool, loaded); err != nil { ... }
```

- **What:**
  - replace the parallel migration loop with a call to the real runner
- **Why:**
  - **B** — **this is the core reason hand-rolling won.** The repo was carrying two migration implementations
  - **C** — a migration the tests could apply but the runner could not would stay invisible until deploy

---

#### internal/migrate/runner_test.go · migrate_test.go **(new files)**

### Change #0017
runner_test.go :294-323
```go
+ func TestRealMigrationsApplyToAnEmptyDatabase(t *testing.T) {
+ 	applied, err := Up(ctx, pool, loaded)
+ 	reverted, err := Down(ctx, pool, loaded, len(loaded))
```

- **What:**
  - apply and reverse the *real* shipped migrations every run
- **Why:**
  - **B** — this is what would have caught 0003's original `NOT NULL` break
  - **C** — fixture migrations prove the runner, not the migrations

### Change #0018
runner_test.go :227-261
```go
+ func TestFailedMigrationRecordsNothing(t *testing.T) {
+ 	// duplicate CREATE TABLE in one migration
+ 	if recorded != 0 { t.Errorf("schema_migrations has %d rows after a failed migration", recorded) }
```

- **What:**
  - assert a half-failed migration leaves neither table nor version row
- **Why:**
  - **C** — the transactional guarantee is only real if the runner puts both in one transaction

---

#### COMMANDS.md · .env.example · migrations/README.md

### Change #0019
COMMANDS.md :23-41
```diff
- ## Apply a migration to the real database
-     psql "$DATABASE_URL" -1 -f migrations/0003_workspace.sql
+ ## Apply pending migrations to the real database
+     set -a && source .env && set +a && go run ./cmd/migrate up
+ ## Roll back / status / baseline ...
```

- **What:**
  - replace the hand-applied `psql -f` recipe with the runner commands
- **Why:**
  - **B** — hand-applying is exactly what LAM-10 removes
  - **C** — the old command named a specific file that would drift

### Change #0020
migrations/README.md — new **File format**, **Guarantees**, **Adopting a hand-built database** sections; **Expand and contract** kept verbatim

- **What:**
  - document the marker format, the transactional/lock guarantees, and the `baseline` path
- **Why:**
  - **B** — the dev database can't start until it's baselined
  - **C** — the old "there is no migration runner yet — LAM-10 owns that" line is now false

---

## Follow-up actions

1. **Merge [PR #6](https://github.com/Willpatbarr/LaminarFlow-Backend/pull/6)** — CI green, both checks passing.
2. **Adopt the dev database** — it has the three tables hand-built with no `schema_migrations`, so the server will refuse to start until:
   ```bash
   set -a && source .env && set +a && go run ./cmd/migrate baseline && go run ./cmd/migrate up
   ```
3. **Stale Linear statuses** — LAM-5 and LAM-38 still `In Progress` despite merging; LAM-10 still `Backlog`.
4. **Switch gh back** — `gh auth switch --user willbarr_church`
