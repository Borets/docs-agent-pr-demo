# Static Sites

Render can build and serve static sites directly from your Git repository. Static sites are free on all plans and deploy automatically on every push.

## What qualifies as a static site

A static site is any project whose build output is a directory of HTML, CSS, JavaScript, and asset files. Common frameworks include:

- Next.js (static export)
- Gatsby
- Vite / Create React App
- Hugo, Jekyll, Eleventy
- Astro (static mode)
- Plain HTML

If your project needs a server to render responses at request time, deploy it as a **Web Service** instead.

## Creating a static site

1. Push your project to GitHub or GitLab.
2. In the Render dashboard, select **New → Static Site** and connect your repository.
3. Set the **Build Command** and **Publish Directory** (see defaults below).
4. Click **Create Static Site**.

Render builds the site and publishes it to a `*.onrender.com` URL. Every subsequent push to the configured branch triggers a new build and deploy automatically.

## Build command and publish directory

Render needs two pieces of information to build and serve your site:

| Setting | Description |
|---|---|
| **Build Command** | The command that produces your static output. |
| **Publish Directory** | The directory Render serves after the build completes. |

Common framework defaults:

| Framework | Build Command | Publish Directory |
|---|---|---|
| Create React App | `npm run build` | `build` |
| Vite | `npm run build` | `dist` |
| Next.js (static export) | `npm run build` | `out` |
| Gatsby | `gatsby build` | `public` |
| Hugo | `hugo` | `public` |
| Eleventy | `npx @11ty/eleventy` | `_site` |

Override both settings from **Settings → Build & Deploy** at any time.

## Custom domains

Add a custom domain from the **Settings → Custom Domains** tab:

1. Enter your domain name and click **Save**.
2. Add a `CNAME` record pointing to your `*.onrender.com` address with your DNS provider.
3. Render automatically provisions and renews a TLS certificate.

Apex domains (bare `example.com` without `www`) require an `ALIAS` or `ANAME` record. Not all DNS providers support these record types; check with your provider if the option is unavailable.

## Redirects and rewrites

For single-page applications, you need to rewrite all paths to `index.html` so that client-side routing works correctly. Create a `_redirects` file in the root of your **publish directory**:

```
/* /index.html 200
```

You can also define explicit redirects:

```
/old-path /new-path 301
/blog/* /posts/:splat 302
```

Rules are evaluated top to bottom. The first matching rule wins.

## Headers

Set custom HTTP response headers using a `_headers` file in your publish directory:

```
/*
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff
  Content-Security-Policy: default-src 'self'

/assets/*
  Cache-Control: public, max-age=31536000, immutable
```

## Environment variables

Build-time environment variables are available during the build process. Set them under **Settings → Environment**.

Note that environment variables in static sites are **build-time only**. They are baked into the built files. Do not put secrets or credentials in static site environment variables — the values may be visible in the compiled output.

For frameworks that require a prefix (for example `VITE_` for Vite or `REACT_APP_` for Create React App), add the prefix when setting the variable name.

## Pull request previews

Enable **Pull Request Previews** under **Settings → Pull Request Previews**. Render builds and deploys a separate preview site for each open pull request. A link to the preview is posted directly to the PR.

Preview sites are torn down when the PR is closed or merged.

## Build caching

Render caches `node_modules` and other package manager directories between builds. Most dependency installs are served from cache, so builds run only your framework's actual compilation step.

To clear the cache and force a full reinstall, select **Manual Deploy → Clear build cache & deploy** from the dashboard.

## Common issues

**Site loads but client-side routes return 404**
Your publish directory is missing a `_redirects` file with the `/* /index.html 200` rewrite rule. Add the file and redeploy.

**Build fails with `command not found`**
The build command references a CLI that is not installed. Add it as a dev dependency in your `package.json` or install it explicitly in the build command (for example `npm install -g gatsby && gatsby build`).

**Environment variable not available at runtime**
Static site environment variables are injected only at build time. If a variable is missing in the compiled output, confirm it was set before the last build and redeploy to pick up the new value.

**Custom domain shows a certificate error**
Render provisions TLS automatically once the `CNAME` record propagates (typically within a few minutes to an hour). If the error persists after an hour, verify that the DNS record points to the correct `*.onrender.com` address shown in the dashboard.
