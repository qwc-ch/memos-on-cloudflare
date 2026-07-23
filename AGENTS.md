# AGENTS.md — memos-on-cloudflare

A port of [Memos](https://github.com/usememos/memos) from Go+SQLite to Cloudflare Workers + D1 + R2 + Workers AI.

## Project structure

- **Root** — deploy orchestrator (`package.json` scripts, `wrangler.toml`)
- **`worker/`** — Hono backend (TypeScript, Cloudflare Workers), entry `worker/src/index.ts`
- **`web/`** — React + Vite + TailwindCSS 4 frontend, separate `package.json`
- **`migrations/`** — D1 SQL migrations (apply in order via wrangler)
- **`scripts/fetch-github-version.mjs`** — used in CI to set `APP_VERSION` from git tags

## First-time setup

```bash
npm install && cd web && npm install && cd ..
# Create D1 + R2 resources (one-time):
wrangler d1 create cfmemos-db
wrangler r2 bucket create cfmemos
# Copy the returned database_id into wrangler.toml [[d1_databases]] database_id
# Then:
npm run db:migrate:remote   # push schema to remote D1
wrangler secret put JWT_SECRET   # production only
```

## Key commands (run from root unless noted)

| Command | What it does |
|---------|-------------|
| `npm run dev` | Start wrangler dev server on `:8787` |
| `npm run dev:web` | Start Vite on `:3001` (proxies `/api`, `/file` to worker) |
| `npm run build:web` | Build frontend into `web/dist/` |
| `npm run deploy` | Build frontend + `wrangler deploy` |
| `npm run db:migrate` | Apply migrations to **local** D1 |
| `npm run db:migrate:remote` | Apply migrations to **remote** D1 |
| `cd web && npm run lint` | `tsc --noEmit --skipLibCheck && biome check src` |
| `cd web && npm run test` | Vitest (jsdom) — tests in `web/tests/` |
| `cd web && npm run format` | Biome format on `web/src/` |

**Local dev** requires two terminals: `npm run dev` + `npm run dev:web`.

**No root-level lint/typecheck script exists** — only `web/` has them.

## Architecture notes

- **Frontend is REST, not gRPC** — `web/src/connect.ts` wraps `fetch` calls with a protobuf-like interface. Shims under `web/src/shims/` stub out `@bufbuild/protobuf` and `@connectrpc/connect`.
- **`wrangler.toml`** must list `run_worker_first = ["/api/*", "/file/*", "/u/*", "/explore/rss.xml"]` in `[assets]`, otherwise Cloudflare's static asset layer catches API paths before the Worker.
- **Auth**: JWT HS256 (15m access, 30d refresh via jose), plus PATs (`memos_pat_*`). Auth middleware in `worker/src/middleware/auth.ts` accepts `Authorization: Bearer`, `X-API-Key`, `Memos-Access-Token`, or `Token` headers.
- **R2** bound as `BUCKET` for file attachments, **KV** as `CACHE` (optional, for memo caching).
- **Webhook URLs** are validated against private IP blocks (RFC 1918, loopback, link-local) and rejected if insecure.

## CI / Deploy

Push to `main` triggers `.github/workflows/deploy.yml`:
1. Install root + web deps
2. `npm run build:web`
3. Run `scripts/fetch-github-version.mjs` to derive `APP_VERSION`
4. `npx wrangler d1 migrations apply cfmemos-db --remote`
5. `npx wrangler deploy --keep-vars --var "APP_VERSION:..."`

Requires secrets: `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`.

## Quirks & gotchas

- `web/tsconfig.json` uses `skipLibCheck: false` and `noUnusedLocals`/`noUnusedParameters: true` (stricter than root).
- Root `tsconfig.json` only covers `worker/src/**/*.ts`, not `web/`.
- Vitest config (`web/vitest.config.mts`) is **separate** from Vite config — both duplicate the `@/` alias.
- Web uses **npm** (has `.npmrc` with `install-strategy=nested`), despite having `pnpm-workspace.yaml` (only used to allow esbuild builds).
- The web `connect.ts` client normalizes protobuf-style enums to REST JSON (state 1→2 maps to `NORMAL`→`ARCHIVED`, visibility 1→3 maps to `PRIVATE`→`PUBLIC`).
- D1 does not support foreign key enforcement (`PRAGMA foreign_keys`); all referential integrity is in application code.
- `fetch-github-version.mjs` requires `GITHUB_TOKEN` with repo access; falls back to `1.0.0`.
- `wrangler deploy --keep-vars` preserves secrets set via `wrangler secret put`.
