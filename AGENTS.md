# AGENTS.md

## Repo state

**Pre-implementation.** No source code exists yet. The repo currently contains only:
- `prd.md` — product requirements and architecture (modular monolith)
- `backend/TASKS.md`, `frontend/TASKS.md` — detailed build checklists; treat as the implementation spec
- `.github/workflows/opencode*.yml` — OpenCode CI hooks (see below)

The empty `backend/{cmd,internal,pkg,db}` and `frontend/` trees are intentional scaffolding matching the planned module layout. Do **not** run `go build`, `go test`, `ng build`, lint, or format — there is nothing to build yet. Do not assume default toolchain commands; check `prd.md` / `TASKS.md` for the intended stack and versions first.

## Before implementing anything

Read `prd.md` (architecture, module specs, schema overview) and the relevant `TASKS.md` section. The TASKS files are checklists of exactly what to build, with file paths, DTOs, SQL queries, and conventions pre-decided. Following them is the project convention; inventing a different structure or stack is almost always wrong.

## Before any commit (mandatory)

Read [`CONTRIBUTING.md`](./CONTRIBUTING.md) and complete the agent pre-commit checklist in §2 before staging or writing a commit. It encodes the branching model, Conventional Commit scopes, verification commands, and the non-obvious backend/frontend conventions summarized below.

## Stack (pinned in `prd.md`)

- Backend: Go 1.25, Gin, `pgx/v5`, sqlc (engine `postgresql`, `sql_package: pgx/v5`), Atlas migrations, Redis, MinIO/S3, River job queue. Postgres 16 with `pgcrypto` + `ltree`.
- Frontend: Angular 22 (standalone components, signals, `inject()`, `OnPush`, native `@if/@for/@switch`), SpartanUI, TailwindCSS v4, NgRx Signal Store.
- Payments are **simulated in-process in v1** — no Stripe/Paystack SDK. Real gateway is v2.

## Non-obvious conventions (from `backend/TASKS.md`)

- **Module boundaries are enforced.** A module's service may depend on another module's *service interface* only — never on another module's repository. `internal/platform` is infrastructure-only (no domain types) and depends on nothing internal.
- `gin.New()` (manual middleware), not `gin.Default()`.
- Errors are RFC 7807 Problem Details via `pkg/apierr`; `Content-Type: application/problem+json`.
- Money is integer cents + ISO 4217 currency (`pkg/money`). Never float arithmetic.
- Pagination is cursor-based (keyset) via `pkg/pagination`; cursor is base64 of `(id, created_at)`.
- JWT is RS256, access 15m, refresh 7d rotating in Redis; bcrypt cost 12; refresh token in `HttpOnly; Secure; SameSite=Strict` cookie.
- sqlc per-module output: `db/queries/<module>.sql` → `internal/<module>/repository/`. Enable `emit_json_tags`, `emit_db_tags`; `output_db_value_as_pointers: false`.
- Atlas: `src = file://db/schema`, `dev = docker://postgres/16/dev`, migrations in `db/migrations`. Never edit applied migrations.

## Non-obvious conventions (from `frontend/TASKS.md`)

- TailwindCSS v4 uses **CSS-based configuration** (`@import "tailwindcss";` + `@theme`). Do not create `tailwind.config.js`.
- Access token in memory only (signal); refresh token is server-managed HttpOnly cookie. Never persist access token to `localStorage`.
- All feature routes are lazy-loaded (`loadComponent` / `loadChildren`).
- Tests: Jest + Angular Testing Library for unit/component; Playwright for E2E.

## OpenCode GitHub Actions

- `.github/workflows/opencode.yml` — triggered by PR/issue comments starting with `/oc` or `/opencode`.
- `opencode-review.yml` — automatic PR review on open/synchronize/reopen/ready_for_review.
- `opencode-triage.yml` — auto-triages new issues from accounts ≥30 days old.
- `opencode-scheduled.yml` — weekly Monday 09:00 UTC TODO sweep.
- All use `opencode/deepseek-v4-flash-free` and require `OPENCODE_API_KEY` secret.

## When in doubt

Prefer the spec. If `prd.md` and the `TASKS.md` files disagree with your intuition, the spec wins. If two spec files disagree, `prd.md` is the architecture authority and the TASKS file is the build detail.
