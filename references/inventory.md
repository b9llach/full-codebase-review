# Phase 1: Inventory

Before any agent runs, map the codebase. A reviewer who doesn't know what they're looking at will miss things and flag phantoms.

## Step 1 — Structural walk

Run this from the repo root:

```bash
# Directory structure, depth 3, no heavy dirs
tree -L 3 -I 'node_modules|.next|dist|build|.git|target|__pycache__|.venv|venv|.pytest_cache|coverage|.turbo'

# Line counts per language (install cloc if missing)
cloc . --exclude-dir=node_modules,.next,dist,build,target,.venv,venv,coverage

# File count per directory
find . -type f -not -path '*/node_modules/*' -not -path '*/.next/*' -not -path '*/dist/*' -not -path '*/.git/*' | awk -F/ '{print $2}' | sort | uniq -c | sort -rn | head -20
```

Capture:
- Total line count by language
- Largest directories
- Framework fingerprints (Next.js app/ dir, pages/, Django apps/, Rails app/, etc.)

## Step 2 — Manifest files

Read and summarize:

- `package.json` — scripts, dependencies, dev dependencies, private flag, workspaces
- `package-lock.json` / `pnpm-lock.yaml` / `yarn.lock` — note lockfile presence (pin discipline)
- `tsconfig.json` — `strict`, `noImplicitAny`, `strictNullChecks`, path aliases
- `next.config.*` / `vite.config.*` / `webpack.config.*` — build config, env exposure
- `requirements.txt` / `pyproject.toml` / `Pipfile` — Python deps
- `Cargo.toml`, `go.mod`, `Gemfile`, `composer.json`, `pom.xml`, `build.gradle` — other ecosystems
- `docker-compose.yml`, `Dockerfile*` — runtime topology
- `.env.example` — shape of required secrets (never the real `.env`)
- `prisma/schema.prisma` / `drizzle.config.*` / `migrations/` — data model source of truth
- `.github/workflows/*.yml` / `.gitlab-ci.yml` — CI discipline

## Step 3 — Author framing

Read anything the authors wrote about their own code:

- `README.md` at root
- `ARCHITECTURE.md`, `DESIGN.md`, `CONTRIBUTING.md`
- `CLAUDE.md` / `.cursorrules` / `.github/copilot-instructions.md` — AI-coding instructions often reveal intent
- `docs/` directory if present
- Top-level comments in entry point files

This is often the best source of intent. A review that contradicts stated intent should note the contradiction, not assume the code is wrong.

## Step 4 — Entry points and critical paths

Identify:

**Entry points** — where execution begins:
- Server: `server.js`, `index.ts`, `main.py`, `app.py`, `cmd/*/main.go`
- Next.js: `app/` route handlers, `pages/api/*`, `middleware.ts`
- CLI: bin/ scripts, package.json `bin` field
- Worker: queue consumers, cron, scheduled jobs
- Frontend: root `layout.tsx` / `_app.tsx` / `main.tsx`

**Critical paths** — files where bugs are expensive:
- Auth: look for `auth`, `session`, `jwt`, `login`, `signup`, `password`, `oauth`, `saml`
- Money: `stripe`, `payment`, `charge`, `refund`, `payout`, `invoice`, `billing`, `subscription`, `wallet`, `balance`, `escrow`, `wager`, `bet`
- Data mutation: `insert`, `update`, `delete`, `migrate`, `sync`
- External integration: `webhook`, `callback`, `ingest`, `webhookHandler`
- Admin: `admin`, `internal`, `dashboard`, `management`
- Crypto: `crypto`, `encrypt`, `sign`, `verify`, `hash`, `hmac`

Build a list. You'll feed this to Phase 2 (triage).

## Step 5 — Domain detection

What does this software *do*? Look for vocabulary clues across the repo:

| Signal | Likely domain |
|--------|---------------|
| `retirement`, `401k`, `ira`, `roth`, `withdrawal`, `nest_egg`, `fire` | Retirement planning → fintech pack |
| `mortgage`, `amortization`, `principal`, `apr` | Lending → fintech pack |
| `tax`, `bracket`, `deduction`, `w2`, `1099` | Tax software → fintech pack |
| `odds`, `wager`, `bet`, `stake`, `payout`, `line`, `spread`, `parlay` | Wagering → gaming pack |
| `tenant`, `organization`, `workspace`, `seat`, row-level `tenant_id` | Multi-tenant SaaS → saas-multitenant pack |
| `telematics`, `gps`, `eld`, `geofence`, `fleet`, `vehicle`, time-series tables | IoT/telemetry → iot-telemetry pack |
| `seller`, `buyer`, `listing`, `vendor`, `commission`, split payouts | Marketplace → marketplace pack |
| `patient`, `phi`, `hipaa`, `medical_record`, `diagnosis` | Healthcare → healthcare pack |
| `course`, `student`, `enrollment`, `grade`, `assignment` | EdTech (no dedicated pack; use generic) |
| `driver`, `ride`, `eta`, `dispatch` | Logistics (use iot-telemetry + marketplace) |

If multiple domains apply, load multiple packs. A telemetry product that also has a marketplace component, for example, would load both `iot-telemetry` and `marketplace`.

## Step 6 — Write the outputs

**`inventory.md`:**

```markdown
# Inventory

## Repo shape
- Total lines: X (Y% TypeScript, Z% Python, ...)
- Top directories: `app/` (N files), `lib/` (N files), ...
- Framework: Next.js 15 (App Router) + PostgreSQL + Drizzle

## Manifest summary
### Dependencies (production)
- Next.js 15.0.2
- Stripe 17.x
- Drizzle ORM 0.36.x
- ...

### Scripts
- dev, build, test, lint, db:migrate

### Build config
- TypeScript strict: ✓
- Strict null checks: ✓

## Author framing
[Summary of README, architecture docs, CLAUDE.md]

## Entry points
- `app/page.tsx` — root page
- `app/api/stripe/webhook/route.ts` — payment webhook
- `app/api/auth/[...nextauth]/route.ts` — auth callbacks
- `lib/cron/daily-report.ts` — scheduled job

## Critical paths detected
[List of files matching the money/auth/mutation/external patterns]

## Domain signals
- Detected: retirement planning (fintech pack)
- Evidence: `lib/retirement/monte-carlo.ts`, `lib/retirement/tax-brackets.ts`
```

**`stack.md`:**

```markdown
# Detected stack

- Runtime: Node.js 20
- Framework: Next.js 15 App Router
- Language: TypeScript (strict mode)
- Database: PostgreSQL via Drizzle ORM
- Auth: NextAuth v5
- Payments: Stripe
- Deployment: Vercel + Railway workers

## Agents to run
Standard set, plus:
- Domain pack: fintech.md
```

With these written, proceed to Phase 2.
