# Backend developer guide

Repo: [`Twenty-backend`](../../../Twenty-backend/README.md). Spec: [`backend-tech-stack.md`](../architecture/backend-tech-stack.md). The repo's `AGENTS.md` is a compact summary of the rules, written for coding agents and equally useful for people.

Stack: TypeScript on Node 22, Hono + `@hono/zod-openapi`, Zod, PostgreSQL 17 + Drizzle, pg-boss (jobs and cron, no Redis), `jose` for auth, Qwen behind an `AiProvider` interface, pino, Vitest + Testcontainers, Docker Compose.

---

## 1. Prerequisites

| Tool | Version | Why |
|---|---|---|
| Node.js | ≥ 22.12 | `engines` in `package.json`. On < 22.19, `openai` stays pinned to v6 and `testcontainers` to v11 (their newer versions pull in undici 8, which needs 22.19). |
| Docker | any recent | Local Postgres, integration tests (Testcontainers), building the image |
| `../Twenty` checked out next to the repo | — | The rules tests read `../Twenty/contracts/rules/fixtures.json` |

## 2. Local setup

```sh
npm ci
cp .env.example .env     # set JWT_SECRET (≥ 32 chars) and APPLE_CLIENT_IDS
                         # AI_PROVIDER=fake runs without a Qwen key or network
docker run -d --name twenty-pg -p 5432:5432 \
  -e POSTGRES_USER=twenty -e POSTGRES_PASSWORD=twenty postgres:17-alpine
npm run migrate:dev      # applies src/db/migrations
npm run dev              # API on :3000 (spec at /openapi.json, health at /healthz)
npm run dev:worker       # jobs and cron, in a second terminal
```

The API and worker are separate processes from the same code. Without the worker, start/finish still work, but auto-finish, missed-event sweeps and AI breakdowns never run.

## 3. Architecture tour

```
src/rules/     pure rules: grid, snap, start, finish, extend + cascade, overlap, tokens, time zones
src/services/  business operations; own transactions and locking
src/routes/    Hono routes + Zod schemas (the OpenAPI source)
src/auth/      Apple ID token verification, JWT access tokens, refresh rotation
src/ai/        AiProvider interface, qwen.ts, fake.ts, prompt + output validation
src/jobs/      pg-boss queue names, options and handlers
src/db/        Drizzle schema, plain-SQL migrations, UUIDv7
src/lib/       errors, logger, clock, pagination
server.ts / worker.ts / migrate.ts   entry points (dist/*.js in the image)
```

What goes where:

- **`src/rules` is pure.** Inputs in, outputs out: no `Date.now()`, no DB, no env. If you need "now", the service passes it in from `deps.clock`. This is what makes the rules testable against the shared fixtures and portable to Swift.
- **Services own correctness.** A service opens the transaction (`transaction()` in `src/db/client.ts`), calls `lockUser` (`SELECT … FOR UPDATE` on the user row) before touching the calendar or ledger, loads state, calls the rules, and writes. Per-user writes are therefore serialised.
- **Routes stay thin.** Validate with Zod, call one service, shape the response. The Zod schemas *are* the API contract.
- **Jobs are enqueued inside the transaction.** `transaction()` exposes an `executeSql` that pg-boss uses, so an `events.auto_finish` job exists exactly when the start that needs it committed. Never enqueue after commit.
- **Only `src/ai` knows the provider.** Everything else uses `AiProvider`. Claude and OpenAI aren't available in mainland China or Hong Kong; keeping this boundary lets a global deployment switch providers by config.

## 4. Common workflows

### Add or change an endpoint
1. Define or extend Zod schemas in `src/routes/schemas.ts` (shared) or the route file.
2. Register the route with `createRoute` in the matching `src/routes/*.ts`. Reuse `IdempotencyHeaders`, `PageQuery`, `errors`, `security` from `src/routes/common.ts`.
3. Put the logic in a service; throw `AppError(status, 'snake_case_code', message)` for expected failures.
4. `npm run openapi` to regenerate `contracts/openapi.yaml`, then commit it **with** the route change. `test/openapi.test.ts` fails if they drift.
5. Tell the client developers: they regenerate their clients from the new yaml.

### Change the database schema
1. Edit `src/db/schema.ts`.
2. `npm run db:generate` writes a new SQL file to `src/db/migrations/`. Read it; hand-edit only to add what Drizzle can't express (triggers, seeds), as `0001_ledger_append_only_and_seed.sql` does.
3. `npm run migrate:dev`, run the tests, commit schema + migration + snapshot together.
4. Migrations only go forward. Never edit one that has been deployed.

### Change a rule
1. Rules are product decisions: confirm against the spec, or ask. Don't invent timing or token behaviour.
2. The fixtures change first (in `../Twenty`, by whoever owns the rules). Then update `src/rules`, run `npm run test:rules`, and let the iOS developers know the Swift port must follow.

### Change reward values
Insert a new `reward_rules` row with the next version. Never update an existing row: ledger entries record the version they were paid under.

### Add a background job
1. Add the name to `JOBS` in `src/jobs/queue.ts` (with retry options if it needs them); `ensureQueues` creates it on boot for both the API and the worker.
2. Add the handler in `src/jobs/handlers.ts` and register it in `src/worker.ts` (plus a `schedule` call for cron jobs).
3. Enqueue through the `Queue` interface, passing the transaction when the job must exist only if the write commits.

## 5. Invariants you must not break

| Invariant | How it's enforced |
|---|---|
| Token ledger is append-only; corrections are new entries | DB trigger rejects UPDATE/DELETE/TRUNCATE on `token_ledger` |
| `users.token_balance` equals the ledger sum | `appendLedgerEntry` (`src/services/ledger.ts`) updates both in one transaction |
| A Cadenza pays out once | Ledger `idempotency_key` is UNIQUE; rewards use `event_reward:<event_id>` |
| Retried writes don't repeat | `Idempotency-Key` middleware stores the first response (below 500) for 24 h and replays it |
| Concurrent edits don't clobber | Events and cues carry `version`; stale writes get `409 version_conflict` |
| Recorded times are the server's | Clients send raw `tapped_at`; `src/rules` computes the rest |
| Grid math in UTC | Only whole-hour time zones are supported (`isSupportedTimezone`) |
| Primary keys are UUIDv7 | `uuidv7()` in `src/db/ids.ts` |
| Every log line has a request ID | `requestContext` middleware + pino mixin (`src/lib/logger.ts`) |

## 6. Testing

```sh
npm test            # everything; Docker must be running (Testcontainers starts postgres:17-alpine)
npm run test:rules  # rules engine against fixtures.json (RULES_FIXTURES_PATH overrides the location)
npm run typecheck
```

- **Rules:** every fixture group (`snap`, `start`, `finish`, `tokens`, `extend`, `create`, `timezone`) must match exactly, with no extra keys; times compare as instants.
- **Integration:** `test/integration/harness.ts` boots a real Postgres, runs migrations and gives you an app with a `FixedClock` you can advance. Never mock the database for ledger, recording or transaction tests.
- **Contract:** `test/openapi.test.ts` fails if `contracts/openapi.yaml` is stale.
- Changes to rules, ledger or recording need tests. Report failing tests honestly; don't loosen assertions to pass.

## 7. Configuration

All config is environment variables, validated at boot by `src/config.ts` (the process refuses to start on bad config). `.env.example` is the template.

| Variable | Default | Notes |
|---|---|---|
| `DATABASE_URL` | — (required) | Compose sets it for containers |
| `PORT`, `LOG_LEVEL`, `DATABASE_POOL_MAX` | 3000, info, 10 | |
| `JWT_SECRET` | — (required) | ≥ 32 characters |
| `ACCESS_TOKEN_TTL_SECONDS`, `REFRESH_TOKEN_TTL_DAYS` | 900, 60 | |
| `APPLE_CLIENT_IDS` | — (required) | Comma-separated: iOS bundle ID, later the web Services ID |
| `TAP_MAX_AGE_HOURS`, `TAP_MAX_FUTURE_SKEW_SECONDS` | 24, 120 | Offline tap window (spec Q4) |
| `AI_PROVIDER` | qwen | `fake` for local dev and tests |
| `QWEN_API_KEY`, `QWEN_BASE_URL`, `QWEN_MODEL`, `AI_TIMEOUT_MS` | —, intl endpoint, qwen-plus, 60000 | Key required when `AI_PROVIDER=qwen` |
| `AI_DAILY_BREAKDOWN_LIMIT` | 5 | Per user per local day |
| `OSS_REGION`, `OSS_BUCKET`, `OSS_ACCESS_KEY_ID`, `OSS_ACCESS_KEY_SECRET`, `OSS_BACKUP_PREFIX` | —, …, `backups/` | `db.backup` is scheduled only when `OSS_BUCKET` is set |
| `BACKUP_CRON` | `30 19 * * *` | UTC; 03:30 Hong Kong time |
| `DOMAIN`, `POSTGRES_PASSWORD`, `TWENTY_IMAGE`, `POSTGRES_IMAGE`, `CADDY_IMAGE` | | Read by `docker-compose.yml`, not the app |

Never commit `.env`. Secrets (JWT secret, Qwen key, OSS keys) never go to clients.

## 8. Deploy and operations

The MVP is one server on Alibaba Cloud Hong Kong running `docker-compose.yml`:

| Service | Runs | Notes |
|---|---|---|
| `postgres` | Postgres 17 | Data in the `pgdata` volume |
| `migrate` | `node dist/migrate.js` once | api and worker wait for it to succeed |
| `api` | `node dist/server.js` | Stateless; port 3000 behind Caddy |
| `worker` | `node dist/worker.js` | pg-boss jobs and cron |
| `caddy` | reverse proxy | Automatic HTTPS for `DOMAIN` (`deploy/Caddyfile`) |

- **Image:** one `Dockerfile` for api, worker and migrate. In mainland China, build with `--build-arg NODE_IMAGE=<ACR mirror> --build-arg NPM_REGISTRY=https://registry.npmmirror.com`, and point `TWENTY_IMAGE`, `POSTGRES_IMAGE`, `CADDY_IMAGE` at Alibaba Container Registry copies. Docker Hub is unreliable from the mainland.
- **Health:** `GET /healthz` returns `{"status":"ok"}`.
- **Logs:** `docker compose logs -f api worker`. JSON lines; follow one request with its `request_id` (also returned as `X-Request-Id`).
- **Jobs:** `events.auto_finish` (delayed to E+5), `events.close_stale` (every 20 min), `ai.breakdown`, `db.backup` (nightly `pg_dump` to OSS), plus an hourly purge of stored idempotency keys.
- **Admin:** there's no admin API; use SQL (`docker compose exec postgres psql -U twenty`). Never UPDATE or DELETE ledger rows; insert an `adjustment` entry instead.

## 9. CI/CD

The repo has no CI configured yet. This is the recommended pipeline; the steps apply to any CI system.

**On every pull request** (all must pass before merge):
1. Check out `Twenty-backend` and `../Twenty` side by side (or set `RULES_FIXTURES_PATH`), so the rules tests find the fixtures.
2. `npm ci`
3. `npm run typecheck`
4. `npm test`. The runner needs Docker for Testcontainers. This includes the rules fixtures and the OpenAPI drift test.
5. Migrations in sync: `npm run db:generate` must produce no new file (`git status --porcelain src/db/migrations` is empty).
6. `npm run build`

**On merge to `main`:**
1. Build the image once and tag it with the git SHA (optionally also `latest`).
2. Push it to Alibaba Container Registry (Hong Kong), not Docker Hub.

**Deploy** (manual approval at first, while there is one server):
```sh
# on the server, in the compose directory
export TWENTY_IMAGE=registry.cn-hongkong.aliyuncs.com/<namespace>/twenty-backend:<sha>
docker compose pull
docker compose up -d         # migrate runs first; api and worker start after it succeeds
curl -fsS https://$DOMAIN/healthz
```
Roll back by deploying the previous SHA tag. Migrations only go forward, so a schema change must stay compatible with the previous image until the rollback window has passed. Split risky changes: add a column first, start using it in a later release.

Example GitHub Actions workflow (adapt the registry, secrets and fixtures checkout to your setup):

```yaml
# .github/workflows/ci.yml
name: ci
on:
  pull_request:
  push:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest          # has Docker, which Testcontainers needs
    steps:
      - uses: actions/checkout@v4
        with: { path: Twenty-backend }
      - uses: actions/checkout@v4   # the shared fixtures
        with: { repository: <owner>/Twenty, path: Twenty, token: ${{ secrets.FIXTURES_TOKEN }} }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm, cache-dependency-path: Twenty-backend/package-lock.json }
      - run: npm ci
        working-directory: Twenty-backend
      - run: npm run typecheck && npm test && npm run build
        working-directory: Twenty-backend
      - name: Migrations match schema
        working-directory: Twenty-backend
        run: npm run db:generate && test -z "$(git status --porcelain src/db/migrations)"
  image:
    needs: test
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: registry.cn-hongkong.aliyuncs.com
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: registry.cn-hongkong.aliyuncs.com/<namespace>/twenty-backend:${{ github.sha }}
```

`../Twenty` is not a git repository yet. Until it is, CI can't check it out: either make it a repo, or vendor a copy of `fixtures.json` into CI and point `RULES_FIXTURES_PATH` at it.

## 10. Decisions the spec left open

The backend made eleven calls the spec didn't cover: late offline taps, the close_stale backstop, account deletion, unsupported time zones, the day-end anchor, cue details, size caps, abandoned events, the AI limit's day, refresh-token reuse, and the default event kind. They are listed in the [backend README](../../../Twenty-backend/README.md#decisions-the-spec-left-open). If you change one, update that list.

## 11. Troubleshooting

| Symptom | Fix |
|---|---|
| `createZstdDecompress.bind` / undici error at startup or in tests | Node < 22.19 with an unpinned `openai` or `testcontainers`. Keep them at `openai@6` and `testcontainers@11`, or upgrade Node. |
| Integration tests hang or fail to start Postgres | Docker isn't running. |
| `contracts/rules/fixtures.json not found` | Put `../Twenty` next to the repo, or set `RULES_FIXTURES_PATH`. |
| 401s in a test after advancing the clock | Access tokens last 15 minutes of *fake* time. Use the harness's per-request tokens. |
| `tsx: command not found` | `node_modules` is missing (for example after removing a git worktree). Run `npm ci`. |
| `Invalid configuration: …` at boot | A required env var is missing or invalid; the message names it. |
| `422 unsupported_timezone` | The user's zone isn't a whole number of hours from UTC (India, Nepal, …). That's intentional (spec §3.1). |
