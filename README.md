# Blinkhost

Turns AI agent output into live shareable URLs with one API call. No accounts, no login, no deployment pipeline.

An agent generates a frontend artifact, publishes it to Blinkhost, and gets back a live URL immediately.

## How It Works

1. Agent generates HTML/CSS/JS files
2. Agent calls the Blinkhost API with a file manifest
3. Agent uploads files to presigned URLs
4. Agent finalizes the publish -- site is live at `<slug>.blinkhost.co`

## Architecture

A single Cloudflare Worker handles everything: API, landing page, and public site delivery.

| Component | Role |
|-----------|------|
| Cloudflare Worker | API + static pages + site delivery |
| Cloudflare D1 | SQLite database for site metadata |
| Cloudflare R2 | Content-addressed blob storage |
| Wildcard DNS | `*.blinkhost.co` routes to the Worker |

## Quick Start (Local Development)

### Prerequisites

- Node.js
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/install-and-update/) (`npm install -g wrangler`)
- A Cloudflare account (free tier works)

### Setup

```bash
# Clone the repo
git clone https://github.com/debjyotibiswas/blinkhost-core.git
cd blinkhost-core

# Install dependencies
npm install

# Create local D1 database and apply migrations
npx wrangler d1 migrations apply blinkhost-prod --local

# Start local dev server
npx wrangler dev
```

The local server runs at `http://localhost:8787`.

### Test the API locally

```bash
# Health check
curl http://localhost:8787/v1/health
```

## Deploying to Cloudflare

```bash
# Login to Cloudflare
npx wrangler login

# Create R2 bucket and D1 database
npx wrangler r2 bucket create blinkhost-sites-prod
npx wrangler d1 create blinkhost-prod

# Update wrangler.toml with your D1 database_id

# Apply migrations to remote D1
npx wrangler d1 migrations apply blinkhost-prod

# Deploy
npx wrangler deploy
```

You also need wildcard DNS configured on Cloudflare:
- `A` record for `@` pointing to `192.0.2.1` (proxied)
- `A` record for `*` pointing to `192.0.2.1` (proxied)

## API Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/sites` | Create a site and get upload URLs |
| `PUT` | `/v1/upload/:token` | Upload a file to its presigned URL |
| `POST` | `/v1/sites/:slug/finalize` | Finalize publish, make site live |
| `GET` | `/v1/sites/:slug` | Get site metadata |
| `PUT` | `/v1/sites/:slug` | Update an existing site (requires manage token) |
| `DELETE` | `/v1/sites/:slug` | Delete a site (requires manage token) |
| `GET` | `/v1/sites/:slug/version` | Version check for auto-refresh polling |
| `GET` | `/v1/health` | Health check |

## Using the Agent Skill

Agents can install the Blinkhost skill to publish directly:

```bash
curl -fsSL https://blinkhost.co/install.sh | bash
```

This installs a `publish.sh` script that handles the full create -> upload -> finalize flow.

```bash
# Publish a directory
publish.sh /path/to/site

# Update an existing site
publish.sh --site-slug <slug> --manage-token <token> /path/to/site
```

## Key Features

- **Instant publishing** -- one API call to create, upload files, finalize
- **Auto-refresh** -- viewers see updates automatically (3s polling)
- **Collaborative editing** -- share the manage token to let multiple agents edit one site
- **Content-addressed storage** -- automatic deduplication
- **TTL-based expiry** -- anonymous sites expire after 24 hours by default

## Limits (Free Tier)

| Limit | Value |
|-------|-------|
| Max file size | 1 MB |
| Max files per site | 100 |
| Max total upload | 5 MB |
| Anonymous site TTL | 24 hours |

## Roadmap

The platform is designed to evolve through stages:

- **V1** -- Core publish flow (create, upload, finalize, serve, delete)
- **V2** -- Multi-user collaborative editing, auto-refresh, mission control view
- **V3** -- Full platform: auth, accounts, incremental deploys, SPA mode, handles, custom domains, proxy routes, access controls

## License

Proprietary
