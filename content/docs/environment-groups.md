# Environment Groups

Environment Groups let you define a set of environment variables once and share them across multiple services. This avoids duplicating secrets and config values across every service that needs them.

## Creating an Environment Group

1. In the Render dashboard, open **Environment Groups** from the left sidebar.
2. Click **New Environment Group**.
3. Give the group a name (for example, `shared-db-config`) and add your key-value pairs.
4. Click **Save**.

## Linking a group to a service

1. Open the service you want to update.
2. Go to **Environment → Environment Groups**.
3. Select the group from the dropdown and click **Link**.

The variables from the group are merged with any service-level variables at deploy time. If the same key exists in both the group and the service, the **service-level value takes precedence**.

## Updating variables in a group

Changes to an Environment Group do **not** automatically redeploy linked services. To apply updated values, trigger a manual deploy or push a new commit to each affected service.

> **Tip:** For secrets that rotate frequently, consider using service-level variables so you can redeploy individual services without touching the group.

## Unlinking a group

To remove a group from a service, go to **Environment → Environment Groups** on that service and click **Unlink** next to the group name. The service retains its last-resolved values until the next deploy, at which point the group's variables are no longer injected.

## Permissions and visibility

Environment Group values are visible to any team member with access to the owning project. Do not store secrets in groups that are shared with projects or teammates who should not see them.

## Limits

| Item | Limit |
|---|---|
| Groups per team | 100 |
| Variables per group | 100 |
| Services linked to a single group | Unlimited |

Limits apply to paid plans. Free-tier accounts share the same limits but group usage counts toward the account's overall environment variable quota.

## Common issues

**Variables from a linked group are not showing up**
Confirm the group is linked and that a deploy has completed since you linked it. Changes take effect only on the next successful deploy.

**Service-level variable is not overriding the group value**
Check that the key name matches exactly (keys are case-sensitive). A typo in either the group or the service-level variable results in both keys being injected rather than one overriding the other.
