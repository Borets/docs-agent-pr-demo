# Render Overview

Render is a unified cloud platform for deploying and running apps, APIs, background
workers, databases, and static sites. You manage everything from a single dashboard
or the Render CLI — no infrastructure expertise required.

## What you can deploy

| Service type | Description |
|---|---|
| **Web Services** | Long-running HTTP servers. Render automatically provisions TLS, a `onrender.com` subdomain, and zero-downtime deploys. |
| **Static Sites** | Frontend builds served from Render's global CDN. Free on all plans. |
| **Background Workers** | Processes that run continuously but don't listen on a public port. |
| **Cron Jobs** | Scheduled commands that run on a POSIX cron schedule. |
| **Private Services** | Internal HTTP services reachable only by other services in the same Render account. |
| **PostgreSQL** | Fully managed Postgres databases with automated backups. |
| **Redis** | Managed Redis instances for caching and queuing. |

## How deploys work

Every push to your connected Git branch triggers a new build. Render checks out
your code, runs your build command, then swaps traffic to the new instance once
it passes health checks. If the health check fails, Render keeps the previous
deploy live and marks the new one as failed.

You can also trigger deploys manually from the dashboard or via the deploy hook URL
shown in your service settings.

## Networking

- Each web service gets a public HTTPS endpoint on `onrender.com`. You can attach a
  custom domain at any time.
- Services in the same Render account communicate over a private network using their
  service name as the hostname (e.g. `my-api:10000`). Traffic on the private network
  does not leave Render's infrastructure.
- Outbound traffic uses a shared IP range by default. Static outbound IPs are
  available on paid plans.

## Scaling

You can scale a service vertically by changing its instance type, or horizontally by
increasing the number of instances. Render distributes incoming requests across all
running instances. Auto-scaling rules let you scale based on CPU or memory usage
without manual intervention.

## Environment variables and secrets

Set environment variables in the **Environment** tab of any service. Values are
encrypted at rest and injected into your process at runtime. For values shared across
multiple services, use an **Environment Group** and link it to each service that needs
it.

## Plans and regions

Render offers free-tier instances for experimentation. Production workloads should
use paid plans for guaranteed resources and SLA coverage. Services can be deployed to
any supported region; the available regions are listed in the dashboard when you
create or edit a service.

## Further reading

- [Getting Started](./getting-started.md) — set up your local environment and run
  the first checks.
- [Release Notes](./release-notes.md) — recent platform changes.
