# Contributing to NimCart

> **Read this before opening a PR or making any commit — humans and AI agents alike.**
> If you are an automated agent, also read [`AGENTS.md`](./AGENTS.md) first; it contains repo-state rules that override your defaults.

NimCart is a modular-monolith eCommerce platform. The source of truth is not this file — it is, in order of authority:

1. [`prd.md`](./prd.md) — architecture and module specs (the authority on disputes).
2. [`backend/TASKS.md`](./backend/TASKS.md), [`frontend/TASKS.md`](./frontend/TASKS.md) — build checklists with file paths, DTOs, SQL, and conventions pre-decided.
3. This document — workflow, commit, and review conventions.
4. Your intuition — last resort. When in doubt, prefer the spec.

---

## 1. Repo state (read first)

The repository is **pre-implementation**. The empty `backend/{cmd,internal,pkg,db}` and `frontend/` trees are intentional scaffolding matching the planned module layout. Until the relevant `TASKS.md` checkbox is implemented:

- Do **not** run `go build`, `go test`, `ng build`, lint, or format against code that does not exist yet.
- Do **not** invent a different stack, structure, or naming than what `prd.md` / `TASKS.md` specify.
- Do **not** assume default toolchain commands. Check the pinned versions below first.

Pinned stack (from `prd.md`):

- **Backend:** Go 1.25, Gin, `pgx/v5`, sqlc (engine `postgresql`, `sql_package: pgx/v5`), Atlas migrations, Redis, MinIO/S3, River job queue. PostgreSQL 16 with `pgcrypto` + `ltree`.
- **Frontend:** Angular 22 (standalone components, signals, `inject()`, `OnPush`, native `@if/@for/@switch`), SpartanUI, TailwindCSS v4, NgRx Signal Store.
- **Payments:** simulated in-process in v1. No Stripe/Paystack SDK. Real gateway is v2.

---

## 2. Agent pre-commit checklist (mandatory)

Before staging or writing any commit, an AI agent **must** verify, at minimum:

- [ ] I have read `AGENTS.md`, `prd.md`, and the relevant `TASKS.md` section for the area I am touching.
- [ ] The change follows the file paths, DTOs, SQL, and conventions already specified in `TASKS.md` (no invented structure).
- [ ] I am respecting module boundaries (see §6). No cross-module repository imports.
- [ ] No secrets, API keys, private keys, or `.env` files are being committed.
- [ ] No money is handled with floating-point arithmetic (see §7).
- [ ] No applied Atlas migration is being edited (see §8).
- [ ] No access token is persisted to `localStorage` and no `tailwind.config.js` is being created (see §9).
- [ ] I am using `gin.New()` (manual middleware), never `gin.Default()`.
- [ ] Errors are returned as RFC 7807 Problem Details via `pkg/apierr` (`application/problem+json`).
- [ ] I have **not** run `go build`/`go test`/`ng build`/lint against non-existent code. Once the target module exists, I ran the verification commands in §5 and they pass.
- [ ] The commit message follows Conventional Commits with the correct module scope (see §4).
- [ ] I am only committing files I intended to change — I checked `git status` and `git diff` first.

If any box cannot be checked honestly, do not commit. Either fix it or stop and ask.

---

## 3. Branching & pull-request workflow

Branching model (from `prd.md` §11.5):

| Branch | Purpose |
|---|---|
| `main` | Production-ready. Protected. Requires PR + 1 approval. |
| `develop` | Integration branch. All feature branches merge here first. |
| `feature/<ticket-id>-short-description` | Standard feature work. |
| `hotfix/<ticket-id>-description` | Branched from `main`; merged back to both `main` and `develop`. |

Rules:

- Never push directly to `main` or `develop`. Open a PR.
- Branch names are lowercase, kebab-case, and include the ticket ID when one exists: `feature/M2-catalog-listing`, `hotfix/M3-cart-merge-race`.
- Keep PRs scoped to one milestone/task area so they map cleanly to a `TASKS.md` checklist item.
- PRs that touch DB schema require an Atlas migration diff (`atlas migrate diff`) and must pass `atlas migrate lint` — see §8.

### OpenCode CI hooks

The repo ships four OpenCode GitHub Actions (see `AGENTS.md` for the full list):

- `/oc` or `/opencode` in a PR/issue comment triggers an OpenCode run.
- `opencode-review.yml` auto-reviews PRs on open/synchronize/reopen/ready_for_review.
- `opencode-triage.yml` auto-triages new issues from accounts ≥ 30 days old.
- `opencode-scheduled.yml` runs a weekly TODO sweep (Mon 09:00 UTC).

All use `opencode/deepseek-v4-flash-free` and require the `OPENCODE_API_KEY` secret. Nothing to configure locally.

---

## 4. Commit messages

Use **Conventional Commits**. The existing history follows this style (`docs:`, `ci(opencode):`).

```
<type>(<scope>): <imperative summary in lowercase, ≤ 72 chars>

<optional body, wrapped at 72 cols, explaining why not what>

<optional footer: BREAKING CHANGE: ..., Closes #123>
```

**Types:** `feat`, `fix`, `docs`, `refactor`, `perf`, `test`, `ci`, `chore`, `build`.

**Scopes** (use the module you touched; omit scope only for cross-cutting changes):

- Backend modules: `auth`, `catalog`, `cart`, `order`, `payment`, `inventory`, `user`, `review`, `analytics`, `notification`, `coupon`.
- Backend infra: `platform`, `db`, `pkg`, `cmd`, `migrations`, `sqlc`, `config`.
- Frontend: `frontend`, `frontend:<feature>` (e.g. `frontend:cart`, `frontend:checkout`, `frontend:admin`).
- Tooling/CI: `ci`, `deps`, `docker`.

Examples:

```
feat(catalog): add cursor-paginated product listing
fix(order): release hard reservation on cancellation
docs(prd): clarify simulated payment provider scope
ci(opencode): add review, triage, scheduled jobs
refactor(pkg/money): remove float arithmetic from Percent
test(auth): cover refresh-token rotation and blacklist
```

- One logical change per commit. Do not bundle unrelated changes.
- Reference the ticket/milestone in the body or footer, not the summary.
- `BREAKING CHANGE:` footer for anything that forces downstream consumers to adapt; bump per `prd.md` versioning.

---

## 5. Verification before committing

> Until the target code exists, skip build/test/lint (see §1). Once it does, run the commands below from `backend/` or `frontend/` as appropriate.

**Backend** (`backend/`):

```bash
golangci-lint run                    # govet, errcheck, staticcheck, gosec, depguard, revive, goimports
go build ./cmd/server                # CGO_ENABLED=0
go test ./... -race -coverprofile    # ≥ 80% line coverage per module (prd §9)
sqlc generate                        # regenerate after any db/queries/*.sql change
atlas migrate lint --env dev         # block destructive migrations
```

**Frontend** (`frontend/`):

```bash
npm run lint                         # @angular-eslint + prettier
npm run test -- --coverage           # Jest + Angular Testing Library
npm run e2e                          # Playwright critical flows
ng build --configuration production  # bundle budgets must pass
```

Do not commit if any of these fail. Fix the root cause; never silence a lint or test failure to land a change.

---

## 6. Module boundaries (backend) — enforced

From `prd.md` §5.3 and `AGENTS.md`:

- A module exposes only types in its top-level package (e.g. `catalog.Product`, `catalog.Service`).
- A module's service may depend on another module's **service interface** only — **never** on another module's repository or DB types.
- `internal/platform` is infrastructure-only (no domain types) and depends on nothing internal.
- Cross-module state changes go through the internal event bus (`platform/eventbus`), not direct calls.
- Violations are caught at CI time by `depguard` rules in `golangci-lint`. They will also be caught in review.

If you need data owned by another module, call its service, do not query its tables.

---

## 7. Backend conventions (non-obvious)

- **HTTP:** `gin.New()` with manual middleware in the order specified in `backend/TASKS.md` → `cmd/server`. Never `gin.Default()`.
- **Errors:** RFC 7807 Problem Details via `pkg/apierr`. `Content-Type: application/problem+json`. Never leak internal details in error messages (middleware returns generic 500 for unknown errors).
- **Money:** integer cents + ISO 4217 currency via `pkg/money`. **Never** float arithmetic. Store as `BIGINT` (cents) in Postgres.
- **Pagination:** cursor-based (keyset) via `pkg/pagination`. Cursor is base64 of `(id, created_at)`. Default limit 20, max 100. Fetch `limit + 1` to compute `has_more`.
- **Auth:** JWT RS256, access 15m, refresh 7d rotating in Redis. bcrypt cost 12. Refresh token in `HttpOnly; Secure; SameSite=Strict` cookie.
- **sqlc:** per-module output `db/queries/<module>.sql` → `internal/<module>/repository/`. Enable `emit_json_tags`, `emit_db_tags`; `output_db_value_as_pointers: false`. Use `-- name: Query :one|:many|:exec`, `sqlc.narg()`, `sqlc.embed()`.
- **API surface:** all endpoints under `/api/v1/`. `snake_case` JSON fields. Mutating order/payment endpoints require `X-Idempotency-Key`.
- **Validation:** `go-playground/validator/v10` struct tags on every request DTO. Handlers stay thin: bind → validate → call service → respond.
- **Logging:** zap structured JSON with `trace_id`, `user_id`, latency, status. Never log passwords, tokens, or PII.

---

## 8. Database & migrations

- **Atlas:** `src = file://db/schema`, `dev = docker://postgres/16/dev`, migrations in `db/migrations`. Never edit an applied migration.
- Workflow: edit `db/schema/*.hcl` → `atlas migrate diff --env dev <name>` → review generated SQL → `atlas migrate apply --env dev`.
- All tables: UUID PKs via `gen_random_uuid()` (requires `pgcrypto`), `created_at TIMESTAMPTZ DEFAULT now()`, `updated_at` managed by trigger, soft deletes via `deleted_at`/`archived_at`.
- Categories use `ltree` (`path`, `depth`). Enable the extension.
- Money columns are `BIGINT` (cents), never `NUMERIC`/float.
- Indexes per `prd.md` §7.2: GIN on `to_tsvector` for full-text search, GIN on `tags`, BTREE on `(category_id, status)`, BTREE on `(user_id, created_at DESC)` for orders, unique partial index on `coupons.code WHERE deleted_at IS NULL`.
- Destructive migrations (drop column/table, type changes with data loss) require explicit approval and will be blocked by `atlas migrate lint`.

---

## 9. Frontend conventions (non-obvious)

- **Angular 22:** standalone components only (no NgModules), `inject()` over constructor injection, `input()`/`output()` over `@Input()`/`@Output()`, `OnPush` everywhere, native `@if/@for/@switch` (not `*ngIf`/`*ngFor`/`*ngSwitch`).
- **State:** signals (`signal()`, `computed()`, `effect()`) + NgRx Signal Store. No `NgRx` classic stores, no `BehaviorSubject`-based services for feature state.
- **TailwindCSS v4:** CSS-based configuration only — `@import "tailwindcss";` + `@theme` in `styles.css`. **Do not create `tailwind.config.js`.**
- **Auth tokens:** access token in memory only (signal). Refresh token is server-managed `HttpOnly` cookie. **Never persist the access token to `localStorage`/`sessionStorage`.**
- **Routing:** all feature routes are lazy-loaded via `loadComponent` / `loadChildren`. Admin routes behind `AdminGuard`; auth routes behind `AuthGuard`; login/register behind `GuestGuard`.
- **Forms:** Reactive Forms only (typed, composable validators). No template-driven forms.
- **Images:** `NgOptimizedImage` for all static images. Product images upload via presigned S3 URLs (client uploads directly).
- **No** `ngClass`/`ngStyle` (use `class`/`style` bindings), **no** `@HostBinding`/`@HostListener` (use the `host` object).
- **Accessibility:** WCAG 2.1 AA. SpartanUI primitives are ARIA-compliant; verify with axe-core. No emoji-only labels or icon-only buttons without `aria-label`.

---

## 10. Testing expectations

- **Coverage gate:** ≥ 80% line coverage per module (backend) — enforced in CI.
- **Backend:** `testify` + `gomock` for unit tests, `httptest` for handler integration, real Postgres (testcontainers or CI DB) for repository tests. Table-driven tests preferred.
- **Frontend:** Jest + Angular Testing Library for unit/component (user-centric queries). Playwright for E2E critical flows: auth, browse → cart → checkout → order confirmation, admin product approval, search.
- **Performance:** Lighthouse ≥ 85 (Performance/Accessibility/Best Practices/SEO); FCP < 1.8s; TTI < 2.5s on 4G; CLS < 0.1.
- Never skip a test to land a change. If a test is genuinely wrong, replace it with one that is correct and explain why in the PR description.

---

## 11. Security checklist (non-negotiable)

- No secrets, keys, or credentials in source. Use `.env.example` for placeholders only.
- Passwords: bcrypt cost 12, never logged or returned in responses.
- Refresh tokens: `HttpOnly; Secure; SameSite=Strict` cookie only.
- Access tokens: in-memory signal on the frontend, never `localStorage`.
- RS256 private key: loaded from Vault/env at startup, never committed.
- Row-level ownership checks enforced in the service layer (customers only touch their own orders/addresses/reviews), not just the router.
- User enumeration prevention: `forgot-password` returns the same response regardless of email existence.
- Uploads: MIME-type validation, ≤ 10 MB cap.
- No real payment webhooks in v1; simulation endpoints disabled in production.

---

## 12. PR description template

```markdown
## What
<1–2 sentences, linked to TASKS.md checklist item or ticket>

## Why
<motivation; what problem this solves>

## How
<bulleted summary of the approach>

## Verification
- [ ] `golangci-lint run` / `npm run lint` passes
- [ ] `go test ./... -race` / `npm run test` passes (coverage ≥ 80%)
- [ ] `ng build --configuration production` / `go build ./cmd/server` passes
- [ ] `atlas migrate lint --env dev` passes (if schema changed)
- [ ] `sqlc generate` is up to date (if queries changed)

## Schema changes
<none, or list of new migrations with filenames>

## Breaking changes
<none, or `BREAKING CHANGE:` summary and migration notes>
```

---

## 13. Spec precedence (final word)

If `prd.md` and `TASKS.md` disagree with your intuition, **the spec wins.** If `prd.md` and a `TASKS.md` file disagree with each other, `prd.md` is the architecture authority and the `TASKS.md` file is the build detail — surface the conflict in your PR rather than silently picking one.

---

*NimCart CONTRIBUTING v1 — maintained alongside `prd.md` and `AGENTS.md`.*
