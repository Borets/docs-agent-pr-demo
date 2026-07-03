# PostgreSQL Databases

Render provides managed PostgreSQL databases that deploy in seconds and connect directly to your other Render services. This page covers creating a database, connecting your service to it, and managing it in production.

## Creating a database

1. In the Render dashboard, select **New → PostgreSQL**.
2. Give the database a name and choose a region. Use the same region as the services that will connect to it to avoid cross-region latency.
3. Select a plan. Free-tier databases are available for development and testing; they expire after 90 days.
4. Click **Create Database**.

Render provisions the database and displays its connection details on the **Info** tab.

## Connection strings

Each database exposes two connection strings:

| String | When to use |
|---|---|
| **Internal Database URL** | Services running in the same Render region. No egress charges, lower latency. |
| **External Database URL** | Connections from outside Render — your local machine, a CI runner, or a third-party service. |

Always use the internal URL for service-to-service connections within Render. The internal URL is not reachable from the public internet.

When you link a database to a service using the **Connect** button in the dashboard, Render automatically injects `DATABASE_URL` (set to the internal URL) as an environment variable on the service.

## Connecting from a service

After linking the database, your service reads the connection string from `DATABASE_URL`. No hard-coded credentials are needed:

```js
// Node.js example using the pg package
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
```

```python
# Python example using psycopg2
import os, psycopg2
conn = psycopg2.connect(os.environ["DATABASE_URL"])
```

```go
// Go example using database/sql
import (
    "database/sql"
    "os"
    _ "github.com/lib/pq"
)

db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
```

Validate the connection at startup and fail fast if `DATABASE_URL` is missing or the connection cannot be established. A service that starts without a working database connection is likely to fail later under real traffic.

## SSL

Render PostgreSQL requires SSL for all external connections. The internal URL does not require additional SSL configuration. If you connect from outside Render, append `?sslmode=require` to the connection string or configure your client library to enforce SSL.

## Running migrations

Run migrations before your new code starts serving traffic. The recommended approach is a **Pre-Deploy Command**:

1. Open your service in the dashboard and go to **Settings → Build & Deploy**.
2. Set **Pre-Deploy Command** to your migration runner. Examples:

   ```bash
   # Prisma
   npx prisma migrate deploy

   # golang-migrate
   ./migrate -database "$DATABASE_URL" -path ./migrations up

   # Alembic (Python)
   alembic upgrade head
   ```

3. If the migration fails, Render cancels the deploy and the previous version of your service continues running against the unchanged schema.

Design migrations to be backward-compatible with the previous version of your code whenever possible. This keeps zero-downtime deploys safe.

## Connection pooling

Each service instance opens its own connection pool. On plans with a low connection limit, multiple scaled instances can exhaust the database's `max_connections`. To avoid this:

- Set conservative pool sizes in your application (see the example below for Go).
- Use [PgBouncer](https://www.pgbouncer.org/) or a similar connection pooler as a separate service.
- Upgrade to a Render PostgreSQL plan with a higher connection limit.

```go
db.SetMaxOpenConns(10)
db.SetMaxIdleConns(5)
db.SetConnMaxLifetime(5 * time.Minute)
```

## Backups

Render automatically takes daily backups of paid-plan databases. You can also trigger a manual backup at any time from the **Backups** tab of your database. Backups are retained according to your plan's retention policy.

Free-tier databases do not include automated backups.

## Accessing the database directly

To connect to your database from a local machine or run one-off queries, use the external connection string with `psql` or any PostgreSQL client:

```bash
psql "$EXTERNAL_DATABASE_URL"
```

You can also open an interactive shell using the **Shell** tab in the database's dashboard page.

## Upgrading PostgreSQL

Render supports major PostgreSQL version upgrades. Before upgrading:

1. Take a manual backup from the **Backups** tab.
2. Test the upgrade on a non-production database first.
3. Review the PostgreSQL release notes for breaking changes that affect your application.

Upgrades require a brief period of downtime. Plan the upgrade during a low-traffic window.

## Common issues

**`too many connections` errors**
Your application is opening more connections than the database plan allows. Reduce pool size per instance, add a connection pooler, or upgrade your database plan.

**`SSL connection is required` when connecting locally**
Add `?sslmode=require` to your external connection string, or set `PGSSLMODE=require` in your shell.

**Migrations fail during pre-deploy**
Check that `DATABASE_URL` is available in the pre-deploy environment. The variable is injected automatically when you link the database to the service through the dashboard.

**Free-tier database expired**
Free-tier PostgreSQL databases expire after 90 days. Upgrade to a paid plan or create a new database and restore your data from a backup.
