# Move execution into a durable worker

PortfolioPilot is a teaching project for a stock portfolio manager with portfolio-aware chat,
news, watchlists, and alerts. This activity changes **where chat runs**: the API saves a job in
PostgreSQL, and a separate background worker executes it. The React frontend remains the only UI.

## Purpose and learning goals

Previously, the API process owned chat execution in memory. Restarting that process could lose
the running task. Prompt 27 asks the coding agent to move execution into an independently running
worker while preserving authentication, approvals, cancellation, budgets, and streamed answers.

Students learn how to:

- Use a durable job: a task saved in a database that survives a process restart.
- Coordinate multiple workers with database locks and temporary ownership leases.
- Reject writes from an old worker after it loses ownership, a technique called **fencing**.
- Recover visible answer text after a browser reload without duplicating the final answer.
- Distinguish a safe retry before execution from an interrupted task that may already have incurred
  cost or applied an approved change.

PostgreSQL stores the authoritative job, messages, approvals, and budget records. Redis delivers
events; it does not own the durable job. The **transactional outbox** saves an event in the same
database transaction as its related change, then a separate worker publishes it. **SSE**
(Server-Sent Events) is the HTTP connection that delivers those updates to the browser.

## Steps performed during implementation

The following summarizes the completed implementation recorded in
[project state](docs/project-state.md) and [lesson 27](docs/lessons/27-durable-agent-worker.md).

1. **Inspect the existing chat, approval, budget, and session boundaries.** The implementation
   retained the existing mock/Claude Agent SDK adapters and authorized application tools. Installed
   SDK, Next.js, and Prisma definitions were checked; dependency versions were not upgraded.
2. **Restore dependencies and prepare a disposable database.** The recorded installation used:

   ```powershell
   npm ci --ignore-scripts --offline --cache .npm-cache
   ```

   Normal npm delegation initially failed under NVM. Using the installed Node 24.21.0/npm 11.19.0
   with the Node directory first on PATH worked. A Prisma cache permission error was resolved with
   an existing schema-engine copy inside the workspace. Sixteen migrations were applied to a fresh
   `portfolio_m27_verify` database on port 5546.
3. **Persist job ownership and progress.** Modified `packages/db/prisma/schema.prisma` and created
   `packages/db/prisma/migrations/20261014100000_durable_agent_jobs/migration.sql`. Created
   `packages/db/src/agent-jobs.ts`, `run-lease.ts`, and `run-progress.ts`; updated chat, approvals,
   and repository exports. Submission saves the queued run, user message, and budget reservation
   atomically, meaning they all commit together or none do.
4. **Extract execution into the worker.** Created `apps/worker/src/agent.ts`,
   `agent-execution.ts`, `agent-run-events.ts`, and `agent-budget-money.ts`; updated worker role
   selection and its environment example. Workers use PostgreSQL `SKIP LOCKED` to claim available
   work without waiting on rows another worker has locked. Each worker handles one run at a time.
5. **Make the API a submission and control boundary.** Updated conversation run submission,
   inspection, cancellation, and approval routes and `apps/api/lib/chat.ts`. Added
   `apps/api/app/api/runs/[runId]/chunks/route.ts` for authenticated recovery of saved progress.
   The old coordinator moved into historical test fixtures; its API runtime file and publisher
   were removed.
6. **Recover the browser from durable events.** Updated `packages/contracts/src/index.ts`,
   `apps/web/src/chat.tsx`, and `apps/web/src/lib/agent-runs.ts`. Sequenced, bounded visible events
   are saved with outbox records. Reloads rebuild draft text; the authoritative final message
   replaces the draft without duplicating it. Hidden reasoning and raw SDK messages are excluded.
7. **Exercise failure and restart cases.** Added
   `apps/worker/test/agent-jobs.integration.test.ts`,
   `apps/web/e2e/durable-workers.spec.ts`, `scripts/verify-agent-worker.mjs`, and
   `scripts/verify-agent-browser.mjs`. Updated earlier chat, session, budget, and approval fixtures
   to claim workers explicitly. Ran builds, checks, database acceptance tests, process restart
   checks, and installed Chrome acceptance. Recorded decisions in
   [ADR 0020](docs/decisions/0020-durable-agent-jobs.md) and teaching notes in lesson 27.

## Results achieved

The recorded local/mock acceptance demonstrated these behaviors:

- A request returns HTTP **202 Accepted** with a queued run. It stays queued until an agent worker
  is available. Another active turn in the same conversation is rejected with HTTP 409.
- Restarting the API preserves an answer running in a separate worker and an approval waiting
  for the user. Closing or reloading the browser does not cancel the job.
- Workers renew a 30-second ownership lease every five seconds. Competing workers cannot claim
  the same attempt; stale workers cannot save results, session bindings, progress, or mutations.
- Explicit cancellation is saved in PostgreSQL. Cancelling a queued job before model invocation
  records `not_started` usage and zero cost.
- A lost claim can retry only if execution never began and no progress or proposals exist, with
  at most three claims. Begun work becomes visibly interrupted after lease expiry and recovery.
  The user must deliberately send a new request. Unused approvals are invalidated; already
  consumed mutation receipts remain saved. Uncertain spending retains the budget reservation.
- Saved answer chunks recover after reload or missing Redis events. Completion saves one final
  assistant message together with terminal events and budget reconciliation.

### Recorded verification results

These are results from the completed milestone, **not new application test runs for this README**.

| Check | Recorded result |
| --- | --- |
| Offline dependency installation | 348 packages; zero vulnerabilities reported |
| Fresh database migration | 16 migrations applied; final deploy had no pending migrations |
| `npm run build`, `typecheck`, `lint`, `test` | Passed; default suite: 285 passed, 110 opt-in skipped |
| `node scripts/check-browser-boundary.mjs` | Passed |
| Durable jobs PostgreSQL suite | 8/8 passed |
| Budget / approval / session-analysis / streamed-chat acceptance | 13/13, 13/13, 10/10, 8/8 passed |
| `node scripts/verify-agent-worker.mjs` | Passed with real Next.js HTTP and separate mock workers |
| `node scripts/verify-agent-browser.mjs` | Installed Chrome: 1/1 passed, 34.9 seconds |
| Prisma schema drift inspection | Exit 2: older foreign-key/index differences remain |

One jobs test run under simultaneous build load exceeded its five-second timeout; the final
isolated run passed. Earlier fixture assumptions, cleanup races, and event-frame assumptions were
corrected. An initial HTTP 503 did not recur in the final isolated verifier. Existing Vite
directive/large-bundle warnings and Next.js instrumentation Edge warnings remained non-fatal.
Full details are in [the milestone report](docs/project-state.md#latest-milestone-report---27).

## How to run the local demo

### Prerequisites

- Node.js 24.21.0 and npm 11.19.0, matching the project baseline in [versions](docs/versions.md).
- Docker with a running Linux container engine for PostgreSQL and Redis.
- Installed Google Chrome for the browser verifier.
- Run commands from the repository root. Examples below use PowerShell.
- Mock mode requires no Anthropic or market-data credentials.

### Install and configure

```powershell
node --version
npm --version
npm ci --ignore-scripts
npm run infra:start
npm run infra:status

# Only copy if the file does not already exist; preserve your current configuration.
if (!(Test-Path apps/api/.env.local)) {
    Copy-Item apps/api/.env.example apps/api/.env.local
}
```

Edit `apps/api/.env.local` so the following values are set. These database credentials and secret
are public local-demo values. Use the same file for the API and workers so model, mode, budgets,
and connection settings agree.

```dotenv
DATA_MODE=mock
AGENT_MODE=mock
DATABASE_URL=postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_pilot
REDIS_URL=redis://127.0.0.1:6379
DEMO_AUTH_ENABLED=true
AUTH_BASE_URL=http://127.0.0.1:5173
AUTH_SECRET=local-demo-only-change-this-secret-27
```

Prisma and seed commands read the shell environment. Set it explicitly before building,
migrating, and seeding the local demo database. The seed creates repeatable demo data.

```powershell
$env:DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_pilot'
$env:NODE_ENV='development'
$env:ALLOW_DEMO_SEED='true'
npm run build
npm run migrate:deploy --workspace=@portfolio-pilot/db
npm run seed:demo --workspace=@portfolio-pilot/db
npm run dev
```

`npm run dev` starts the API on port 3001 and React/Vite on port 5173. It does **not** start workers.
Open two more terminals in the repository root:

```powershell
# Terminal 2: execute chat jobs.
$env:WORKER_ROLE='agent'
node --env-file=apps/api/.env.local apps/worker/dist/index.js
```

```powershell
# Terminal 3: publish committed events to Redis for SSE delivery.
$env:WORKER_ROLE='outbox'
node --env-file=apps/api/.env.local apps/worker/dist/index.js
```

### Try the expected behavior

1. Open `http://127.0.0.1:5173/assistant`, sign in as demo user Alice, and create a conversation.
2. Send “Explain my holdings.” Expect a labeled mock answer and one final message.
3. Refresh during an answer. Expect saved partial text to recover, then the final message to
   replace it. The automated browser test verified this case.
4. Stop the agent worker, submit a new question, and check that it remains queued. Cancel it or
   restart the worker to let it execute.
5. Ask to add a known stock absent from your watchlist. Expect an approval card before a change
   is applied. Cancellation invalidates unused permission.

For a manual API-only restart, run API and web dev scripts in separate terminals instead of
`npm run dev`, then restart only the API terminal. Keep both worker processes running.

```powershell
# Separate terminals, in place of the combined dev command:
npm run dev --workspace=@portfolio-pilot/api
npm run dev --workspace=@portfolio-pilot/web
```

Without the outbox worker, database polling can recover answers, but SSE delivery is delayed.
Mock answers may finish too quickly for manual restart experiments; use the verifier below for
repeatable scenarios. Stop processes with Ctrl+C; `npm run infra:stop` preserves database volumes.

## How to verify the implementation

Run local code checks after installation and build:

```powershell
npm run typecheck
npm run lint
npm run test
npm run check:browser-boundary
```

The default suite skips tests that need explicit database variables. A default pass alone does
not verify durable recovery. `lint` currently runs TypeScript checking.

### Repeat the database, process, and browser acceptance checks

Use a **disposable** database named exactly `portfolio_m27_verify` on `127.0.0.1`. The completed
milestone used port 5546 in an existing verification container. The commands below instead create
that isolated database in the Compose PostgreSQL service on port 5432; this setup recipe was
checked against the scripts but was not executed during this README update. Run `CREATE DATABASE`
only once. These tests modify fixture data; keep them separate from the application database.

Stop existing demo servers and workers first. Redis must be available at `127.0.0.1:6379`; ports
5301, 3001, and 5173 must be free. Run the verifiers sequentially, without other demo-user suites.

```powershell
npm run infra:start
docker compose exec -T postgres psql -U portfolio_local -d postgres -c 'CREATE DATABASE portfolio_m27_verify;'
$env:DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_m27_verify'
$env:NODE_ENV='development'
$env:ALLOW_DEMO_SEED='true'
npm run build
npm run migrate:deploy --workspace=@portfolio-pilot/db
npm run seed:demo --workspace=@portfolio-pilot/db
$env:AGENT_JOBS_TEST_DATABASE_URL=$env:DATABASE_URL
npm run test --workspace=@portfolio-pilot/worker -- test/agent-jobs.integration.test.ts
node scripts/verify-agent-worker.mjs
node scripts/verify-agent-browser.mjs
```

The process verifier starts the compiled API and separate mock agent/outbox workers, restarts the
API during an answer and an approval, tests cancellation, kills a worker, and checks authenticated
event recovery and one final message. It advances only its fixture's lease to test crash recovery
quickly. The browser verifier starts API/Vite and runs the Chrome scenario. Both stop their own
processes; database fixtures can remain.

The process verifier's successful output includes:

```text
PASS: actual Next.js API restart preserves worker execution, sequenced chunk recovery, approval after API restart, explicit cancellation, crashed-worker interruption, authenticated SSE, and exactly-once completion.
```

## Limitations and unfinished work

- **Live Claude is unverified.** No credentialed model call was included in the milestone's
  acceptance. To perform that check, configure `ANTHROPIC_API_KEY`, an account-supported
  `AGENT_MODEL_ID`, and an isolated absolute `AGENT_WORKSPACE_DIR` outside the repository in the
  shared server environment. Set `AGENT_MODE=claude`, restart API/workers, then repeat a focused
  answer, approval, and cancellation using the worker command above. Live calls can incur cost.
- **SDK session artifacts remain host-local.** Another host reseeds from saved history rather
  than promising portable continuation. Persisting session artifacts is milestone 28.
- **Recovery needs an available worker and PostgreSQL.** A busy worker can delay recovery after
  the 30-second lease expires. A lost lease cannot instantly stop an external provider, but
  fencing prevents stale saved results and application mutations.
- **Older Prisma drift remains.** A schema drift check compares migration-created tables with
  the Prisma model. Older ApprovalRequest/NewsRead foreign-key update actions and a Recommendation
  evidence GIN index differ; no new job/chunk drift remained. To inspect using the disposable URL:

  ```powershell
  npx --no-install prisma migrate diff --config packages/db/prisma.config.ts --from-config-datasource --to-schema packages/db/prisma/schema.prisma --exit-code
  ```

  The recorded result was exit 2, not a passing check. Repair of those earlier differences remains.
- Fleet-wide limits, monitoring, retention, and administrative recovery remain milestone 30.
  This activity does not deploy to Azure, provision cloud resources, or execute broker trades.
- This README update changed documentation only. Application checks were not rerun; its static
  validation is recorded separately in project state and lesson 27.

## Further reading

- [Project contract](AGENTS.md): architecture and implementation rules.
- [Project plan](docs/project-plan.md): milestone sequence and acceptance criteria.
- [Project state](docs/project-state.md): actual check results, fixes, and remaining blockers.
- [Lesson 27](docs/lessons/27-durable-agent-worker.md): teaching notes and recovery exercise.
- [ADR 0020](docs/decisions/0020-durable-agent-jobs.md): ownership, retry, and fencing decisions.
