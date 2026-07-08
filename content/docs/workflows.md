# Workflows

Workflows let you define and automate multi-step processes across your Render
services. Use them to coordinate deploys, run migrations, trigger health checks,
or chain any sequence of operations that would otherwise require manual steps.

## Overview

A workflow is an ordered list of steps. Each step runs a command or calls a
Render API action. Steps run sequentially by default; a step that fails stops
the workflow and surfaces an error.

You can trigger workflows:

- **Manually** — from the Render dashboard or via the CLI.
- **On deploy** — automatically after a successful service deploy.
- **On a schedule** — at a fixed interval using cron syntax.

## Creating a workflow

1. Navigate to your service in the Render dashboard.
2. Select **Workflows** from the left sidebar.
3. Click **New Workflow**.
4. Give the workflow a name and, optionally, a description.
5. Add one or more steps. For each step, choose a step type and provide the
   required configuration.
6. Click **Save**.

The workflow is inactive until you enable it or trigger it manually.

## Step types

| Step type | What it does |
|-----------|--------------|
| **Shell command** | Runs an arbitrary shell command inside your service's environment. |
| **Deploy** | Triggers a deploy for a specified Render service. |
| **Health check** | Polls a URL until it returns a success status code or times out. |
| **Notification** | Sends a message to a configured notification channel. |

## Triggering a workflow manually

From the **Workflows** page, click the **Run** button next to the workflow you
want to execute. Render queues the workflow immediately. You can monitor
progress in the run log that appears below the workflow definition.

To trigger a workflow from the CLI:

```
render workflows run <workflow-id>
```

## Scheduling a workflow

When creating or editing a workflow, set **Trigger** to **Schedule** and enter a
cron expression. Render evaluates the expression in UTC.

Example — run every day at 03:00 UTC:

```
0 3 * * *
```

Scheduled workflows are paused automatically if they fail three consecutive times.
You can re-enable them from the **Workflows** page after you resolve the
underlying issue.

## Viewing run history

Each workflow run is recorded with a timestamp, trigger source, step-by-step
status, and log output. To view run history:

1. Open the workflow from the **Workflows** page.
2. Click the **History** tab.

Logs are retained for 30 days.

## Limits

| Resource | Limit |
|----------|-------|
| Workflows per service | 10 |
| Steps per workflow | 20 |
| Maximum run duration | 30 minutes |

Limits may vary by plan. Check your plan details in the Render dashboard for the
values that apply to your account.
