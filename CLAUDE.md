<!-- HEADY_BRAND:BEGIN -->
<!-- HEADY SYSTEMS :: SACRED GEOMETRY -->
<!-- FILE: CLAUDE.md -->
<!-- LAYER: root -->
<!--  -->
<!--         _   _  _____    _    ____   __   __ -->
<!--        | | | || ____|  / \  |  _ \ \ \ / / -->
<!--        | |_| ||  _|   / _ \ | | | | \ V /  -->
<!--        |  _  || |___ / ___ \| |_| |  | |   -->
<!--        |_| |_||_____/_/   \_\____/   |_|   -->
<!--  -->
<!--    Sacred Geometry :: Organic Systems :: Breathing Interfaces -->
<!-- HEADY_BRAND:END -->

# CLAUDE.md — HeadyEcosystem

## What This Repo Is

**HeadyEcosystem** (`HeadySystems/HeadyEcosystem`) is a pnpm + Turborepo monorepo hosting the unified development platform for two organizations: **HeadyConnection** (nonprofit) and **HeadySystems** (C-Corp). It contains two Next.js web frontends, a shared Express/Prisma backend API, a Playwright browser-automation service, and a Docker Compose stack that includes an optional **Drupal 11 headless CMS** integration (JSON:API-based hybrid content management, synced into the API's Postgres database).

Key top-level docs:
- `README.md` — platform overview, quick start, project structure
- `DRUPAL_CMS_GUIDE.md` — Drupal 11 hybrid CMS architecture and setup
- `HEADYCONDUCTOR_DELEGATION.md` — HeadyConductor delegation spec for the Drupal integration work (status, tasks, success criteria)

## Workspace Layout

`pnpm-workspace.yaml` declares two workspace globs:

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

Actual contents:

| Path | Package Name | Purpose |
|------|--------------|---------|
| `apps/api` | `@heady/api` | Shared backend: Express 4 + Prisma 5 (PostgreSQL) + Redis + Socket.IO + Zod. Routes: `/api/tasks`, `/api/content` (incl. `/api/content/drupal-sync`, `/drupal-push`, categories/tags), `/health`, `/api`. Services in `src/services/` (`task.service.ts`, `drupal-sync.service.ts`). Prisma schema (`apps/api/prisma/schema.prisma`) models: User, Organization, Project, ProjectMember, Task, AutomationConfig, Content, Tag, Category, DrupalSyncLog. |
| `apps/web-heady-connection` | `@heady/web-heady-connection` | Nonprofit public app. Next.js 14 App Router (`src/app/`), runs on port 3000. |
| `apps/web-heady-systems` | `@heady/web-heady-systems` | Commercial product app. Next.js 14 App Router, runs on port 3001 (`next dev -p 3001`). |
| `services/browser-automation` | `@heady/browser-automation` | Playwright 1.41 automation service with Express + Socket.IO. **Not** covered by the workspace globs (`services/*` is not in `pnpm-workspace.yaml`); it has its own Dockerfile and is orchestrated via docker-compose. |

**Important gap to know about:** the `packages/` directory referenced by the workspace glob, `tsconfig.base.json` path aliases, and the README structure diagram (`packages/ui`, `packages/core-domain`, `packages/infra`) does **not exist in this repo yet**. Both web apps declare `"@heady/ui": "workspace:*"` and `"@heady/core-domain": "workspace:*"` dependencies against those missing packages. Extracting shared UI into `packages/ui` is a pending task per `HEADYCONDUCTOR_DELEGATION.md`. Do not assume those packages resolve; check before building on them.

## Tech Stack (verified versions)

- **Runtime / tooling** (root `package.json`): Node `>=20.0.0`, pnpm `>=8.0.0` (`packageManager: pnpm@8.12.0`), Turborepo `^1.11.2`, TypeScript `^5.3.3`, ESLint `^8.56.0`, Prettier `^3.1.1`, Husky `^8.0.3`, lint-staged `^15.2.0`
- **Frontends:** Next.js `^14.1.0`, React `^18.2.0`
- **API:** Express `^4.18.2`, Prisma `^5.7.1`, Redis client `^4.6.11`, Socket.IO `^4.6.0`, Zod `^3.22.4`, JWT (`jsonwebtoken ^9.0.2`) + bcrypt, Helmet, CORS; tests via Jest `^29.7.0`
- **Automation:** Playwright `^1.41.1`
- **Infra (docker-compose):** postgres:16-alpine, redis:7-alpine, drupal:11-php8.3-apache, mariadb:10.11, cloudflare/cloudflared

## Development Workflow

Root scripts (all real, from `package.json`):

```bash
pnpm install          # install all workspace deps
pnpm dev              # turbo dev  (persistent, uncached)
pnpm build            # turbo build (dependsOn ^build; outputs dist/**, .next/**)
pnpm lint             # turbo lint
pnpm test             # turbo test
pnpm format           # prettier --write "**/*.{js,jsx,ts,tsx,json,md}"
pnpm run setup:env    # node scripts/setup-environment.js
pnpm run setup:desktop # node scripts/setup-desktop-icons.js
pnpm run docker:up    # docker-compose up -d
pnpm run docker:down  # docker-compose down
pnpm run docker:logs  # docker-compose logs -f
```

Notes:
- `turbo.json` defines pipeline tasks for `build`, `lint`, `dev`, and `clean` only. The root `test` script invokes `turbo test`, and CI additionally runs `pnpm turbo run typecheck` — neither `test` nor `typecheck` is declared in `turbo.json`'s pipeline, so verify task wiring before relying on them.
- `deploy:dev` / `deploy:staging` / `deploy:prod` root scripts chain to `deploy:*:backend` / `deploy:*:frontend` scripts that are not defined in the root `package.json` — they will fail as-is.
- Per-package scripts: `@heady/api` has `dev` (nodemon + ts-node), `build` (tsc), `start`, `test` (jest), `lint` (eslint); both web apps have standard Next `dev`/`build`/`start`/`lint`; browser-automation has `dev`/`build`/`start` (no lint/test).
- Environment setup: `cp .env.example .env` and fill in values. All secrets are `HC_*`-prefixed env vars (JWT secret, API keys, Cloudflare tunnel token, Drupal API key) — never hardcode them.

### `hc.bat` (Windows CLI)

```
hc -a | sysmonitor      → node scripts/sysmonitor.mjs
hc realmonitor [prompt] → python scripts/realmonitor.py
hc tasks list|create    → node scripts/task_cli.mjs
```

### `scripts/`

`brand_headers.js`, `init-db.sql` (mounted as Postgres init script), `realmonitor.py`, `setup-desktop-icons.js`, `setup-environment.js`, `sysmonitor.mjs`, `task_cli.mjs`, `verify-deployment.js`.

## Docker Compose Stack

Defined in `docker-compose.yml`; all services join the `heady-network` bridge network. Host-port mappings are env-overridable defaults as written in the file:

| Service | Image / Build | Host Port (default) | Notes |
|---------|---------------|---------------------|-------|
| `postgres` | postgres:16-alpine | `${DATABASE_PORT:-5433}→5432` | Seeded by `scripts/init-db.sql`; healthcheck via `pg_isready` |
| `redis` | redis:7-alpine | `${REDIS_PORT:-6380}→6379` | |
| `api` | `apps/api/Dockerfile` | `${HC_API_PORT:-8001}→8000` | Gets `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET` (`HC_JWT_SECRET`) |
| `web-heady-connection` | `apps/web-heady-connection/Dockerfile` | `${HC_CONNECTION_PORT:-3000}→3000` | `NEXT_PUBLIC_API_URL: http://api:8000` (internal network name) |
| `web-heady-systems` | `apps/web-heady-systems/Dockerfile` | `${HC_SYSTEMS_PORT:-3001}→3001` | |
| `browser-automation` | `services/browser-automation/Dockerfile` | `${BROWSER_PORT:-9222}→9222` | `shm_size: 2gb` for Playwright |
| `cloudflared` | cloudflare/cloudflared | — | **profile `tunnel`**; needs `HC_CLOUDFLARE_TUNNEL_TOKEN` |
| `drupal` | drupal:11-php8.3-apache | `${DRUPAL_PORT:-8080}→80` | **profile `drupal`** |
| `drupal-db` | mariadb:10.11 | — | **profile `drupal`** |

Start with Drupal CMS: `docker-compose --profile drupal up -d`. Start the Cloudflare tunnel: `--profile tunnel`.

## CI/CD

`.github/workflows/ci.yml` runs on push/PR to `main` and `develop`:
1. **test job** — pnpm 8 / Node 20.x with Postgres 16 + Redis 7 service containers; runs `pnpm lint`, `pnpm turbo run typecheck`, `pnpm test`, `pnpm build`.
2. **deploy-dev** (on `develop`) — Render deploy hook for the API + Cloudflare Pages for `apps/web-heady-connection/out`.
3. **deploy-staging / deploy-production** (on `main`) — GitHub environments `staging` (staging.headyconnection.org) and `production` (headyconnection.org); the actual deploy commands are still stubs (`echo` placeholders) in the workflow.

## Conventions

- **Brand header:** every source/config/doc file starts with the `HEADY_BRAND:BEGIN` / `HEADY_BRAND:END` ASCII-art block (see any existing file for the exact template). Preserve it when editing; add it when creating files.
- **TypeScript strictness** (`tsconfig.base.json`): `strict: true` plus `noUnusedLocals`, `noUnusedParameters`, `noImplicitReturns`, `noFallthroughCasesInSwitch`, `forceConsistentCasingInFileNames`. Target ES2022, CommonJS modules. Path aliases `@heady/ui`, `@heady/core-domain`, `@heady/infra` map to `packages/*/src` (packages not yet present — see Workspace Layout).
- **Package naming:** workspace packages use the `@heady/` scope and are `private: true`.
- **Formatting/linting:** Prettier (`pnpm format`) + ESLint with `eslint-config-prettier`; Husky (`prepare: husky install`) + lint-staged are configured as devDependencies.
- **Editor defaults** (`.editorconfig`): consistent indentation/encoding across the repo.
- **Commits:** conventional commits (per `README.md` contributing section).
- **Secrets:** all credentials come from env vars (`.env`, populated from `.env.example`); `HC_*` prefix for Heady-specific keys. No secrets in code or committed config.
- **MCP:** `.mcp/config.json` registers MCP servers (heady core-domain, jules, huggingface, playwright, github-gists) with env-var-injected tokens.
- **Windsurf workflows:** `.windsurf/workflows/` contains `setup-local.md`, `start-tunnel.md`, `stop-tunnel.md`, `verify-deployment.md`.

## Licensing Split

HeadyConnection components are Apache 2.0 (open source); HeadySystems components are proprietary. Keep the nonprofit/commercial separation in mind when moving code between `web-heady-connection` and `web-heady-systems`.
