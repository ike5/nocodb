# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Layout

This is a **pnpm + lerna monorepo**. Use `pnpm` (not `npm`/`yarn`) — `preinstall` enforces this. The workspace packages are in `packages/`:

- **`packages/nocodb`** — backend (NestJS 10, Node ≥22). Bundled with rspack + SWC; no `tsc` build step.
- **`packages/nc-gui`** — frontend (Nuxt 3, Vue 3, Pinia, Ant Design Vue, Windi CSS). SPA-mode (`ssr: false`), bundled into `nc-lib-gui` and served by the backend in production.
- **`packages/nocodb-sdk`** — TypeScript SDK generated from the backend OpenAPI spec (`pnpm generate:sdk`). Imported by both backend and frontend as `workspace:^`.
- **`packages/nocodb-sdk-v2`** — newer SDK built with rslib + Biome.
- **`packages/nc-lib-gui`** — packaged frontend dist consumed by the backend.
- **`packages/noco-integrations`** — separate pnpm workspace of integration packages (`@noco-integrations/*`). Built independently, then `registerIntegrations` wires them into the backend by scanning `package.json` for `@noco-integrations/*` deps and generating `src/integrations/index.ts` / `src/ee/integrations/index.ts`.
- **`packages/nc-secret-mgr`** — standalone CLI for rotating secrets in the meta DB.
- **`packages/nc-mail-templates`**, **`packages/nc-mail-assets`** — React Email templates.

The root `tsconfig.json` is a solution-style config with `"include": []` and project references — keep it that way so the TS server doesn't try to load 5000+ files in a single program.

## Common Commands

### Bootstrap & install
```bash
pnpm bootstrap            # installs sdk, builds it, installs nocodb+nc-gui+mail-templates, builds & registers integrations
pnpm run integrations:build
pnpm run registerIntegrations   # re-run after adding a @noco-integrations/* dep
```

### Run locally (two terminals)
```bash
pnpm run start:backend    # rspack watch + nodemon-like restart, default entrypoint src/run/docker
pnpm run start:frontend   # nuxt dev on http://localhost:3000 → proxies API to backend on :8080
```
Backend entrypoint is controlled by the `ENTRYPOINT` env var — alternates:
```bash
pnpm --filter=nocodb run watch:run:pg      # uses src/run/dockerRunPG.ts (PostgreSQL meta DB)
pnpm --filter=nocodb run watch:run:mysql   # uses src/run/dockerRunMysql.ts
```
Other entrypoints in `packages/nocodb/src/run/`: `local.ts`, `cloud.ts`, `timely.ts`, `testDocker.ts`.

### Lint & format
```bash
pnpm --filter=nocodb run lint          # eslint --fix on backend src
pnpm --filter=nc-gui  run lint         # eslint --fix on frontend (incl. .vue)
pnpm --filter=nocodb run format        # prettier on src and test
```

### Tests
- **Backend unit/integration tests** use Jest:
  - `jest.config.js` has `testRegex: '(Integration|Source)\\.spec\\.ts$'` and `rootDir: 'src'`, so `pnpm --filter=nocodb test` only picks up `*Integration.spec.ts` / `*Source.spec.ts`. The hundreds of `*.spec.ts` files next to controllers/services are NOT executed by the default `test` script — they're typically run individually or via IDE.
  - Run a single file: `pnpm --filter=nocodb exec jest src/path/to/file.spec.ts --runInBand`
  - E2E config: `pnpm --filter=nocodb run test:e2e` (uses `test/jest-e2e.json`).
- **Frontend tests** use Vitest with config `packages/nc-gui/test/vite.config.ts`:
  - `pnpm --filter=nc-gui exec vitest run --config test/vite.config.ts`
  - Specs live in `packages/nc-gui/test/*.test.ts`, environment is `happy-dom`.
- **SDK tests**: `pnpm --filter=nocodb-sdk run test`.

### Meta DB migrations
Don't hand-create migration files. Use the generator (writes to `packages/nocodb/src/meta/migrations/v0/`):
```bash
node scripts/addMigration.js add my feature
```
There are multiple Knex `MigrationSource` classes under `src/meta/migrations/`: `XcMigrationSource{,v0,v2,Audit,ChatMessages,DocsContent,OperationLogs}.ts` — pick the matching one when wiring a new migration in.

## Architecture Notes

### Backend (NestJS, but not idiomatic)
- `src/Noco.ts` is the bootstrap class — instantiated by every entrypoint in `src/run/`. It builds the Nest app from `AppModule` (`src/app.module.ts`) and exposes the Express router.
- `src/modules/noco.module.ts` is the giant aggregate module: every controller and service the OSS edition exposes is listed here explicitly. Controllers are **conditionally registered** when `NC_WORKER_CONTAINER !== 'true'` — in worker mode the same module runs without HTTP controllers, only background jobs.
- The codebase ships hooks for an enterprise edition: `src/ee/`, `src/ee-on-prem/`, `src/ee-cloud/` are excluded from this OSS tree but referenced in `tsconfig` paths (`~/*` resolves to `src/ee/*` first, then `src/`) and in Jest's `moduleNameMapper`. When you see `~/foo`, that's the alias — don't write relative `../../` imports.
- `src/db/` is the SQL layer:
  - `BaseModelSqlv2.ts` (and the `BaseModelSqlv2/` folder) is the central data-access class — every CRUD/query path on user data goes through it.
  - `CustomKnex.ts` wraps Knex with NocoDB-specific dialect quirks.
  - `sql-client/`, `sql-mgr/`, `sql-migrator/` handle external data-source connections, schema sync, and per-source migrations (distinct from meta-DB migrations).
  - `formulav2/`, `conditionV2.ts`, `sortV2.ts`, `genRollupSelectv2.ts`, `generateLookupSelectQuery.ts`, `mapFunctionName.ts` — the query-building primitives behind formula/rollup/lookup columns.
  - `field-handler/` — per-column-type read/write logic.
- `src/meta/meta.service.ts` is the meta-DB facade (Knex). `MetaTable` constants are the canonical table names. Every persistent backend model under `src/models/` is built on top of it.
- `src/modules/jobs/` runs background jobs. Two backends: BullMQ (Redis) when `NC_REDIS_URL` is set, otherwise `fallback/` in-process emitter. `NC_WORKER_CONTAINER=true` runs a job-only worker process.
- `src/integrations/` is generated by `build-utils/registerIntegrations.js` — DO NOT hand-edit `src/integrations/index.ts` or `src/ee/integrations/index.ts`. Add a dep to `packages/nocodb/package.json` (or drop a package into `packages/noco-integrations/packages/`) and re-run `pnpm run registerIntegrations`.
- `src/plugins/` contains storage/email/chat plugins (S3, SES, SMTP, Slack, etc.) — these are the legacy "app store" plugins, separate from `noco-integrations`.
- `src/version-upgrader/` runs idempotent upgrades against existing meta DBs on startup.

### Frontend (Nuxt 3, but not idiomatic)
- `ssr: false` — this is a pure SPA. Dashboard lives at `/dashboard`. Pages under `pages/` use catch-all routes (`pages/nc/[baseId].vue` etc.) and most state lives in **composables** (`composables/`, 130+ files) and Pinia stores (`store/`), not in components.
- Component registry is auto-generated via `unplugin-vue-components` — you generally don't need to `import` Vue components by hand.
- `extensions/` contains the in-app extensions framework (separate from `packages/noco-integrations`).
- `nocodb-sdk` is the wire format — change a backend DTO and the SDK must be regenerated and rebuilt before the frontend will type-check against it.

### Cross-cutting
- The frontend and backend share types through `nocodb-sdk`. When changing API shapes, update the controller/service, regenerate the SDK (`pnpm --filter=nocodb-sdk run build`), then update frontend consumers.
- Realtime updates flow through Socket.IO (`src/gateways/socket.gateway.ts` on the backend, `useRealtime` on the frontend), optionally backed by Redis (`@socket.io/redis-adapter`) when `NC_REDIS_URL` is set.
- `NC_WORKER_CONTAINER=true` is the toggle for "this process is a job worker, not an API server" — used in production to scale jobs separately.

## Conventions

- **Commits**: Conventional Commits (`feat:`, `fix:`, `chore:`, `feat(scope)!:` for breaking). See `.github/COMMIT_CONVENTION.md`.
- **Branches & PRs**: Branch from and target `develop`, NOT `master`. `master` is just the last stable release. PRs are squash-merged.
- **License**: Sustainable Use License (not OSI-open-source). All packages inherit it.
- **Supply chain**: `pnpm-workspace.yaml` enforces a 7-day `minimumReleaseAge` and `blockExoticSubdeps: true`. New dep versions younger than 7 days won't install — use `minimumReleaseAgeExclude` for emergency security patches.
- **Path aliases**: `~/foo` and `src/foo` and `@/foo` in the backend all resolve to `src/foo` (with `ee/` checked first). Use them; don't write `../../`.
