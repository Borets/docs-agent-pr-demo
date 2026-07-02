# Render Go

Render supports Go natively. You can deploy any Go application as a Web Service, Background Worker, or Cron Job with no additional tooling required — Render detects your Go module, builds your binary, and runs it.

## Requirements

- A `go.mod` file at the root of your repository (or a subdirectory you specify as the root directory).
- Go 1.20 or later is recommended. Render defaults to a recent stable release; pin a specific version with the `GOVERSION` environment variable if your app requires it.

## Quick start

1. Push your Go repository to GitHub or GitLab.
2. In the Render dashboard, select **New → Web Service** and connect your repository.
3. Confirm the detected build and start commands (see below) or override them.
4. Click **Create Web Service**.

Render builds and deploys your service automatically on every push to the configured branch.

## Build and start commands

Render auto-detects Go projects and sets sensible defaults.

| Setting | Default value |
|---|---|
| **Build command** | `go build -o server .` |
| **Start command** | `./server` |

If your `main` package lives in a subdirectory, adjust the build command accordingly:

```bash
# Build command for a project whose main package is in cmd/api
go build -o server ./cmd/api
```

Override either command from the **Settings → Build & Deploy** section of your service at any time.

## Pinning a Go version

Set the `GOVERSION` environment variable on your service to request a specific release:

```
GOVERSION=1.22.4
```

Render installs the requested version at build time. The variable must be set before the next deploy to take effect.

## Environment variables

Go apps running on Render read environment variables the same way they do locally — via `os.Getenv`. Set variables in the **Environment** tab of your service or reference them from a Render Environment Group to share values across services.

Your app should read the `PORT` environment variable to know which port to listen on:

```go
port := os.Getenv("PORT")
if port == "" {
    port = "8080"
}
http.ListenAndServe(":"+port, nil)
```

Render injects `PORT` automatically for Web Services. Hard-coding a port number will cause health checks to fail.

## Health checks

Render sends HTTP GET requests to `/` by default to verify that your service is up. Configure a different path under **Settings → Health & Alerts → Health Check Path** if your app exposes a dedicated health endpoint (for example `/healthz` or `/ping`).

The check expects a 2xx response within the default timeout. Services that do not bind to `PORT` in time are marked unhealthy and the deploy is rolled back.

## Static files and assets

If your app serves static assets, embed them with `//go:embed` or include them relative to the working directory. The working directory at runtime is the root of your repository. Paths that rely on an absolute build-machine path will not resolve correctly.

## Private modules

To fetch modules from a private repository, add a `GONOSUMCHECK` or `GONOSUMDB` variable and supply credentials via a `NETRC` environment variable or a `~/.netrc` file injected through a secret file. Render does not cache private module downloads outside the build container, so each deploy fetches fresh copies.

## Monorepo setup

If your service is one of several in a monorepo, set the **Root Directory** for the service to the subdirectory containing `go.mod`. Render then runs all build and start commands relative to that directory.

## Cron jobs and background workers

Go binaries work equally well as Background Workers or Cron Jobs on Render. Select the appropriate service type when creating the service. Cron Jobs do not need to bind to `PORT`; Background Workers should only bind if they also expose an internal or public HTTP endpoint.

## Common issues

**Build fails with `no Go files in .`**
The build command is running in the wrong directory. Set **Root Directory** to the folder containing your `go.mod`, or update the build command to target the correct package path (for example `go build -o server ./cmd/myapp`).

**Service restarts immediately after deploy**
Check that your app reads `PORT` from the environment and listens on that port. A process that exits immediately (for example because the port is already in use or the flag is wrong) causes Render to restart it repeatedly.

**Module download errors during build**
Confirm that all dependencies are listed in `go.sum` and committed. If you use a private module proxy, set `GOPROXY` and any required credentials as environment variables on the service.
