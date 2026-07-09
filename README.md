# Madden Franchise Discord Bot

A production-oriented Discord.js v14 bot for Connected Franchise leagues using only organizer-provided or officially supported export data. It **does not** log in to EA, scrape authenticated services, reverse-engineer endpoints, or retain EA credentials.

## Architecture

`Discord gateway -> command handlers -> permissions -> snapshot service -> provider adapter -> official/export URL`

- **Discord bot:** slash commands, embeds, autocomplete, deferred replies.
- **Fastify:** health check, install-link helper, internal status route (place this behind auth in production).
- **PostgreSQL/Prisma:** league configuration, immutable JSON snapshots, audit log.
- **Provider adapter:** `ExportProvider` isolates ingestion. `JsonProvider` accepts a validated organizer URL or local payload; add CSV/Companion exports by implementing the same interface.
- **Sync:** cron task, 3 attempts with exponential delay, state/error/audit recording.

## Project tree

```text
src/{api.ts,config.ts,db.ts,index.ts,types.ts}
src/discord/{commands.ts,embeds.ts,handlers.ts,register-commands.ts}
src/providers/{base.ts,json.ts}
src/services/{normalize.ts,permissions.ts,sync.ts}
prisma/schema.prisma
sample/league.json
tests/{normalize,provider,permissions}.test.ts
Dockerfile docker-compose.yml .env.example
```

## Local setup

1. `cp .env.example .env`; set Discord client ID/token and a PostgreSQL `DATABASE_URL`.
2. For Docker, use `DATABASE_URL=postgresql://madden:madden@postgres:5432/madden?schema=public`. For host `npm run dev`, change host to `localhost`.
3. Add the exact hostname serving your owner-approved HTTPS JSON to `EXPORT_ALLOWED_HOSTS`. This is SSRF protection; never use a wildcard.
4. `npm install && npm run prisma:generate && npm run prisma:migrate && npm run commands && npm run dev`.
5. In Discord, create a **Commissioner** role, assign it to the league owner, then run `/league connect`. For a local mock, serve `sample/league.json` from an allowed HTTPS host or use the `MOCK` payload through a controlled admin API implementation.

## Discord installation

Create an application and bot in the Discord Developer Portal. Set the redirect URI if using the browser OAuth callback. Install with `bot` and `applications.commands` scopes; use the `/invite` endpoint to generate a baseline install URL. Register to `DISCORD_TEST_GUILD_ID` during development, then omit it for global registration.

## Data contract

See `sample/league.json`. At a minimum export `leagueName`, `season`, `updatedAt`, teams (name/wins/losses), and schedule. Optional roster, injury, cap, transaction, and stats fields safely degrade to “not supplied.” Image URLs are only rendered if present in trusted normalized data; validate/logo-proxy them before enabling external images in production.

## Production deployment

- **Railway/Render/Fly/VPS:** deploy the Dockerfile, attach managed Postgres, set every `.env.example` variable as a secret, and run `npx prisma migrate deploy` at release.
- Set a stable `DISCORD_REDIRECT_URI` with HTTPS, `NODE_ENV=production`, a strict allow-list, log aggregation, database backups, and an authenticated reverse proxy for API routes.
- Run one bot/scheduler instance, or use a distributed lock before scaling workers to prevent duplicate syncs.

## Tests

`npm test` runs parser and standings tests. Expand permission tests with mocked Prisma/Discord interactions and add fixtures for every official export format before accepting it.

## Security policy

Store Discord IDs, minimal league configuration, snapshots, and audit events only. Treat export URLs as secrets: never return them in embeds, logs, public API responses, or error messages. Use TLS, least-privilege DB credentials, strict URL allow-lists, size limits for uploads, malware scanning/object storage for uploads, and expiry/deletion policies for snapshots.


## GitHub to Railway continuous deployment

1. Create a **private** GitHub repository and upload this entire project (including `railway.toml`, `prisma/migrations`, and `.github/workflows`).
2. In Railway, create a project, add PostgreSQL, then choose **New → GitHub Repo** and select the repository. Railway reads `railway.toml`, builds the Dockerfile, runs the Prisma pre-deploy migration, and starts the bot.
3. In Railway Variables, set `DATABASE_URL=${{Postgres.DATABASE_URL}}` and add all Discord/export secrets from `.env.example`. Generate a Railway domain and configure `/health` as the health check.
4. In the Railway service settings, select `main` as the trigger branch and enable **Wait for CI**. Every successful push to `main` then runs GitHub validation and Railway deploys it.
5. The optional manual workflow is a fallback only. Create a Railway project token and GitHub repository secrets named `RAILWAY_TOKEN` and `RAILWAY_SERVICE`; run it from Actions → Railway manual deploy. Do not enable it for every push when Railway GitHub autodeploy is enabled, or you can produce duplicate deployments.
