# Getting Started

This page is intentionally small so documentation-agent PRs are easy to review.

## Local setup

1. Install dependencies.
2. Start the development server.
3. Open the local documentation site.

## Autoscaling

Render can automatically scale your service up or down based on CPU or memory
utilization. You configure autoscaling in **Settings → Scaling** for any web
service or background worker.

### Scaling policies

| Setting | What it controls |
|---|---|
| **Min instances** | The floor — Render never scales below this count. Set it to `1` or higher to avoid cold starts. |
| **Max instances** | The ceiling — Render will not provision more instances than this value, regardless of load. |
| **Scale-up threshold** | CPU or memory utilization percentage at which Render adds an instance. A common starting point is `75 %`. |
| **Scale-down threshold** | Utilization percentage at which Render removes an instance. Keep it comfortably below the scale-up threshold (e.g. `25 %`) to prevent oscillation. |
| **Cooldown period** | Minimum seconds between consecutive scaling events. The default is `300 s`; increase it for services with slow startup times. |

### Tips

- **Health checks gate scale-up.** A new instance starts receiving traffic only
  after it passes its health-check endpoint. Configure the endpoint in
  **Settings → Health & Alerts** before enabling autoscaling.
- **Session state must be external.** Multiple instances do not share memory.
  Store sessions in a Render Redis instance or another external store.
- **Database connections multiply.** Each instance opens its own connection
  pool. Stay within your database plan's connection limit; consider a connection
  pooler such as PgBouncer if you scale to many instances.
- **Scaling is plan-gated.** Autoscaling is available on paid plans. Check
  **Settings → Scaling** to confirm it is enabled for your service type.

## Validation

Before opening a PR, run the project checks and include the results in the PR
description.

