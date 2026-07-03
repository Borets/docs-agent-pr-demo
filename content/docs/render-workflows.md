# Render Workflows

A Render workflow is the full sequence of steps that runs every time you deploy a service — from the moment a change is triggered through the moment traffic shifts to the new version. Understanding each phase lets you ship faster, catch problems earlier, and recover from failures with confidence.

## How a deploy is triggered

Every deploy starts with a trigger. Render supports three:

| Trigger | When it runs |
|---|---|
| **Auto-deploy** | Automatically on every push to your configured branch (enabled by default). |
| **Manual deploy** | On demand from the dashboard (**Manual Deploy → Deploy latest commit**) or via the Render API. |
| **Deploy hook** | On demand via an authenticated webhook URL you generate from the service settings. |

Configure auto-deploy under **Settings → Build & Deploy → Auto-Deploy**. Turning it off lets you gate deploys behind your own CI pipeline or approval step — call the deploy hook or API only after your checks pass.

### Deploy hooks

A deploy hook is a unique HTTPS URL. A `POST` request to that URL starts a deploy of the latest commit on your configured branch. Use hooks to integrate Render deploys into any CI system, script, or external service without giving it full dashboard access.

Generate a hook from **Settings → Build & Deploy → Deploy Hook**. Treat the URL as a secret — anyone with the URL can trigger a deploy.

## The build phase

After a deploy is triggered, Render runs your **Build Command** inside a fresh build container. The container is isolated per deploy; nothing from a previous build's filesystem carries over (module and package caches are stored separately — see [Caching](#caching) below).

Render streams build output to the dashboard log viewer in real time. A non-zero exit code from the build command fails the deploy immediately. The current version continues serving traffic.

### Default build commands by runtime

| Runtime | Default build command |
|---|---|
| Go | `go build -o server .` |
| Node | `npm install; npm run build` |
| Python | `pip install -r requirements.txt` |
| Ruby | `bundle install` |
| Docker | _(image built from Dockerfile)_ |

Override any default from **Settings → Build & Deploy → Build Command**.

### Caching

Render caches language package managers between builds (Go module cache, `node_modules`, pip wheel cache, etc.). The cache is keyed per service and per branch. Commits that add or remove dependencies invalidate only the changed packages — the rest of the cache stays warm.

To force a clean build, select **Manual Deploy → Clear build cache & deploy** from the dashboard.

## Pre-deploy commands

A **Pre-Deploy Command** runs inside a short-lived container *after* the build succeeds but *before* traffic shifts to the new version. Use this step for tasks that must be atomic with the deploy:

- Database migrations
- Cache warming
- Schema validation

Set the command under **Settings → Build & Deploy → Pre-Deploy Command**. If the command exits non-zero, Render cancels the deploy and leaves the previous version in place. No traffic is disrupted.

> **Keep pre-deploy commands idempotent.** If a deploy is retried after a partial failure, the pre-deploy command runs again. Migrations and other state-changing operations should be safe to run more than once.

## Health checks

Once the new instance starts, Render waits for it to pass its health check before routing any traffic to it. For Web Services, Render sends an HTTP GET to your configured **Health Check Path** (default: `/`). The service must return a `2xx` response within the timeout window.

Configure the path under **Settings → Health & Alerts → Health Check Path**. A dedicated endpoint — `/healthz` or `/ping` — that checks critical dependencies (database connectivity, required config) gives you the most reliable signal.

If the health check never succeeds:

- The deploy is marked failed.
- The previous version continues serving traffic.
- The new instance is stopped.

## Deploy lifecycle at a glance

```
Trigger received
  └─ Build command runs
       └─ [Build fails] → deploy cancelled, previous version stays
       └─ Build succeeds
            └─ Pre-deploy command runs (if configured)
                 └─ [Pre-deploy fails] → deploy cancelled, previous version stays
                 └─ Pre-deploy succeeds
                      └─ New instance starts, health check begins
                           └─ [Health check fails] → deploy failed, previous version stays
                           └─ Health check passes
                                └─ Traffic shifts to new version
                                     └─ Previous instance drains and stops
```

## Zero-downtime deploys

Render performs zero-downtime deploys on paid plans. The new version must pass its health check before the old version receives a stop signal. During the overlap window, both instances are running — design your application so both versions can coexist safely:

- Make database schema changes backward-compatible with the previous version of your code.
- Do not remove columns or rename fields in the same deploy that adds code expecting the new schema.
- Use the pre-deploy command to run migrations *before* the new code starts serving traffic.

## Rollbacks

To roll back to a previous deploy, open the service in the dashboard, go to the **Events** tab, find the deploy you want to restore, and select **Rollback to this deploy**. Render runs the same workflow — including the pre-deploy command — for the rolled-back version.

Because rollbacks replay the pre-deploy command, ensure that migrations are reversible or that rolling back to an older version of the code is safe against the current database schema.

## Pull request previews

When you enable **Pull Request Previews** for a service, Render automatically creates a temporary copy of the service for each open pull request. The preview uses the same build command, environment variables (minus any you explicitly exclude), and settings as your main service.

Previews are created when a PR is opened and torn down when it is closed or merged. A link to the preview URL is posted directly to the pull request.

Enable previews under **Settings → Pull Request Previews**. You can choose to inherit all environment variables or configure a separate set for preview environments.

> **Preview environments share resources.** If your service connects to a database, previews will connect to the same database unless you override `DATABASE_URL` for previews. Use a separate database or schema for isolated preview testing.

## Notifications and alerts

Render can notify you when a deploy succeeds, fails, or is rolled back. Configure notification destinations under **Settings → Notifications**:

- **Email** — sent to the service owner or a configured address.
- **Slack** — post deploy events to a channel via a webhook.
- **PagerDuty / generic webhook** — forward events to any endpoint.

Health check failures also trigger alerts. Set alert thresholds under **Settings → Health & Alerts**.

## Environment promotion pattern

For services that progress through staging and production environments, a common pattern is:

1. **Staging service** — auto-deploys on every push to `main`. Deploy hooks are disabled.
2. **Production service** — auto-deploy is off. A deploy hook is called only after staging passes your acceptance criteria.

This gives you a human or automated gate between staging and production while using the same Render workflow infrastructure for both environments.

## Common issues

**Build succeeds but the deploy never goes live**
The health check is likely failing. Check that your app binds to the `PORT` environment variable and responds `2xx` on your configured health check path before the timeout.

**Pre-deploy command runs every time even when migrations are already applied**
This is expected behavior — design your migration tool to be a no-op when the schema is already current (most migration frameworks do this by default).

**Auto-deploy triggers on a branch I didn't configure**
Verify the branch name under **Settings → Build & Deploy → Branch**. A branch rename in your repository does not automatically update the Render setting.

**Old version keeps serving traffic after a new deploy**
On free-tier services, Render does not perform zero-downtime deploys. There is a brief period where neither version is serving traffic while the old instance stops and the new one starts.
