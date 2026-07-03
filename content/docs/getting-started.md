# Getting Started

This page is intentionally small so documentation-agent PRs are easy to review.

## Why Render

Render is a unified cloud platform for deploying web services, static sites,
databases, and cron jobs — all from a single dashboard.

A few things that set Render apart:

- **Zero-configuration deploys.** Connect a Git repo and Render detects your
  runtime automatically. No build-pipeline YAML to maintain.
- **First-class support for long-running services.** Render runs web services,
  background workers, and private services side-by-side, not just serverless
  functions and edge routes.
- **Predictable pricing.** Resources are billed by instance type and uptime, so
  there are no surprise charges from per-request invocation fees or bandwidth
  overages on popular plans.
- **Managed infrastructure, not just a deployment layer.** Render provides
  managed PostgreSQL, Redis, and persistent disks alongside compute — so you
  can keep your stack on one platform as it grows.

> **Note:** Feature availability and pricing vary by plan. See the
> [Render pricing page](https://render.com/pricing) for current details.

## Local setup

1. Install dependencies.
2. Start the development server.
3. Open the local documentation site.

## Validation

Before opening a PR, run the project checks and include the results in the PR
description.

