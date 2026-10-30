# Go Learning Plan — Task Manager (CLI → Backend Service)

> One project, grown in 7 stages. You write every line of code. AI explains concepts, gives hints, and reviews — it never writes the solution.

**Start date:** 2026-10-30
**Pace:** ~1 hr/day on weekdays + one longer weekend session
**Estimated total:** 6–7 weeks
**Background leverage:** You know Python/Django, so each stage lists the Python equivalent to anchor the new idea.

---

## Stage 0 — Setup (Day 1, ~1 hr)

Go is **not installed** on this machine yet.

- [ ] Install: `brew install go`, then confirm with `go version`
- [ ] Editor: VS Code + the official **Go** extension (it installs `gopls`, the language server)
- [ ] Create the module: `go mod init github.com/<you>/taskmgr` (`go.mod` plays the role of `pyproject.toml`)
- [ ] Write a hello world, then `go run .` and `go build`
- [ ] Do the first 3 sections of [A Tour of Go](https://go.dev/tour)
- [ ] Bookmark [Go by Example](https://gobyexample.com) and [Effective Go](https://go.dev/doc/effective_go)

**Done when:** `go run .` prints something and you can explain what `package main` and `func main()` mean.

---

## Stage 1 — CLI basics (Week 1, ~4 days)

**Build:**
```
$ task add "buy milk"
Added task 1: buy milk
$ task list
[ ] 1  buy milk
$ task done 1
Marked task 1 as done
```
(Tasks live in memory only, so they're gone when the program exits. That's fine for now.)

| Concept | Python equivalent | Practice |
|---|---|---|
| Variables, `:=`, zero values | `x = 0` | Declare each type and print its zero value |
| Basic types and functions, multiple return values | `def f() -> tuple` | A function returning `(Task, error)` |
| Structs: `Task{ID, Title, Done}` | `@dataclass` | Define `Task` and print it with `%+v` |
| Slices, `append`, `for range` | `list`, `for x in` | Store and loop over `[]Task` |
| `os.Args`, then the `flag` package | `sys.argv`, `argparse` | Parse the subcommand, then add a `--verbose` flag |
| `switch` statement | `match` / if-elif | Dispatch on `add` / `list` / `done` |

**Milestones**
1. Parse `os.Args` and print which subcommand was called
2. Implement `add` and `list` against a slice
3. Implement `done <id>`, including handling a bad id ("task 99 not found")
4. Swap in the `flag` package for any options

**Done when:** all 3 commands work, and an unknown command prints usage help.
**Stretch:** `task delete <id>`, and colored output for done tasks.

---

## Stage 2 — Save to a file (Week 2, ~4 days)

**Build:** tasks persist in `tasks.json` between runs.

| Concept | Python equivalent | Practice |
|---|---|---|
| Errors as values: `if err != nil` | `try/except` | Never ignore an error, not even with `_` |
| Wrapping errors: `fmt.Errorf("...: %w", err)` | `raise X from e` | Add context at each layer |
| `encoding/json`, struct tags `` `json:"title"` `` | `json.dumps`, serializers | Marshal `[]Task`, then unmarshal it back |
| `os.ReadFile`, `os.WriteFile` | `open().read()` | Handle the "file doesn't exist yet" case |
| Packages: `internal/task`, `cmd/task` | Django apps / modules | Move `Task` out of `main` |
| Exported vs unexported (`Task` vs `task`) | `_private` convention | Decide what each package exposes |

**Target layout**
```
taskmgr/
├── go.mod
├── cmd/task/main.go        # CLI entry point
└── internal/task/
    ├── task.go             # Task struct + logic
    └── store.go            # load/save JSON
```

**Done when:** you can add tasks, quit, rerun, and still see them. A corrupted JSON file gives a clear error instead of a panic.
**Stretch:** write to a temp file and then `os.Rename` it, so a crash mid-write can't corrupt your data.

---

## Stage 3 — Methods, interfaces, tests (Week 3, ~5 days) ⭐ most important stage

**Build:** a `Storage` interface with a `FileStorage` implementation, plus a `MemoryStorage` for tests.

```go
// You design the exact method set — this is just the shape
type Storage interface {
    List() ...
    Add(...) ...
    MarkDone(...) ...
}
```

| Concept | Python equivalent | Practice |
|---|---|---|
| Methods on structs | instance methods | `func (t *Task) Complete()` |
| Pointer vs value receivers | (no direct equivalent) | Prove to yourself why a value receiver can't mutate |
| Interfaces, satisfied implicitly | Duck typing / `Protocol` | `FileStorage` and `MemoryStorage` both satisfy `Storage` |
| "Accept interfaces, return structs" | Dependency injection | Your CLI takes a `Storage`, not a `*FileStorage` |
| `go test`, table-driven tests | `pytest.mark.parametrize` | Test each `Storage` method |
| `t.TempDir()` | `tmp_path` fixture | Test `FileStorage` against a real temporary file |

**Milestones**
1. Convert free functions into methods
2. Extract the interface, and have the CLI depend only on it
3. Write `MemoryStorage`, and use it in tests
4. Get `go test ./...` green, then run `go test -cover ./...`

**Done when:** the coverage on `internal/task` is above 70%, and you can explain in your own words why the interface lives where it's *used*, not where it's implemented.
**Stretch:** run one shared test suite against both implementations (a contract test).

---

## Stage 4 — HTTP API (Week 4, ~5 days)

**Build:** the same logic served over HTTP.

| Method | Path | Response |
|---|---|---|
| `GET` | `/tasks` | `200` + JSON list |
| `POST` | `/tasks` | `201` + created task / `400` for bad input |
| `PATCH` | `/tasks/{id}` | `200` / `404` |
| `DELETE` | `/tasks/{id}` | `204` / `404` |

| Concept | Python equivalent | Practice |
|---|---|---|
| `net/http`, `http.HandlerFunc` | Django views | One handler per route |
| Go 1.22+ routing: `mux.HandleFunc("GET /tasks/{id}", ...)`, `r.PathValue("id")` | `urls.py` | No framework needed |
| `json.NewDecoder(r.Body)` / `json.NewEncoder(w)` | DRF serializers | Validate input by hand |
| Middleware: `func(http.Handler) http.Handler` | Django middleware | Logging + request timing |
| Consistent error responses | DRF exception handler | `{"error": "task not found"}` |
| `httptest.NewRecorder` | Django test client | Test every handler |

**New layout addition:** `cmd/server/main.go` and `internal/api/handlers.go`. The CLI and the server share `internal/task`.

**Done when:** you can exercise every endpoint with curl or Bruno, every handler has a test, and bad JSON returns `400`, not `500`.
**Stretch:** recovery middleware that turns a panic into a `500`, and a request-ID middleware.

---

## Stage 5 — Real database (Week 5, ~4 days)

**Build:** a `MongoStorage` that satisfies the *same* `Storage` interface.

| Concept | Python equivalent | Practice |
|---|---|---|
| `go.mongodb.org/mongo-driver` | `pymongo` | CRUD against a local Mongo instance |
| `context.Context` as the first argument everywhere | (no direct equivalent) | Add `ctx` to the `Storage` interface — notice how far that change ripples |
| `context.WithTimeout` | query timeouts | 3s timeout per DB call |
| Config from env vars (`os.Getenv`) | `django-environ` | `STORAGE=file\|mongo`, `MONGO_URI` |
| Dependency wiring in `main` | `settings.py` | Pick the implementation at startup |
| `docker compose` for Mongo | — | One command brings up the DB |

**Key moment:** changing the storage backend should need **zero handler changes**. If you had to edit handlers, revisit Stage 3.

**Done when:** `STORAGE=mongo go run ./cmd/server` works, and the handler tests still pass using `MemoryStorage`.
**Stretch:** integration tests with a build tag (`//go:build integration`).

---

## Stage 6 — Concurrency (Week 6, ~5 days)

**Build:** tasks get an optional `DueAt`. A background worker checks every 30s and "sends" a reminder (a log line) for each due task. Think of it as a tiny Celery Beat.

| Concept | Python equivalent | Practice |
|---|---|---|
| Goroutines: `go f()` | threads / asyncio tasks | Start the worker next to the HTTP server |
| Channels | `queue.Queue` | Worker → notifier pipeline |
| `select` + `time.Ticker` | Celery Beat schedule | Tick, or stop on `ctx.Done()` |
| `sync.WaitGroup` | `thread.join()` | Wait for workers to finish |
| `sync.Mutex` | `threading.Lock` | Make `MemoryStorage` safe under concurrent access |
| `go test -race` | — | Find the data race you *will* have |
| Graceful shutdown: `signal.NotifyContext` + `server.Shutdown(ctx)` | Gunicorn graceful reload | Ctrl+C finishes in-flight requests and stops workers cleanly |

**Milestones**
1. A ticker loop that prints every N seconds and stops cleanly on Ctrl+C
2. A worker that queries due tasks and sends them to a channel
3. A notifier goroutine that reads from the channel
4. Run `go test -race ./...` until it's clean

**Done when:** Ctrl+C logs "shutting down… done" with no goroutine leaks, and `-race` is clean.
**Stretch:** a worker pool of N notifiers, and avoiding duplicate reminders (store `RemindedAt`).

---

## Stage 7 — Production-ready (Week 7, ~4 days)

| Task | Practice |
|---|---|
| Multi-stage Dockerfile | Build in `golang:1.x`, run in `gcr.io/distroless/static` and compare image sizes (aim for <20 MB) |
| `log/slog` structured logging | JSON logs with `request_id`, `method`, `status`, `duration_ms` |
| Health checks | `/healthz` (liveness) and `/readyz` (pings Mongo) |
| Kubernetes | Deployment + Service + ConfigMap + Secret (you already have k8s/helm experience from linkpulse) |
| Makefile | `make test`, `make run`, `make docker` |
| Linting | `go vet ./...` + `golangci-lint run` |

**Done when:** `kubectl apply` brings the service up, `/readyz` is green, and logs are JSON.
**Stretch:** a GitHub Actions CI job running test + lint + build, and Prometheus metrics with `/metrics`.

---

## How to use AI while learning

| ✅ Ask | ❌ Avoid |
|---|---|
| "Explain how interfaces work in Go" | "Write the storage layer for me" |
| "Why does this error happen?" | Pasting code and asking for a fix |
| "What's the idiomatic way to do X?" | Copying full solutions |
| "Review my code — what's not idiomatic?" | Asking for code before trying yourself |

**The loop for each stage:**
1. **Learn** — ask AI to explain the stage's concepts (Python comparisons help)
2. **Try** — write it yourself, and struggle for at least 20–30 min before asking anything
3. **Hint** — when stuck, ask for a *hint*, not code ("which package should I look at?")
4. **Review** — once it works, ask for an idiomatic-Go review
5. **Refactor** — apply the feedback, then commit
6. **Log** — write 3–5 bullet points in `notes/stage-N.md`: what clicked, what confused you, any gotchas

---

## Progress tracker

| Stage | Topic | Status | Started | Finished | Notes |
|---|---|---|---|---|---|
| 0 | Setup | ⬜ Not started | | | |
| 1 | CLI basics | ⬜ Not started | | | |
| 2 | Save to file | ⬜ Not started | | | |
| 3 | Methods, interfaces, tests | ⬜ Not started | | | |
| 4 | HTTP API | ⬜ Not started | | | |
| 5 | MongoDB | ⬜ Not started | | | |
| 6 | Concurrency | ⬜ Not started | | | |
| 7 | Production-ready | ⬜ Not started | | | |

Legend: ⬜ Not started · 🟨 In progress · ✅ Done
