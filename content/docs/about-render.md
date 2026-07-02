# About Render

Render is a unified cloud platform for deploying web apps, APIs, background workers,
cron jobs, static sites, and databases — all from a single dashboard or via
infrastructure-as-code.

## What you can deploy

| Service type | Description |
|---|---|
| **Web Service** | Long-running HTTP servers (Node, Python, Go, Ruby, Rust, and more). Render handles TLS, custom domains, and zero-downtime deploys automatically. |
| **Static Site** | Pre-built or framework-generated static files served from Render's global CDN. |
| **Background Worker** | Processes that run continuously without an inbound HTTP port. |
| **Cron Job** | Scheduled commands that run on a configurable cron schedule. |
| **Private Service** | Internal HTTP services reachable only by other services in your account. |
| **PostgreSQL** | Managed PostgreSQL databases with automatic backups and point-in-time recovery (paid plans). |
| **Redis** | Managed Redis instances for caching and queues. |

## How deploys work

1. **Connect a Git repository.** Render watches your chosen branch. Every push
   triggers a new build and deploy automatically.
2. **Build phase.** Render runs your build command (e.g. `npm run build`) in an
   isolated environment. The build fails fast and never replaces your live service
   until it succeeds.
3. **Deploy phase.** Render swaps in the new instance. For web services, the old
   instance stays live until the new one passes its health check.
4. **Rollback.** You can redeploy any previous successful build from the dashboard
   with one click.

You can also deploy from a `render.yaml` file checked into your repository
(Infrastructure as Code), which lets you define all your services and environment
variables in version control.

## Regions

Render services can be deployed to the following regions:

- Oregon, USA (`oregon`)
- Frankfurt, Germany (`frankfurt`)
- Singapore (`singapore`)
- Ohio, USA (`ohio`)

Region availability depends on your plan. Check the dashboard when creating a
service to see which regions are available to your account.

## Pricing

Render offers a free tier for static sites and individual web services with
[usage limits](https://render.com/pricing). Paid plans unlock higher RAM and CPU
options, more database storage, and additional team features. Pricing is
per-service — you pay only for what you deploy.

## Next steps

- [Getting Started](./getting-started.md) — set up your local environment and open
  your first pull request against the docs.
- [Release Notes](./release-notes.md) — see what has changed recently.
