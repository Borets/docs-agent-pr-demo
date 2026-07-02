# Go on Render

Render supports Go natively. You can deploy Go web services, background workers, and cron jobs without managing a runtime or configuring a build environment yourself. Render detects Go projects automatically and builds them using the Go toolchain.

## Supported Go versions

Render supports Go 1.20 and later. To pin a specific version, add a `go` directive to your `go.mod` file:

```
go 1.22
```

Render reads `go.mod` at build time and uses the declared version. If no version is declared, Render uses the latest stable release.

## Auto-detection

Render detects a Go project when it finds a `go.mod` file at the root of your repository. No additional configuration is required to trigger Go builds.

## Build and start commands

Render sets default build and start commands for detected Go projects:

| Setting | Default value |
|---|---|
| **Build command** | `go build -o app ./...` |
| **Start command** | `./app` |

You can override either command in the Render Dashboard or in your `render.yaml` file.

### Custom build command example

If your `main` package is not at the module root, point the build command at the correct path:

```
go build -o app ./cmd/server
```

## Deploying a Go web service

### From the Dashboard

1. Go to the [Render Dashboard](https://dashboard.render.com) and select **New → Web Service**.
2. Connect your repository.
3. Render auto-detects Go and populates the build and start commands.
4. Set your **Environment** to **Go** if it is not already selected.
5. Select your instance type and click **Create Web Service**.

### Using render.yaml

Define your service in `render.yaml` at the root of your repository:

```yaml
services:
  - type: web
    name: my-go-service
    runtime: go
    buildCommand: go build -o app ./cmd/server
    startCommand: ./app
    envVars:
      - key: PORT
        value: 8080
```

Commit `render.yaml` and Render will apply it on the next deploy.

## Environment variables

Render injects `PORT` automatically. Your server must listen on the port provided by this variable. A minimal example:

```go
package main

import (
    "fmt"
    "net/http"
    "os"
)

func main() {
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello from Render!")
    })

    http.ListenAndServe(":"+port, nil)
}
```

Set additional environment variables in the **Environment** tab of your service in the Dashboard, or declare them under `envVars` in `render.yaml`.

## Health checks

Render performs HTTP health checks against your web service after each deploy. By default, Render checks the `/` path. To use a dedicated health endpoint, set the **Health Check Path** field in the Dashboard or add it to `render.yaml`:

```yaml
services:
  - type: web
    name: my-go-service
    runtime: go
    healthCheckPath: /healthz
```

Your health endpoint should return a `200` status when the service is ready to handle traffic.

## Static files and asset embedding

Go 1.16+ supports embedding static assets directly into the binary with `//go:embed`. This is the recommended approach for serving static files on Render because it avoids relying on file paths relative to the working directory at runtime:

```go
import "embed"

//go:embed static/*
var staticFiles embed.FS
```

## Background workers and cron jobs

Deploy non-HTTP Go programs as a **Background Worker** or **Cron Job** service type. Use the same build command and replace the start command with the entrypoint for your worker binary.

```yaml
services:
  - type: worker
    name: my-go-worker
    runtime: go
    buildCommand: go build -o worker ./cmd/worker
    startCommand: ./worker
```

## Databases and external services

Connect your Go service to a [Render PostgreSQL database](https://render.com/docs/databases) by adding the `DATABASE_URL` environment variable. Use the internal URL for services deployed in the same region to avoid egress fees.

Popular Go database libraries work without additional configuration:

- [`database/sql`](https://pkg.go.dev/database/sql) with `lib/pq` or `pgx`
- [`gorm`](https://gorm.io) with the PostgreSQL driver
- [`sqlx`](https://github.com/jmoiron/sqlx)

## Monorepos

If your Go service is not at the repository root, set the **Root Directory** field in the Dashboard to the subdirectory containing `go.mod`. Build and start commands run relative to that directory.

## Private modules

To pull private Go modules during the build, set the `GOPRIVATE` environment variable and supply credentials via a `NETRC` secret file or a `GIT_CONFIG` environment variable. See [Go module authentication](https://go.dev/ref/mod#authenticating) for details.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Build fails with `no Go files` | Build command targets a directory without `.go` files | Update the build command to point at the correct package path |
| Service crashes immediately | Binary can't bind to the expected port | Ensure your server reads `PORT` from the environment |
| `go: updates to go.sum needed` | `go.sum` is out of date or not committed | Run `go mod tidy` locally and commit the updated `go.sum` |
| Health check fails | Service not ready before the deadline | Increase the health check grace period in the Dashboard or speed up startup |

## Further reading

- [Render web services](https://render.com/docs/web-services)
- [Infrastructure as code with render.yaml](https://render.com/docs/infrastructure-as-code)
- [Environment variables on Render](https://render.com/docs/environment-variables)
- [Render managed databases](https://render.com/docs/databases)
