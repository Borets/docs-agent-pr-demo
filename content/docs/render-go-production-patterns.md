# Go on Render: Production Patterns

This page covers production-ready patterns for Go services running on Render. It builds on the basics in [Render Go](./render-go.md) and focuses on areas that matter most once you move beyond a toy deployment: graceful shutdown, database connections, structured logging, Docker-based builds, and observability.

## Graceful shutdown

Render sends `SIGTERM` to your process before stopping it. If your service does not handle `SIGTERM`, in-flight requests are cut off immediately. A clean shutdown drains active connections before the process exits.

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

    srv := &http.Server{
        Addr:    ":" + port,
        Handler: routes(),
    }

    // Start serving in a goroutine so the main goroutine can wait for a signal.
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %v", err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit
    log.Println("shutting down...")

    ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("forced shutdown: %v", err)
    }
    log.Println("exited cleanly")
}
```

Render waits up to 30 seconds after sending `SIGTERM` before force-killing the process. Keep your shutdown window well under that limit.

## Database connections

### Connection pool sizing

Go's `database/sql` manages a connection pool automatically. The defaults (unlimited open connections, 0 idle connections) are rarely right for a production service. Set these limits early to avoid exhausting your database's connection limit.

```go
db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
if err != nil {
    log.Fatal(err)
}

db.SetMaxOpenConns(25)
db.SetMaxIdleConns(10)
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(2 * time.Minute)
```

The right numbers depend on your database plan and service count. Start conservative and tune based on observed connection counts.

### Using `DATABASE_URL`

If you attach a Render PostgreSQL database to your service, Render injects `DATABASE_URL` automatically. Your app should read this variable rather than constructing a connection string from individual parts.

```go
dsn := os.Getenv("DATABASE_URL")
if dsn == "" {
    log.Fatal("DATABASE_URL is not set")
}
```

### Migrations at deploy time

Run schema migrations as part of your deploy, not in the main process. A Render [pre-deploy command](https://render.com/docs/deploys#pre-deploy-command) runs before traffic shifts to the new instance, which prevents the new code from serving requests before the schema is ready.

**Pre-deploy command example (using [golang-migrate](https://github.com/golang-migrate/migrate)):**
```bash
go run ./cmd/migrate up
```

## Structured logging

Plain `log.Println` output is captured by Render and available in the dashboard log viewer. Structured JSON logs are easier to filter and query, especially at scale.

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
slog.SetDefault(logger)

slog.Info("request completed",
    "method", r.Method,
    "path",   r.URL.Path,
    "status", statusCode,
    "dur_ms", duration.Milliseconds(),
)
```

`log/slog` is available from Go 1.21. For earlier versions, [zerolog](https://github.com/rs/zerolog) or [zap](https://github.com/uber-go/zap) provide the same structured output.

Avoid logging sensitive data (tokens, passwords, full request bodies) — log output is visible to anyone with dashboard access for your service.

## Docker-based builds

Render supports a `Dockerfile` at the root of your repository. A multi-stage build produces a small final image that does not include the Go toolchain.

```dockerfile
# syntax=docker/dockerfile:1

# ── Build stage ──────────────────────────────────────────────────────────────
FROM golang:1.22-alpine AS builder
WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/api

# ── Runtime stage ─────────────────────────────────────────────────────────────
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

**Why this matters on Render:**
- The final image contains only your binary, which reduces attack surface and image pull time.
- `CGO_ENABLED=0` produces a statically linked binary that runs in a minimal base image without a C library.
- Render reads `EXPOSE` for documentation only; your app must still read `PORT` from the environment.

If you use a Dockerfile, Render uses it instead of the native Go buildpack. You no longer need to set build or start commands manually.

## Health check endpoint

The default health check path is `/`. For production services, expose a dedicated endpoint that verifies your app's critical dependencies before returning 200.

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    // Check your database connection.
    if err := db.PingContext(r.Context()); err != nil {
        http.Error(w, "db unavailable", http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ok"))
})
```

Set the health check path to `/healthz` under **Settings → Health & Alerts → Health Check Path**. A service whose health check returns 503 will not receive traffic until it recovers.

## Zero-downtime deploys

Render keeps the previous instance running until the new instance passes its health check. To take full advantage of this:

1. Handle `SIGTERM` gracefully (see above).
2. Ensure your health check endpoint returns 200 only when the service is fully ready to accept traffic.
3. Avoid startup tasks — such as cache warming or long migrations — that block the health check from succeeding within the timeout.

If your service needs significant warm-up time, increase the **Health Check Timeout** under **Settings → Health & Alerts**.

## Environment-specific configuration

Use environment variables for all configuration that differs between local development and production. Avoid `init()` functions or package-level `var` blocks that read environment at import time — they run before your application has a chance to validate or provide defaults for missing variables.

```go
type Config struct {
    Port        string
    DatabaseURL string
    LogLevel    string
}

func configFromEnv() Config {
    return Config{
        Port:        envOrDefault("PORT", "8080"),
        DatabaseURL: requireEnv("DATABASE_URL"),
        LogLevel:    envOrDefault("LOG_LEVEL", "info"),
    }
}

func requireEnv(key string) string {
    v := os.Getenv(key)
    if v == "" {
        log.Fatalf("required environment variable %s is not set", key)
    }
    return v
}

func envOrDefault(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}
```

Use **Environment Groups** in the Render dashboard to share variables across services without duplicating them.

## Observability

Render exposes logs in the dashboard and via the [Log Stream](https://render.com/docs/log-streams) feature. For deeper observability:

- **Metrics:** Expose a `/metrics` endpoint compatible with Prometheus and scrape it with an external monitoring service, or push metrics to a hosted provider using an OpenTelemetry exporter.
- **Tracing:** Instrument with the OpenTelemetry Go SDK and export traces to your preferred backend. Set the exporter endpoint and service name via environment variables so no code changes are needed between environments.
- **Error tracking:** Integrate a Sentry or Rollbar SDK and set `SENTRY_DSN` (or equivalent) as an environment variable on the service.

## Common production issues

**Memory grows unbounded over time**
Enable Go's built-in memory profiling with `net/http/pprof` in non-production environments, or ship a pprof endpoint gated behind a secret header. Goroutine leaks and large allocations in hot paths are the most common causes.

**Deploys time out on health check**
The new instance is not accepting connections within the health check timeout. Check that `PORT` is being read from the environment, that the server starts before the first health check fires (within ~30 s by default), and that any startup blocking work (migrations, pre-warming) finishes promptly.

**`context canceled` errors in logs**
Clients disconnecting mid-request propagate cancellation through `r.Context()`. Handle `context.Canceled` explicitly in database queries and downstream calls so you do not log spurious errors for normal client disconnects.
