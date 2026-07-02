# Render Overview

Render is a unified cloud platform for deploying and scaling web applications,
APIs, background workers, cron jobs, static sites, and databases — all from a
single dashboard or via infrastructure-as-code.

## What you can deploy

Render supports a wide range of workload types:

- **Web Services** — Long-running HTTP servers in any language or framework.
- **Static Sites** — Automatically built and served from a global CDN.
- **Background Workers** — Processes that run continuously without an inbound
  HTTP listener.
- **Cron Jobs** — Scheduled tasks that run on a defined interval.
- **Private Services** — Internal services reachable only within your Render
  private network.
- **Databases** — Managed PostgreSQL instances with automated backups.

## How deploys work

Render connects to your Git repository (GitHub or GitLab). Every push to your
configured branch triggers a build and deploy. You can also trigger deploys
manually from the dashboard or via the Render API.

Builds run in isolated environments. Render detects your runtime automatically
for common stacks, or you can supply a `Dockerfile` for full control.

## Scaling and availability

You can scale services horizontally by adjusting the instance count, or
vertically by selecting a larger instance type. Render routes traffic across
instances automatically.

Plan availability and scaling limits vary by plan tier. Refer to the
[Render pricing page](https://render.com/pricing) for current details.

## Infrastructure as Code

`render.yaml` lets you define all of your services, databases, and environment
variables in a single file checked in to your repository. Render applies the
configuration on every deploy, keeping your infrastructure in sync with your
code.

## Further reading

- [Getting Started](./getting-started.md)
- [Release Notes](./release-notes.md)
