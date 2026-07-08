# Web Services

Web services let you deploy any HTTP server on Render. Render builds and runs your
service from source, a Docker image, or a pre-built artifact.

## Instance types

Each web service runs on an instance type that determines CPU, RAM, and pricing.
You select the instance type when you create the service and can change it later
from the **Settings** tab.

### Free instances

Free instances are available for individual web services at no cost. Keep the
following behaviors in mind when using them.

**Spin-down on inactivity**
A free instance automatically spins down after 15 minutes without an inbound
request. The instance stops consuming resources while it is spun down.

**Idle snapshot**
After a free instance has been idle for one minute, Render takes an in-memory
snapshot of the running process. If the service receives a request before the
15-minute spin-down threshold, Render can restore from this snapshot instead of
performing a full cold start, reducing resume latency. The snapshot is discarded
once the instance fully spins down.

**Cold starts**
When a new request arrives for a fully spun-down service, Render starts the instance
from scratch before serving the response. This cold start typically adds 30–60 seconds
of latency to the first request. Subsequent requests are served normally until the
service spins down again.

**Resource limits**
Free instances include 512 MB RAM and a shared CPU. Workloads that require
sustained CPU or memory beyond these limits need a paid instance type.

**No SLA**
Free instances do not carry an uptime SLA. For production traffic that requires
consistent availability, use a Starter or higher instance type.

**Build minutes**
Free instances share the build-minute allowance of your account's free plan.
Check the **Billing** page in the Render dashboard to view current usage.

## Scaling

Free instances do not support autoscaling. To enable multiple instance replicas,
upgrade to a paid instance type and turn on autoscaling in the **Scaling** tab.

## Custom domains

You can add a custom domain to any web service regardless of instance type. See the
[custom domains documentation](https://render.com/docs/custom-domains) for setup
instructions.

## Health checks

Render uses the health-check path you configure in **Settings → Health & Alerts**
to decide whether a deploy succeeded. If the health check fails, Render rolls back
to the previous deploy.

Free instances respond to health checks, but because the instance may be spun down,
allow extra time for the initial health check during a new deploy.
