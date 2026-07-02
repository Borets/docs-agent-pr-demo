# Go on Render: Production Guide

This guide walks through the full lifecycle of running a production Go web service on Render — from wiring up a database to handling graceful shutdowns and keeping deploys fast. It assumes you have already completed the [basic Go setup](render-go.md).

## Structuring your service for Render

Render runs your service as a stateless process. A few structural choices upfront make everything else easier:

- **Read all configuration from environment variables.** Go's `os.Getenv` works as expected. Use a startup helper to validate required variables and fail fast if any are missing; this surfaces misconfiguration at deploy time rather than at runtime.
- **Listen on `PORT`.** Render sets this variable for every Web Service. Hard-coding a port causes health checks to fail and the deploy to be rolled back.
- **Exit cleanly on `SIGTERM`.** Render sends `SIGTERM` before stopping a container. A service that does not handle it may drop in-flight requests.

## Graceful shutdown

Wrap your `http.Server` with a shutdown handler so Render can drain requests before stopping the process:

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })
    // register your other routes here

    srv := &http.Server{
        Addr:    ":" + port,
        Handler: mux,
    }

    // Start the server in a goroutine.
    go func() {
        log.Printf("listening on :%s", port)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // Block until SIGINT or SIGTERM is received.
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    // Give in-flight requests up to 30 seconds to complete.
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("shutdown error: %v", err)
    }
    log.Println("server stopped")
}
```

The 30-second window matches Render's default drain timeout. Adjust it to suit the longest requests your service handles.

## Structured logging

Plain `log.Printf` output is captured by Render and available in the dashboard log viewer. For structured logs that are easier to filter, switch to a JSON logger such as `log/slog` (Go 1.21+):

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("request completed",
    "method", r.Method,
    "path", r.URL.Path,
    "status", statusCode,
    "duration_ms", duration.Milliseconds(),
)
```

Render streams all stdout and stderr output. Do not write logs to a file — they will not appear in the dashboard and will be lost when the container restarts.

## Connecting to a Render PostgreSQL database

When you attach a Render PostgreSQL database to a service, Render sets the `DATABASE_URL` environment variable automatically. Read it at startup:

```go
import (
    "database/sql"
    "os"

    _ "github.com/lib/pq"
)

func openDB() (*sql.DB, error) {
    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" {
        return nil, fmt.Errorf("DATABASE_URL is not set")
    }
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }
    // Set conservative pool limits; Render's free-tier Postgres caps connections.
    db.SetMaxOpenConns(10)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    return db, nil
}
```

Call `db.PingContext` during startup to verify connectivity before accepting traffic. If the ping fails, exit with a non-zero code so Render marks the deploy as failed instead of routing requests to an unhealthy instance.

> **Internal vs. external connection strings.** Render provides both an external URL (accessible from anywhere) and an internal URL (only routable between services in the same region). Use the internal URL for service-to-service connections to avoid egress charges and reduce latency. The internal URL is available as `DATABASE_URL` when the database and service share the same environment.

## Running database migrations

Run migrations as part of the build step or as a separate [Pre-Deploy job](https://render.com/docs/deploys#pre-deploy-commands). Using a Pre-Deploy job keeps migrations atomic with the deploy:

1. In the Render dashboard, open your service and go to **Settings → Deploy**.
2. Set **Pre-Deploy Command** to your migration runner, for example:
   ```
   ./migrate -database "$DATABASE_URL" -path ./migrations up
   ```
3. If the migration command exits non-zero, Render cancels the deploy and the previous version continues serving traffic.

## Caching dependencies across deploys

Render caches the Go module download cache between builds. To take advantage of it, make sure your `go.sum` file is committed and up to date. Builds that require downloading uncached modules are slower; a fully resolved `go.sum` keeps the cache warm.

If you manage dependencies with a vendor directory, set your build command to:

```bash
go build -mod=vendor -o server .
```

Vendored builds are fully reproducible and do not require network access at build time.

## Health check endpoint

Configure a dedicated health check path under **Settings → Health & Alerts → Health Check Path**. A lightweight endpoint that checks critical dependencies gives Render a reliable signal:

```go
mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    if err := db.PingContext(r.Context()); err != nil {
        http.Error(w, "database unreachable", http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

Keep health checks fast — they run on a short interval and a slow response counts against the timeout. Avoid expensive queries inside the handler.

## Zero-downtime deploys

Render performs rolling deploys by default for paid plans: the new version starts and passes its health check before the old version is stopped. To take full advantage of this:

- Ensure health checks respond correctly as soon as the process binds to `PORT`, even before background workers or caches are warm.
- Make database schema changes backward-compatible with the previous version of your code. Deploy the migration first, then the code change.
- Use the graceful shutdown pattern above so the old instance drains requests cleanly while the new one is starting.

## Scaling horizontally

Add instances to your service from **Settings → Scaling**. Go's standard library HTTP server is goroutine-per-connection and scales well across cores. A few things to verify before adding instances:

- **Session state must live outside the process.** Use a Render Redis instance or an external store. In-memory session state is not shared across instances.
- **Scheduled work must be idempotent.** If you use a `time.Ticker` inside your service for background tasks, multiple instances will run the ticker concurrently. Move time-sensitive scheduled work to a dedicated [Cron Job](https://render.com/docs/cronjobs) service.
- **Database connection limits.** Each additional instance opens its own connection pool. Stay within the connection limits of your Postgres plan or add a connection pooler.

## Production deploy checklist

Before promoting a Go service to production on Render, confirm the following:

- [ ] App reads `PORT` from the environment and listens on it.
- [ ] `SIGTERM` is caught and the server shuts down gracefully.
- [ ] All required environment variables are validated at startup.
- [ ] `DATABASE_URL` uses the internal connection string for same-region services.
- [ ] Database migrations run as a Pre-Deploy command or a controlled step before code deploys.
- [ ] A `/healthz` (or equivalent) endpoint is registered and configured in the dashboard.
- [ ] Logs go to stdout/stderr in a structured format.
- [ ] `go.sum` is committed and up to date.
- [ ] No secrets are hard-coded; all credentials are set as environment variables or secret files.
