# Node.js on Render

Render supports Node.js natively. You can deploy any Node.js application as a Web Service, Background Worker, or Cron Job — Render detects your project, installs dependencies, and runs your start command.

## Requirements

- A `package.json` file at the root of your repository (or a subdirectory you specify as the root directory).
- Node.js 18 or later is recommended. Render defaults to a recent LTS release; pin a specific version with a `.node-version` or `.nvmrc` file, or with the `NODE_VERSION` environment variable.

## Quick start

1. Push your Node.js repository to GitHub or GitLab.
2. In the Render dashboard, select **New → Web Service** and connect your repository.
3. Confirm the detected build and start commands (see below) or override them.
4. Click **Create Web Service**.

Render builds and deploys your service automatically on every push to the configured branch.

## Build and start commands

Render auto-detects Node.js projects and sets sensible defaults.

| Setting | Default value |
|---|---|
| **Build command** | `npm install` |
| **Start command** | `node index.js` |

If your project uses a different entry point or a framework CLI, override the start command. Common examples:

| Framework / tool | Start command |
|---|---|
| Express (`server.js`) | `node server.js` |
| Fastify | `node app.js` |
| NestJS | `node dist/main.js` |
| Next.js (server) | `npm run start` |

Override either command from **Settings → Build & Deploy** at any time.

## Pinning a Node version

Create a `.node-version` or `.nvmrc` file in the root of your repository:

```
# .node-version
20.14.0
```

Alternatively, set the `NODE_VERSION` environment variable on your service. The file-based approach is preferred because it keeps the version in source control and applies to all environments consistently.

## Listening on PORT

Render injects the `PORT` environment variable for every Web Service. Your app must listen on that port:

```js
const express = require('express');
const app = express();

const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`Server listening on port ${port}`);
});
```

Hard-coding a port number causes health checks to fail and the deploy to be rolled back.

## Environment variables

Set environment variables under the **Environment** tab of your service, or reference them from a Render Environment Group to share values across services. Access them in your code via `process.env`:

```js
const db = new Client({ connectionString: process.env.DATABASE_URL });
const apiKey = process.env.THIRD_PARTY_API_KEY;
```

Never commit secrets to your repository. Set all credentials and API keys as environment variables in the dashboard.

## Health checks

Render sends HTTP GET requests to your configured health check path (default: `/`) to verify your service is running. Respond with a `2xx` status code as soon as your app is ready:

```js
app.get('/healthz', (req, res) => {
  res.sendStatus(200);
});
```

Configure the path under **Settings → Health & Alerts → Health Check Path**. A dedicated endpoint that checks critical dependencies (database connectivity, required config) provides a more reliable signal than the root path.

## TypeScript

For TypeScript projects, compile to JavaScript in the build step and run the compiled output:

| Setting | Example value |
|---|---|
| **Build command** | `npm install && npm run build` |
| **Start command** | `node dist/index.js` |

Make sure `tsc` or your build tool (esbuild, tsup, etc.) is listed as a dev dependency in `package.json` so it is available during the build.

## Static assets

Avoid serving static files from a Node.js Web Service for high-traffic sites. Instead, host assets on a Render Static Site or a CDN and have your Node.js service handle only API requests. If you do serve assets from Node.js, use a caching middleware and set appropriate `Cache-Control` headers.

## Using yarn or pnpm

To use `yarn` or `pnpm` instead of `npm`, update the build command:

```bash
# yarn
yarn install --frozen-lockfile

# pnpm
npm install -g pnpm && pnpm install --frozen-lockfile
```

Commit your lockfile (`yarn.lock` or `pnpm-lock.yaml`) to keep installs reproducible.

## Monorepo setup

If your service is one of several in a monorepo, set the **Root Directory** for the service to the subdirectory containing the relevant `package.json`. Render runs all build and start commands relative to that directory.

If your monorepo uses a workspace tool (Turborepo, Nx, Lerna), keep the build command scoped to the service being deployed:

```bash
# Turborepo: build only the 'api' package
npx turbo build --filter=api
```

## Common issues

**Build fails with `Cannot find module`**
The module is not listed in `package.json` or the lockfile is out of sync. Run `npm install` locally, commit the updated lockfile, and redeploy.

**Service exits immediately after deploy**
Check that the `PORT` environment variable is read and the server binds to it before logging any ready message. A process that exits without binding a port is marked as failed.

**`npm run build` is not recognized**
The `build` script is missing from your `package.json`. Either add it or update the build command in the dashboard to the exact command you want to run.

**TypeScript compilation errors in production but not locally**
Local builds may use a different Node or TypeScript version. Pin `NODE_VERSION` on your Render service to match your local environment, and add TypeScript as a dev dependency so the same version is used in CI and production builds.
