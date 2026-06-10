# NimCart Backend — Complete TASKS.md

## M0 — Foundation & Project Scaffolding

### Go Module & Tooling
- [ ] Initialize Go module (`go mod init github.com/<org>/nimcart`)
- [ ] Pin Go version to 1.25 in `go.mod`
- [ ] Install and configure `golangci-lint` with `depguard` rules to enforce module boundary imports
- [ ] Create `.golangci.yml` with linters: `govet`, `errcheck`, `staticcheck`, `gosec`, `depguard`, `revive`, `goimports`
- [ ] Add `depguard` rules: forbid `internal/auth` from importing `internal/catalog/internal/...`, etc.
- [ ] Create `Makefile` with targets: `lint`, `test`, `build`, `generate`, `migrate-up`, `migrate-diff`, `serve`
- [ ] Create `.air.toml` for hot-reload during development
- [ ] Create `.env.example` with all required environment variables

### Docker Compose
- [ ] Create `docker-compose.yml` with services: `postgres:16`, `redis:7`, `minio`, `pgbouncer`
- [ ] Configure PostgreSQL with `pgcrypto` and `ltree` extensions enabled
- [ ] Configure PgBouncer in transaction mode with `pool_mode = transaction`
- [ ] Add health checks for all services
- [ ] Create `docker-compose.override.yml` for local dev port mappings
- [ ] Add MinIO init container to create default bucket (`nimcart-assets`)

### sqlc Configuration
- [ ] Create `sqlc.yaml` at project root
- [ ] Configure engine: `postgresql`, sql_package: `pgx/v5`
- [ ] Set query paths: `db/queries/**/*.sql`
- [ ] Set schema paths: `db/migrations/*.sql` (Atlas output)
- [ ] Configure per-module output directories:
  - `internal/auth/repository/` for `db/queries/auth.sql`
  - `internal/catalog/repository/` for `db/queries/catalog.sql`
  - `internal/cart/repository/` for `db/queries/cart.sql`
  - `internal/order/repository/` for `db/queries/order.sql`
  - `internal/payment/repository/` for `db/queries/payment.sql`
  - `internal/inventory/repository/` for `db/queries/inventory.sql`
  - `internal/user/repository/` for `db/queries/user.sql`
  - `internal/review/repository/` for `db/queries/review.sql`
  - `internal/analytics/repository/` for `db/queries/analytics.sql`
- [ ] Enable `emit_json_tags: true`, `emit_db_tags: true`
- [ ] Set `output_db_value_as_pointers: false` (prefer value types)

### Atlas Configuration
- [ ] Create `atlas.hcl` with `env "dev"` and `env "prod"` blocks
- [ ] Configure `dev` env: `src = "file://db/schema"`, `dev = "docker://postgres/16/dev"`
- [ ] Configure migration directory: `file://db/migrations`
- [ ] Set `qualifier = "schema"` for PostgreSQL
- [ ] Create initial Atlas schema files in `db/schema/` for all tables

### CI/CD Pipeline (GitHub Actions)
- [ ] Create `.github/workflows/ci.yml`
- [ ] Add `lint` job: `golangci-lint run` (Go) + `eslint` + `prettier` (Angular)
- [ ] Add `test` job: `go test ./... -race -coverprofile` with 80% coverage gate
- [ ] Add `build` job: `go build ./cmd/server` + `ng build --configuration production`
- [ ] Add `migration-lint` job: `atlas migrate lint --env dev` to block destructive changes
- [ ] Add `docker-build` job: multi-arch build (main branch only)
- [ ] Add `deploy-staging` job: auto on merge to `main`
- [ ] Add `deploy-prod` job: manual approval gate

### Documentation
- [ ] Create `README.md` with project overview, setup instructions, architecture diagram
- [ ] Create `CONTRIBUTING.md` with branching strategy, PR checklist, code review guidelines
- [ ] Create `docs/architecture.md` with module boundary rules and event bus documentation

### Go Dependencies
- [ ] `github.com/gin-gonic/gin` — HTTP framework
- [ ] `github.com/jackc/pgx/v5` — PostgreSQL driver
- [ ] `github.com/sqlc-dev/sqlc` — query codegen (dev tool)
- [ ] `github.com/go-playground/validator/v10` — request validation
- [ ] `github.com/golang-jwt/jwt/v5` — JWT auth
- [ ] `golang.org/x/crypto/bcrypt` — password hashing
- [ ] `github.com/spf13/viper` — configuration
- [ ] `go.uber.org/zap` — structured logging
- [ ] `go.opentelemetry.io/otel` — observability
- [ ] `github.com/stretchr/testify` — test assertions
- [ ] `github.com/riverqueue/river` — background jobs
- [ ] `github.com/aws/aws-sdk-go-v2` — S3/MinIO storage
- [ ] `github.com/redis/go-redis/v9` — caching & rate limiting
- [ ] `ariga.io/atlas` — migrations (CLI tool)

---

## cmd/server — Application Entry Point & Wiring

### main.go
- [ ] Parse configuration via Viper (env vars, YAML file, CLI flags)
- [ ] Initialize zap logger (JSON format, structured, with `trace_id` field)
- [ ] Establish PostgreSQL connection pool via `pgxpool.New()` with:
  - [ ] `MaxConns` from config (default 25)
  - [ ] `MinConns` from config (default 5)
  - [ ] `MaxConnLifetime` = 30 min
  - [ ] `MaxConnIdleTime` = 5 min
- [ ] Initialize Redis client (`go-redis/v9`) with connection pooling
- [ ] Initialize MinIO/S3 client with presigned URL support
- [ ] Initialize River job queue client with `riverpgxv5.New(dbPool)`:
  - [ ] Register all job workers
  - [ ] Configure default queue with `MaxWorkers: 100`
  - [ ] Configure priority queues: `email` (50 workers), `webhook` (20 workers)
- [ ] Start River client (`riverClient.Start(ctx)`)

### Module Wiring
- [ ] Instantiate each module's repository from sqlc-generated code (`repository.New(dbPool)`)
- [ ] Instantiate each module's service with its dependencies injected:
  - [ ] `auth.NewService(authRepo, userRepo, redisClient, jwtConfig, emailService)`
  - [ ] `catalog.NewService(catalogRepo, s3Client, eventBus)`
  - [ ] `cart.NewService(cartRepo, inventoryService, redisClient, eventBus)`
  - [ ] `coupon.NewService(cartRepo)` (coupon admin CRUD shares cart repository for coupon queries)
  - [ ] `order.NewService(orderRepo, cartService, paymentService, inventoryService, eventBus)`
  - [ ] `payment.NewService(paymentRepo, orderService, stripeClient, eventBus)`
  - [ ] `inventory.NewService(inventoryRepo, redisClient, eventBus)`
  - [ ] `user.NewService(userRepo, s3Client)`
  - [ ] `notification.NewService(emailClient, eventBus)`
  - [ ] `review.NewService(reviewRepo, orderService)`
  - [ ] `analytics.NewService(analyticsRepo, orderRepo, productRepo)`
- [ ] Initialize internal event bus (`eventbus.New()`) with typed event channels
- [ ] Register all event handlers (cross-module subscriptions)

### Gin Router Setup
- [ ] Create `gin.New()` engine (NOT `gin.Default()` — manual middleware control)
- [ ] Register global middleware in order:
  1. [ ] `gin.Recovery()` — panic recovery
  2. [ ] `RequestIDMiddleware` — generate/propagate `X-Request-ID`
  3. [ ] `CORSMiddleware` — strict allowlist, credentials for same-site only
  4. [ ] `RateLimitMiddleware` — Redis token bucket (1000 req/min/user, 10 req/min/IP for login)
  5. [ ] `LoggerMiddleware` — zap structured log with `trace_id`, `user_id`, latency, status
  6. [ ] `OpenTelemetryMiddleware` — inject trace spans
  7. [ ] `ErrorHandlerMiddleware` — RFC 7807 Problem Details responses
- [ ] Create `/api/v1` route group
- [ ] Register each module's handler routes:
  - [ ] `auth.RegisterRoutes(v1, authService, jwtMiddleware)`
  - [ ] `catalog.RegisterRoutes(v1, catalogService, authMiddleware)`
  - [ ] `cart.RegisterRoutes(v1, cartService, optionalAuthMiddleware)`
  - [ ] `order.RegisterRoutes(v1, orderService, authMiddleware)`
  - [ ] `payment.RegisterRoutes(v1, paymentService)`
  - [ ] `inventory.RegisterRoutes(v1, inventoryService, authMiddleware)`
  - [ ] `user.RegisterRoutes(v1, userService, authMiddleware)`
  - [ ] `review.RegisterRoutes(v1, reviewService, authMiddleware)`
  - [ ] `analytics.RegisterRoutes(v1, analyticsService, authMiddleware)`
  - [ ] `coupon.RegisterRoutes(v1, couponService, authMiddleware)`
- [ ] Register Stripe webhook route: `POST /api/v1/webhooks/stripe` (raw body, no JSON parsing)

### Graceful Shutdown
- [ ] Listen for `SIGINT`, `SIGTERM` via `signal.NotifyContext`
- [ ] On signal: stop River client, drain in-flight jobs (30s timeout)
- [ ] Close Gin server with `srv.Shutdown(ctx)` (15s timeout)
- [ ] Close database pool, Redis client, MinIO client
- [ ] Flush zap logger

### Health & Readiness Endpoints
- [ ] `GET /health` — liveness probe (returns 200 if process alive)
- [ ] `GET /ready` — readiness probe (pings DB, Redis, returns 200 if all healthy)

---

## db — Database Schema, Migrations, sqlc Queries

### Atlas HCL Schema Definitions

#### Users & Auth Tables
- [ ] `users` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `email VARCHAR(255) UNIQUE NOT NULL`
  - [ ] `password_hash VARCHAR(255) NOT NULL`
  - [ ] `name VARCHAR(100) NOT NULL`
  - [ ] `avatar_url TEXT`
  - [ ] `phone VARCHAR(20)`
  - [ ] `role user_role NOT NULL DEFAULT 'customer'` (ENUM: guest, customer, seller, admin)
  - [ ] `status user_status NOT NULL DEFAULT 'active'` (ENUM: active, suspended, deactivated)
  - [ ] `email_verified_at TIMESTAMPTZ`
  - [ ] `created_at TIMESTAMPTZ DEFAULT now()`
  - [ ] `updated_at TIMESTAMPTZ DEFAULT now()`
  - [ ] `deleted_at TIMESTAMPTZ` (soft delete)
  - [ ] Index: `idx_users_email` UNIQUE on `email WHERE deleted_at IS NULL`

- [ ] `addresses` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `user_id UUID NOT NULL REFERENCES users(id)`
  - [ ] `full_name VARCHAR(200) NOT NULL`
  - [ ] `line1`, `line2`, `city`, `state`, `country`, `postal_code`
  - [ ] `is_default BOOLEAN DEFAULT false`
  - [ ] `created_at`, `updated_at`

#### Catalog Tables
- [ ] `categories` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `name VARCHAR(100) NOT NULL`
  - [ ] `slug VARCHAR(120) UNIQUE NOT NULL`
  - [ ] `parent_id UUID REFERENCES categories(id)` (self-referential FK)
  - [ ] `depth INTEGER DEFAULT 0`
  - [ ] `path ltree` (for hierarchical queries)
  - [ ] `created_at`, `updated_at`
  - [ ] Index: GIST on `path` for ltree queries

- [ ] `products` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `seller_id UUID NOT NULL REFERENCES users(id)`
  - [ ] `category_id UUID REFERENCES categories(id)`
  - [ ] `name VARCHAR(255) NOT NULL`
  - [ ] `slug VARCHAR(280) UNIQUE NOT NULL`
  - [ ] `description TEXT`
  - [ ] `status product_status NOT NULL DEFAULT 'draft'` (ENUM: draft, pending, active, archived)
  - [ ] `base_price BIGINT NOT NULL` (cents)
  - [ ] `currency VARCHAR(3) NOT NULL DEFAULT 'USD'`
  - [ ] `metadata JSONB DEFAULT '{}'`
  - [ ] `images TEXT[] DEFAULT '{}'`
  - [ ] `created_at`, `updated_at`, `archived_at`
  - [ ] Index: GIN on `to_tsvector('english', name || ' ' || description)`
  - [ ] Index: BTREE on `(category_id, status)`
  - [ ] Partial index: `WHERE status = 'active' AND archived_at IS NULL`

- [ ] `product_variants` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `product_id UUID NOT NULL REFERENCES products(id)`
  - [ ] `sku VARCHAR(50) UNIQUE NOT NULL`
  - [ ] `price_override BIGINT` (nullable, overrides base_price)
  - [ ] `options JSONB NOT NULL DEFAULT '{}'`
  - [ ] `images TEXT[] DEFAULT '{}'`
  - [ ] `created_at`, `updated_at`

#### Inventory Table
- [ ] `inventory` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `variant_id UUID UNIQUE NOT NULL REFERENCES product_variants(id)`
  - [ ] `physical_stock INTEGER NOT NULL DEFAULT 0`
  - [ ] `reserved_hard INTEGER NOT NULL DEFAULT 0`
  - [ ] `reorder_threshold INTEGER NOT NULL DEFAULT 5`
  - [ ] `updated_at`
  - [ ] CHECK constraint: `reserved_hard <= physical_stock`
  - [ ] CHECK constraint: `physical_stock >= 0`

#### Cart Tables
- [ ] `carts` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `user_id UUID REFERENCES users(id)` (nullable for guest)
  - [ ] `session_id VARCHAR(100)`
  - [ ] `coupon_code VARCHAR(50)`
  - [ ] `expires_at TIMESTAMPTZ`
  - [ ] `created_at`, `updated_at`
  - [ ] Index: `idx_carts_user_id` on `user_id WHERE user_id IS NOT NULL`
  - [ ] Index: `idx_carts_session_id` on `session_id WHERE session_id IS NOT NULL`

- [ ] `cart_items` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `cart_id UUID NOT NULL REFERENCES carts(id) ON DELETE CASCADE`
  - [ ] `variant_id UUID NOT NULL REFERENCES product_variants(id)`
  - [ ] `quantity INTEGER NOT NULL CHECK (quantity > 0)`
  - [ ] `unit_price_snapshot BIGINT NOT NULL`
  - [ ] UNIQUE constraint: `(cart_id, variant_id)`

#### Order Tables
- [ ] `orders` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `user_id UUID NOT NULL REFERENCES users(id)`
  - [ ] `status order_status NOT NULL DEFAULT 'pending_payment'`
  - [ ] `shipping_address JSONB NOT NULL`
  - [ ] `subtotal BIGINT NOT NULL`
  - [ ] `discount BIGINT DEFAULT 0`
  - [ ] `tax BIGINT DEFAULT 0`
  - [ ] `total BIGINT NOT NULL`
  - [ ] `currency VARCHAR(3) NOT NULL`
  - [ ] `idempotency_key VARCHAR(100) UNIQUE`
  - [ ] `coupon_code VARCHAR(50)`
  - [ ] `created_at`, `updated_at`
  - [ ] Index: BTREE on `(user_id, created_at DESC)`

- [ ] `order_items` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `order_id UUID NOT NULL REFERENCES orders(id)`
  - [ ] `variant_id UUID NOT NULL REFERENCES product_variants(id)`
  - [ ] `quantity INTEGER NOT NULL`
  - [ ] `unit_price BIGINT NOT NULL`
  - [ ] `product_snapshot JSONB NOT NULL`

- [ ] `order_events` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `order_id UUID NOT NULL REFERENCES orders(id)`
  - [ ] `status order_status NOT NULL`
  - [ ] `notes TEXT`
  - [ ] `created_by UUID REFERENCES users(id)`
  - [ ] `created_at TIMESTAMPTZ DEFAULT now()`
  - [ ] Index: `idx_order_events_order_id` on `order_id, created_at`

#### Payment Table
- [ ] `payments` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `order_id UUID NOT NULL REFERENCES orders(id)`
  - [ ] `gateway VARCHAR(20) NOT NULL` (stripe, paystack)
  - [ ] `gateway_payment_id VARCHAR(255)`
  - [ ] `amount BIGINT NOT NULL`
  - [ ] `currency VARCHAR(3) NOT NULL`
  - [ ] `status payment_status NOT NULL DEFAULT 'pending'`
  - [ ] `metadata JSONB DEFAULT '{}'`
  - [ ] `created_at`, `updated_at`
  - [ ] Index: `idx_payments_gateway_id` on `gateway_payment_id`

#### Review Table
- [ ] `reviews` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `product_id UUID NOT NULL REFERENCES products(id)`
  - [ ] `user_id UUID NOT NULL REFERENCES users(id)`
  - [ ] `order_id UUID NOT NULL REFERENCES orders(id)`
  - [ ] `rating SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5)`
  - [ ] `title VARCHAR(100) NOT NULL`
  - [ ] `body TEXT NOT NULL`
  - [ ] `status review_status NOT NULL DEFAULT 'pending'`
  - [ ] `moderated_by UUID REFERENCES users(id)`
  - [ ] `moderator_notes TEXT`
  - [ ] `created_at`, `updated_at`
  - [ ] UNIQUE constraint: `(product_id, user_id)`

#### Coupon Table
- [ ] `coupons` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `code VARCHAR(50) UNIQUE NOT NULL`
  - [ ] `type coupon_type NOT NULL` (ENUM: percent, fixed)
  - [ ] `value BIGINT NOT NULL`
  - [ ] `min_order_amount BIGINT DEFAULT 0`
  - [ ] `max_uses INTEGER DEFAULT 0` (0 = unlimited)
  - [ ] `used_count INTEGER DEFAULT 0`
  - [ ] `expires_at TIMESTAMPTZ`
  - [ ] `created_at`, `updated_at`, `deleted_at`
  - [ ] Partial index: `WHERE deleted_at IS NULL`

#### Wishlist Table
- [ ] `wishlist` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `user_id UUID NOT NULL REFERENCES users(id)`
  - [ ] `product_id UUID NOT NULL REFERENCES products(id)`
  - [ ] `created_at TIMESTAMPTZ DEFAULT now()`
  - [ ] UNIQUE constraint: `(user_id, product_id)`

#### Notification Table
- [ ] `notifications` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `user_id UUID NOT NULL REFERENCES users(id)`
  - [ ] `title VARCHAR(200) NOT NULL`
  - [ ] `message TEXT NOT NULL`
  - [ ] `type VARCHAR(50) NOT NULL`
  - [ ] `read BOOLEAN DEFAULT false`
  - [ ] `created_at TIMESTAMPTZ DEFAULT now()`

#### Audit & Utility Tables
- [ ] `inventory_audit_log` table:
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `variant_id UUID NOT NULL REFERENCES product_variants(id)`
  - [ ] `action VARCHAR(50) NOT NULL`
  - [ ] `old_stock INTEGER`, `new_stock INTEGER`
  - [ ] `reason TEXT`
  - [ ] `performed_by UUID REFERENCES users(id)`
  - [ ] `created_at TIMESTAMPTZ DEFAULT now()`

- [ ] `failed_emails` table (dead-letter):
  - [ ] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - [ ] `to_email VARCHAR(255) NOT NULL`
  - [ ] `subject VARCHAR(500)`
  - [ ] `template VARCHAR(100)`
  - [ ] `data JSONB`
  - [ ] `error TEXT`
  - [ ] `attempts INTEGER DEFAULT 0`
  - [ ] `created_at TIMESTAMPTZ DEFAULT now()`

### Enums
- [ ] Define all ENUM types in `enums.hcl`:
  - [ ] `user_role`: guest, customer, seller, admin
  - [ ] `user_status`: active, suspended, deactivated
  - [ ] `product_status`: draft, pending, active, archived
  - [ ] `order_status`: pending_payment, paid, processing, shipped, delivered, cancelled, refund_requested, refunded
  - [ ] `payment_status`: pending, succeeded, failed, refunded, partially_refunded
  - [ ] `review_status`: pending, approved, rejected
  - [ ] `coupon_type`: percent, fixed

### Triggers
- [ ] `updated_at` trigger function: auto-update `updated_at` on every UPDATE
- [ ] Apply trigger to all tables with `updated_at` column

### Extensions
- [ ] Enable `pgcrypto` (for `gen_random_uuid()`)
- [ ] Enable `ltree` (for category hierarchy)

### Atlas Migrations
- [ ] Run `atlas migrate diff --env dev initial_schema` to generate first migration
- [ ] Review generated SQL for correctness
- [ ] Run `atlas migrate apply --env dev` to apply to dev database
- [ ] Run `atlas migrate lint --env dev` to check for issues
- [ ] Version migrations in git (never edit applied migrations)

### sqlc Query Files
- [ ] `auth.sql` — user CRUD, password update, email verification
- [ ] `catalog.sql` — product CRUD, category tree, full-text search, variants
- [ ] `cart.sql` — cart CRUD, cart items, coupon CRUD, coupon validation
- [ ] `order.sql` — order CRUD, order items, order events, idempotency
- [ ] `payment.sql` — payment CRUD, gateway lookup
- [ ] `inventory.sql` — stock levels, reservations, audit log
- [ ] `user.sql` — profile, addresses, wishlist
- [ ] `review.sql` — review CRUD, moderation, summaries
- [ ] `analytics.sql` — revenue summary, orders by status, top products, sales by category
- [ ] `notification.sql` — notifications, failed emails
- [ ] Each file must use sqlc annotations: `-- name: QueryName :one`, `:many`, `:exec`, `:execresult`
- [ ] Use `sqlc.narg()` for nullable parameters
- [ ] Use `sqlc.embed()` for JOIN queries that return full rows

---

## pkg — Shared Packages

### pkg/apierr — Typed API Error Responses (RFC 7807)
- [ ] Create `apierr.go` implementing RFC 7807 Problem Details:
  - [ ] `ProblemDetail` struct: `Type` (URI), `Title`, `Status` (int), `Detail`, `Instance`, `Extensions` (map)
  - [ ] JSON tags: `type`, `title`, `status`, `detail`, `instance`
  - [ ] Implement `error` interface on `ProblemDetail`
- [ ] Define standard error constructors:
  - [ ] `BadRequest(detail string) *ProblemDetail` — status 400
  - [ ] `Unauthorized(detail string) *ProblemDetail` — status 401
  - [ ] `Forbidden(detail string) *ProblemDetail` — status 403
  - [ ] `NotFound(resource string) *ProblemDetail` — status 404
  - [ ] `Conflict(detail string) *ProblemDetail` — status 409
  - [ ] `ValidationFailed(errors []FieldError) *ProblemDetail` — status 422, includes field-level errors
  - [ ] `Internal(detail string) *ProblemDetail` — status 500
  - [ ] `TooManyRequests(detail string) *ProblemDetail` — status 429
- [ ] `FieldError` struct: `Field`, `Message`, `RejectedValue`
- [ ] Create `middleware.go` — Gin error handler middleware:
  - [ ] Check `c.Errors` after handler execution
  - [ ] If error is `*ProblemDetail`: write as JSON response with correct status
  - [ ] If error is `validator.ValidationErrors`: convert to `ValidationFailed` with field details
  - [ ] If unknown error: return `Internal("an unexpected error occurred")` (never leak internal details)
  - [ ] Set `Content-Type: application/problem+json`
- [ ] Create `sentinel.go` — predefined sentinel errors:
  - [ ] `ErrNotFound`, `ErrUnauthorized`, `ErrForbidden`, `ErrConflict`, `ErrInvalidStatusTransition`
  - [ ] Each with appropriate type URI (e.g., `https://nimcart.dev/errors/not-found`)

#### Tests
- [ ] Test each constructor produces correct status and structure
- [ ] Test middleware: ProblemDetail error → correct JSON response
- [ ] Test middleware: validation errors → field-level detail
- [ ] Test middleware: unknown error → generic 500 (no leak)

### pkg/pagination — Cursor-Based Pagination Helpers
- [ ] Create `pagination.go`:
  - [ ] `Cursor` struct: `ID` (uuid.UUID), `CreatedAt` (time.Time)
  - [ ] `Encode(cursor) string` — base64-encode `{id}:{created_at_rfc3339}`
  - [ ] `Decode(encoded string) (*Cursor, error)` — base64-decode, parse fields
  - [ ] `Params` struct: `Cursor` (optional string), `Limit` (int, default 20, max 100)
  - [ ] `ParseParams(c *gin.Context) (*Params, error)` — extract from query params, validate limit bounds
  - [ ] `Response[T any]` struct: `Data []T`, `NextCursor` (string, nullable), `HasMore` (bool)
  - [ ] `NewResponse[T](items []T, limit int) *Response[T]`:
    - [ ] If `len(items) > limit`: set `HasMore = true`, `NextCursor` = encode last item's cursor, trim items to limit
    - [ ] If `len(items) <= limit`: `HasMore = false`, `NextCursor = nil`
- [ ] Create `sql_helpers.go`:
  - [ ] `WhereClause(cursor *Cursor) string` — generate `WHERE (created_at, id) < ($1, $2)` for backward pagination
  - [ ] `OrderByClause() string` — `ORDER BY created_at DESC, id DESC`
  - [ ] `LimitClause(limit int) int` — return `limit + 1` (fetch one extra to determine HasMore)

#### Tests
- [ ] Test encode/decode roundtrip
- [ ] Test decode with invalid base64
- [ ] Test ParseParams: default limit, max limit, invalid cursor
- [ ] Test NewResponse: has more items, no more items, empty list

### pkg/money — Monetary Value Type
- [ ] Create `money.go`:
  - [ ] `Money` struct: `Amount` (int64, in smallest currency unit — cents), `Currency` (string, ISO 4217)
  - [ ] `New(amount int64, currency string) Money` — constructor with validation
  - [ ] `FromFloat(amount float64, currency string) Money` — convert from float (for API input only, round to cents)
  - [ ] Arithmetic methods (all return new Money, never mutate):
    - [ ] `Add(other Money) (Money, error)` — same currency check
    - [ ] `Subtract(other Money) (Money, error)` — same currency check
    - [ ] `Multiply(factor int64) Money` — integer multiplication (for quantity)
    - [ ] `Percent(pct int64) Money` — calculate percentage (for discounts: `amount * pct / 100`)
  - [ ] Comparison methods:
    - [ ] `Equal(other Money) bool`
    - [ ] `GreaterThan(other Money) bool`
    - [ ] `LessThan(other Money) bool`
    - [ ] `IsZero() bool`
    - [ ] `IsNegative() bool`
  - [ ] Display methods:
    - [ ] `String() string` — formatted: "$12.34 USD"
    - [ ] `ToFloat() float64` — for display only, never for computation
  - [ ] JSON marshaling:
    - [ ] `MarshalJSON()` — `{"amount": 1234, "currency": "USD"}`
    - [ ] `UnmarshalJSON()` — parse from object
- [ ] Define supported currencies: `USD`, `NGN`, `EUR`
- [ ] Currency-specific decimal places: USD=2, NGN=2, EUR=2

#### Tests
- [ ] Test arithmetic: add, subtract, multiply, percent
- [ ] Test currency mismatch: add USD + NGN returns error
- [ ] Test JSON roundtrip
- [ ] Test display formatting
- [ ] Test zero and negative values
- [ ] Test no float precision loss (all internal math is integer)

---

## internal/platform — Shared Infrastructure

### Standards & Conventions
- **No business logic:** Platform provides infrastructure only. No domain types.
- **Single responsibility per sub-package:** `db/`, `config/`, `eventbus/`, `logger/`.
- **All other modules depend on platform, but platform depends on nothing internal.**

### platform/config — Configuration
- [ ] Create `config.go` with Viper-based configuration loader:
  - [ ] Load from env vars (prefix `NIMCART_`), YAML file (`config.yaml`), CLI flags
  - [ ] Support hot-reload via `viper.WatchConfig()`
- [ ] Define `Config` struct with all application settings:
  - [ ] `Server` — `Port`, `Host`, `ReadTimeout`, `WriteTimeout`, `ShutdownTimeout`
  - [ ] `Database` — `URL`, `MaxConns`, `MinConns`, `MaxConnLifetime`, `MaxConnIdleTime`
  - [ ] `Redis` — `URL`, `PoolSize`, `MinIdleConns`
  - [ ] `JWT` — `AccessTTL` (15m), `RefreshTTL` (7d), `RSAPrivateKeyPath`, `RSAPublicKeyPath`
  - [ ] `Stripe` — `SecretKey`, `WebhookSecret`, `PublishableKey`
  - [ ] `Paystack` — `SecretKey`, `WebhookSecret`
  - [ ] `S3` — `Endpoint`, `Bucket`, `AccessKey`, `SecretKey`, `Region`, `UseSSL`
  - [ ] `Email` — `Provider` (sendgrid/smtp), `APIKey`, `FromAddress`, `FromName`, `SMTPHost`, `SMTPPort`
  - [ ] `RateLimit` — `GeneralRPM` (1000), `LoginRPM` (10)
  - [ ] `CORS` — `AllowedOrigins` (slice)
  - [ ] `LogLevel` — `debug`, `info`, `warn`, `error`
- [ ] `Load() (*Config, error)` — parse all sources, validate required fields, return config
- [ ] Validate: all required fields present, URLs parseable, TTLs positive
- [ ] Support environment-specific defaults (dev vs prod)

#### Tests
- [ ] Test config loading from env vars
- [ ] Test config loading from YAML file
- [ ] Test validation of missing required fields

### platform/db — Database Connection Pool
- [ ] Create `pool.go` with PostgreSQL connection pool initialization:
  - [ ] `NewPool(ctx, config) (*pgxpool.Pool, error)`
  - [ ] Configure `pgxpool.Config` from `config.Database`:
    - [ ] `MaxConns`, `MinConns`, `MaxConnLifetime`, `MaxConnIdleTime`
    - [ ] `HealthCheckPeriod` = 30s
  - [ ] Set `AfterConnect` callback: prepare statements, set `search_path`
  - [ ] Set `BeforeAcquire` callback: validate connection health
- [ ] Create `health.go` with DB health check:
  - [ ] `Ping(ctx, pool) error` — execute `SELECT 1`, return error if timeout
- [ ] Create `tx.go` with transaction helper:
  - [ ] `WithTransaction(ctx, pool, fn func(tx pgx.Tx) error) error`
  - [ ] Begin tx, execute fn, commit on success, rollback on error
  - [ ] Support nested savepoints if needed
- [ ] Create `migrate.go` for Atlas migration runner (optional, for programmatic migrations)

#### Tests
- [ ] Test pool creation with valid config
- [ ] Test health check with mocked pool
- [ ] Test transaction rollback on error

### platform/eventbus — Internal Event Bus
- [ ] Create `event.go` with typed event definitions:
  - [ ] `Event` interface: `EventName() string`, `OccurredAt() time.Time`
  - [ ] Define all domain events:
    - [ ] `OrderPlaced` — `OrderID`, `UserID`, `Total`, `Currency`
    - [ ] `OrderCancelled` — `OrderID`, `Reason`
    - [ ] `OrderRefundRequested` — `OrderID`, `Reason`
    - [ ] `OrderShipped` — `OrderID`, `TrackingNumber`
    - [ ] `PaymentConfirmed` — `OrderID`, `PaymentID`, `Amount`
    - [ ] `PaymentFailed` — `OrderID`, `PaymentID`, `Reason`
    - [ ] `RefundProcessed` — `OrderID`, `PaymentID`, `Amount`
    - [ ] `InventoryDepleted` — `VariantID`
    - [ ] `InventoryLowStock` — `VariantID`, `Available`
    - [ ] `ReviewSubmitted` — `ReviewID`, `ProductID`, `UserID`
    - [ ] `ReviewApproved` — `ReviewID`, `ProductID`
    - [ ] `ReviewRejected` — `ReviewID`, `ProductID`, `Reason`
    - [ ] `PasswordResetRequested` — `UserID`, `Email`, `Token`
- [ ] Create `bus.go` with event bus implementation:
  - [ ] `Bus` struct with typed channels per event type (Go generics)
  - [ ] `Publish(ctx, event)` — send event to channel (non-blocking, drop if full with log warning)
  - [ ] `Subscribe[T Event](handler func(T))` — register handler for event type
  - [ ] `Start(ctx)` — launch goroutines per event type, fan-out to handlers
  - [ ] `Stop()` — drain channels, wait for in-flight handlers (with timeout)
- [ ] Create `middleware.go` for event bus middleware:
  - [ ] Logging middleware: log event name, handler, duration
  - [ ] Recovery middleware: catch panics in handlers
  - [ ] Metrics middleware: count events published/handled/failed
- [ ] Buffer size configurable per event type (default 1000)
- [ ] Dead-letter: events that fail all retries logged to `failed_events` table

#### Tests
- [ ] Test publish/subscribe: handler receives event
- [ ] Test multiple subscribers: all receive event
- [ ] Test panic recovery: bus continues after handler panic
- [ ] Test graceful shutdown: in-flight handlers complete
- [ ] Test buffer overflow: warning logged, event dropped

### platform/logger — Structured Logging
- [ ] Create `logger.go` with zap logger initialization:
  - [ ] `NewLogger(level, format) (*zap.Logger, error)`
  - [ ] JSON format for production, console for development
  - [ ] Configure fields: `timestamp`, `level`, `caller`, `message`
  - [ ] Add custom fields: `service = "nimcart"`, `version`
- [ ] Create `gin_logger.go` — Gin middleware for request logging:
  - [ ] Log: `method`, `path`, `query`, `status`, `latency`, `client_ip`, `body_size`, `trace_id`, `user_id`
  - [ ] Extract `user_id` from Gin context (set by auth middleware)
  - [ ] Extract/generate `trace_id` from OpenTelemetry span
  - [ ] Log at `Info` for 2xx/3xx, `Warn` for 4xx, `Error` for 5xx
- [ ] Create `context.go` — context-aware logger:
  - [ ] `FromContext(ctx) *zap.Logger` — extract logger with request-scoped fields
  - [ ] `WithContext(ctx, logger) context.Context` — store logger in context

#### Tests
- [ ] Test JSON output format
- [ ] Test log level filtering
- [ ] Test context propagation of request fields

---

## internal/auth — JWT Authentication, Registration, RBAC

### Standards & Conventions
- **Gin best practice:** Handlers are thin — validate input, call service, format response. No business logic in handlers.
- **Error responses:** RFC 7807 Problem Details via `pkg/apierr`.
- **Validation:** `go-playground/validator/v10` struct tags on all request DTOs.
- **JWT:** RS256 asymmetric signing. Access token 15 min TTL. Refresh token 7-day rotating.
- **Password:** bcrypt cost 12. Never log or return password fields.

### Types & DTOs
- [ ] `RegisterRequest` — `email` (required, email), `password` (required, min 8, max 128), `name` (required, min 2)
- [ ] `LoginRequest` — `email` (required, email), `password` (required)
- [ ] `RefreshRequest` — `refresh_token` (required)
- [ ] `ForgotPasswordRequest` — `email` (required, email)
- [ ] `ResetPasswordRequest` — `token` (required), `password` (required, min 8)
- [ ] `AuthResponse` — `access_token`, `refresh_token`, `expires_in`, `token_type`
- [ ] `UserResponse` — `id`, `email`, `name`, `role`, `status`, `email_verified_at`, `created_at`

### Service Layer (`service.go`)
- [ ] `Register(ctx, req) (*AuthResponse, error)` — hash password (bcrypt 12), create user in DB, issue JWT pair
- [ ] `Login(ctx, req) (*AuthResponse, error)` — verify password, check user status (active), issue JWT pair
- [ ] `Refresh(ctx, refreshToken) (*AuthResponse, error)` — validate refresh token, rotate (invalidate old, issue new pair), check Redis blacklist
- [ ] `Logout(ctx, userID, refreshToken) error` — add refresh token to Redis blacklist (TTL = token expiry)
- [ ] `ForgotPassword(ctx, email) error` — generate reset token (crypto/rand, 32 bytes), store in Redis (TTL 1 hour), enqueue email via River job
- [ ] `ResetPassword(ctx, token, newPassword) error` — validate token from Redis, hash new password, update DB, invalidate all existing refresh tokens
- [ ] `GetMe(ctx, userID) (*UserResponse, error)` — fetch user profile from DB

### JWT Utilities
- [ ] `GenerateAccessToken(user) (string, error)` — RS256 sign, claims: `sub` (user ID), `role`, `exp` (15 min), `iat`, `jti`
- [ ] `GenerateRefreshToken(user) (string, error)` — opaque token stored in Redis, TTL 7 days
- [ ] `ValidateAccessToken(tokenString) (*Claims, error)` — verify signature, check expiry
- [ ] `ValidateRefreshToken(tokenString) (*Claims, error)` — check Redis existence, not blacklisted
- [ ] Load RS256 key pair from config/Vault at startup

### Middleware (`middleware.go`)
- [ ] `JWTAuthMiddleware` — extract Bearer token from `Authorization` header, validate, set user in Gin context
- [ ] `OptionalAuthMiddleware` — same as JWT but does NOT abort on missing/invalid token (for guest cart)
- [ ] `RequireRole(roles ...string)` — check `user.role` in allowed roles, return 403 if not
- [ ] `RefreshTokenMiddleware` — validate refresh token from `HttpOnly` cookie for `/auth/refresh` endpoint

### Handler Layer (`handler.go`)
- [ ] `POST /api/v1/auth/register` — bind+validate `RegisterRequest`, call `service.Register`, return 201 + `AuthResponse`
- [ ] `POST /api/v1/auth/login` — bind+validate `LoginRequest`, call `service.Login`, set refresh token as `HttpOnly; Secure; SameSite=Strict` cookie, return `AuthResponse`
- [ ] `POST /api/v1/auth/refresh` — extract refresh token from cookie, call `service.Refresh`, rotate cookie, return new `AuthResponse`
- [ ] `POST /api/v1/auth/logout` — require auth, call `service.Logout`, clear cookie, return 204
- [ ] `POST /api/v1/auth/forgot-password` — bind+validate, call `service.ForgotPassword`, return 200 (always, even if email not found — prevent enumeration)
- [ ] `POST /api/v1/auth/reset-password` — bind+validate, call `service.ResetPassword`, return 200
- [ ] `GET /api/v1/auth/me` — require auth, call `service.GetMe`, return `UserResponse`

### Repository Layer (`repository/`)
- [ ] `db/queries/auth.sql` — write SQL queries:
  - [ ] `CreateUser` — INSERT with email, password_hash, name, role='customer'
  - [ ] `GetUserByEmail` — SELECT by email
  - [ ] `GetUserByID` — SELECT by id
  - [ ] `UpdatePassword` — UPDATE password_hash, updated_at
  - [ ] `UpdateEmailVerified` — UPDATE email_verified_at
  - [ ] `InvalidateRefreshTokens` — DELETE all refresh tokens for user
- [ ] Run `sqlc generate` to produce Go types in `repository/`

### Tests
- [ ] Unit tests for `service.go` using `testify` + `gomock`:
  - [ ] Register: success, duplicate email, weak password
  - [ ] Login: success, wrong password, inactive user, non-existent user
  - [ ] Refresh: success, expired token, blacklisted token
  - [ ] ForgotPassword: success, non-existent email (should not error)
  - [ ] ResetPassword: success, expired token, invalid token
- [ ] Integration tests for handlers using `httptest`:
  - [ ] Full register → login → refresh → logout flow
  - [ ] Rate limiting on login (10 req/min/IP)
- [ ] Middleware tests:
  - [ ] Valid token → user in context
  - [ ] Expired token → 401
  - [ ] Missing token → 401
  - [ ] Role check → 403 for unauthorized role

### Security Checklist
- [ ] Password never appears in logs, responses, or error messages
- [ ] Refresh token stored in `HttpOnly; Secure; SameSite=Strict` cookie — never `localStorage`
- [ ] Access token kept in memory only (frontend) — never persisted
- [ ] RS256 private key loaded from Vault/env, never committed to source
- [ ] Rate limiting on login and forgot-password endpoints
- [ ] Constant-time password comparison via bcrypt
- [ ] User enumeration prevention: forgot-password returns same response regardless of email existence

---

## internal/catalog — Products, Categories, Variants, Search, Images

### Standards & Conventions
- **Gin:** Thin handlers — validate, delegate to service, respond. Use `c.Error()` for error middleware.
- **sqlc:** All DB access via generated code from `db/queries/catalog.sql`. No raw SQL in service layer.
- **Pagination:** Cursor-based (keyset) via `pkg/pagination`. Opaque cursor = base64-encoded `(id, created_at)`.
- **Money:** Use `pkg/money.Money` type (integer cents + currency code). Never float arithmetic.
- **Validation:** `go-playground/validator/v10` on all request DTOs.
- **S3:** Product images uploaded via presigned URLs. Service returns presigned PUT URL; client uploads directly.

### Types & DTOs
- [ ] `Product` — exported domain type: `ID`, `SellerID`, `CategoryID`, `Name`, `Slug`, `Description`, `Status`, `BasePrice` (money.Money), `Currency`, `Metadata` (JSONB), `Variants`, `Images`, `CreatedAt`, `UpdatedAt`
- [ ] `ProductStatus` — enum: `Draft`, `PendingReview`, `Active`, `Archived`
- [ ] `CreateProductRequest` — `name`, `description`, `category_id`, `base_price` (int cents), `currency`, `metadata`
- [ ] `UpdateProductRequest` — all fields optional (pointer types)
- [ ] `UpdateStatusRequest` — `status` (required, oneof=draft pending active archived)
- [ ] `ProductFilter` — `category_id`, `min_price`, `max_price`, `tags`, `sort` (price_asc, price_desc, newest), `cursor`, `limit`
- [ ] `SearchRequest` — `q` (required, min 2 chars), `cursor`, `limit`
- [ ] `Category` — `ID`, `Name`, `Slug`, `ParentID`, `Depth`, `Path` (ltree), `Children`
- [ ] `CreateCategoryRequest` — `name`, `parent_id` (optional)
- [ ] `Variant` — `ID`, `ProductID`, `SKU`, `PriceOverride`, `Options` (JSONB), `Images`
- [ ] `CreateVariantRequest` — `sku`, `price_override`, `options`
- [ ] `ImageUploadResponse` — `upload_url` (presigned PUT), `image_key`, `expires_in`

### Service Layer (`service.go`)
- [ ] `ListProducts(ctx, filter) (*PaginatedProducts, error)` — query active products with filters, cursor pagination
- [ ] `GetProductBySlug(ctx, slug) (*Product, error)` — fetch with variants and images
- [ ] `SearchProducts(ctx, query) (*PaginatedProducts, error)` — PostgreSQL full-text search via `plainto_tsquery` on `tsvector` index
- [ ] `CreateProduct(ctx, sellerID, req) (*Product, error)` — create in Draft status, generate slug (unique, URL-safe)
- [ ] `UpdateProduct(ctx, productID, sellerID, req) (*Product, error)` — verify ownership (seller_id match), update fields
- [ ] `TransitionStatus(ctx, productID, adminID, req) error` — FSM validation: Draft→PendingReview→Active→Archived
- [ ] `DeleteProduct(ctx, productID) error` — soft delete (set `archived_at`, `status = archived`)
- [ ] `ListCategories(ctx) ([]Category, error)` — fetch category tree using ltree `path` for hierarchical ordering
- [ ] `CreateCategory(ctx, req) (*Category, error)` — auto-generate slug, compute `depth` and `path` from parent
- [ ] `CreateVariant(ctx, productID, req) (*Variant, error)` — validate SKU uniqueness
- [ ] `InitiateImageUpload(ctx, productID, filename, contentType) (*ImageUploadResponse, error)` — validate MIME type, generate S3 key, return presigned PUT URL (15 min expiry)
- [ ] `ConfirmImageUpload(ctx, productID, imageKey) error` — add image key to product's images array

### FSM (Product Status)
- [ ] Implement state machine with valid transitions:
  - `Draft` → `PendingReview` (seller submits)
  - `PendingReview` → `Active` (admin approves)
  - `PendingReview` → `Draft` (admin rejects)
  - `Active` → `Archived` (admin or seller archives)
- [ ] Return `ErrInvalidStatusTransition` for invalid moves

### Handler Layer (`handler.go`)
- [ ] `GET /api/v1/products` — parse query params into `ProductFilter`, call `ListProducts`, return paginated response with `next_cursor`
- [ ] `GET /api/v1/products/:slug` — call `GetProductBySlug`, return product with variants/images
- [ ] `GET /api/v1/products/search` — parse `q` param, call `SearchProducts`, return paginated results
- [ ] `POST /api/v1/products` — require Seller/Admin role, bind+validate, call `CreateProduct`, return 201
- [ ] `PUT /api/v1/products/:id` — require Seller/Admin, verify ownership, call `UpdateProduct`
- [ ] `PATCH /api/v1/products/:id/status` — require Admin, call `TransitionStatus`
- [ ] `DELETE /api/v1/products/:id` — require Admin, call `DeleteProduct`, return 204
- [ ] `GET /api/v1/categories` — call `ListCategories`, return tree
- [ ] `POST /api/v1/categories` — require Admin, call `CreateCategory`, return 201
- [ ] `POST /api/v1/products/:id/images` — require Seller/Admin, validate filename/MIME, call `InitiateImageUpload`, return presigned URL

### Repository Layer (`repository/`)
- [ ] `db/queries/catalog.sql` — write SQL:
  - [ ] `ListProducts` — SELECT with WHERE filters, ORDER BY, LIMIT, cursor-based keyset pagination
  - [ ] `GetProductBySlug` — SELECT with JOIN variants, images
  - [ ] `SearchProducts` — SELECT with `WHERE tsvector_col @@ plainto_tsquery($1)`, rank ordering
  - [ ] `CreateProduct` — INSERT returning *
  - [ ] `UpdateProduct` — UPDATE with dynamic SET (sqlc `sqlc.narg()` for nullable fields)
  - [ ] `UpdateProductStatus` — UPDATE status, updated_at
  - [ ] `SoftDeleteProduct` — UPDATE archived_at, status
  - [ ] `ListCategories` — SELECT ordered by `path` (ltree)
  - [ ] `CreateCategory` — INSERT with computed path
  - [ ] `CreateVariant` — INSERT with SKU unique constraint
  - [ ] `GetVariantBySKU` — SELECT for uniqueness check
  - [ ] `AddProductImage` — UPDATE images array append
- [ ] Run `sqlc generate`

### Tests
- [ ] Unit tests for service:
  - [ ] Product CRUD: success cases, ownership validation, slug generation
  - [ ] FSM transitions: all valid + invalid transitions
  - [ ] Search: empty query, special characters
  - [ ] Image upload: invalid MIME type, oversized file
- [ ] Integration tests:
  - [ ] Full product lifecycle: create → submit → approve → archive
  - [ ] Category tree: nested creation, path computation
  - [ ] Pagination: cursor continuity, empty pages
- [ ] Repository tests with real PostgreSQL (testcontainers or CI DB)

### Performance
- [ ] GIN index on `to_tsvector('english', name || ' ' || description)` for full-text search
- [ ] BTREE index on `(category_id, status)` for catalog listing
- [ ] Partial index on `products WHERE status = 'active' AND deleted_at IS NULL`
- [ ] Cache hot product pages in Redis (TTL 5 min, invalidate on update)

---

## internal/cart — Guest Session Cart, User Cart, Coupon Application, Cart Merge

### Standards & Conventions
- **Gin:** Handlers validate input, delegate to service, respond. Use optional auth middleware for guest/authenticated duality.
- **Redis:** Guest cart stored as Redis hash with 30-day TTL. Key: `cart:session:{session_id}`.
- **PostgreSQL:** Authenticated user cart persisted in `carts` + `cart_items` tables.
- **Stock validation:** Real-time stock check on every cart mutation via inventory service.
- **Money:** All price calculations use `pkg/money.Money` (integer cents). No floats.

### Types & DTOs
- [ ] `Cart` — `ID`, `UserID` (nullable), `SessionID`, `Items []CartItem`, `Subtotal`, `Discount`, `Total`, `CouponCode` (nullable), `ExpiresAt`
- [ ] `CartItem` — `ID`, `VariantID`, `ProductID`, `ProductName`, `VariantOptions` (JSONB), `Quantity`, `UnitPrice` (money.Money), `StockAvailable` (int), `ImageURL`
- [ ] `AddItemRequest` — `variant_id` (required, UUID), `quantity` (required, min 1, max 99)
- [ ] `UpdateItemRequest` — `quantity` (required, min 1, max 99)
- [ ] `ApplyCouponRequest` — `code` (required, alphanumeric)
- [ ] `CartResponse` — full cart with computed totals

### Service Layer (`service.go`)
- [ ] `GetCart(ctx, userID, sessionID) (*Cart, error)` — return user cart (if authenticated) or session cart (from Redis). Hydrate with real-time stock and current prices.
- [ ] `AddItem(ctx, userID, sessionID, req) (*Cart, error)`:
  - [ ] Validate variant exists and is active
  - [ ] Check available stock via inventory service
  - [ ] Create soft stock reservation in Redis (TTL 30 min) via inventory service
  - [ ] If item already in cart, increment quantity (re-validate stock)
  - [ ] Snapshot unit price at time of add
  - [ ] Recalculate totals
- [ ] `UpdateItemQuantity(ctx, userID, sessionID, itemID, req) (*Cart, error)`:
  - [ ] Validate new quantity against available stock
  - [ ] Update soft reservation TTL
  - [ ] Recalculate totals
- [ ] `RemoveItem(ctx, userID, sessionID, itemID) (*Cart, error)`:
  - [ ] Release soft stock reservation via inventory service
  - [ ] Remove item from cart
  - [ ] Recalculate totals
- [ ] `ClearCart(ctx, userID, sessionID) error` — release all soft reservations, delete cart
- [ ] `ApplyCoupon(ctx, userID, req) (*Cart, error)`:
  - [ ] Validate coupon exists, not expired, usage limit not reached
  - [ ] Check minimum order amount
  - [ ] Calculate discount (percent or fixed)
  - [ ] Apply to cart, recalculate totals
- [ ] `MergeCarts(ctx, userID, sessionID) (*Cart, error)`:
  - [ ] Fetch guest cart from Redis
  - [ ] For each guest item: check stock, merge into user cart (skip duplicates, sum quantities)
  - [ ] Release guest cart soft reservations
  - [ ] Delete guest cart from Redis
  - [ ] Return merged user cart

### Guest vs Authenticated Cart Logic
- [ ] If `userID` is present: use PostgreSQL cart
- [ ] If only `sessionID`: use Redis cart
- [ ] On login: trigger `MergeCarts` automatically (called from auth handler post-login)
- [ ] Session ID generation: crypto/rand, stored in cookie for guests

### Coupon Logic
- [ ] Percent coupon: `discount = subtotal * (value / 100)`, capped at subtotal
- [ ] Fixed coupon: `discount = min(value, subtotal)`
- [ ] Validate: `expires_at > now()`, `used_count < max_uses`, `subtotal >= min_order_amount`
- [ ] Increment `used_count` on order creation (not on apply)

### Handler Layer (`handler.go`)
- [ ] `GET /api/v1/cart` — optional auth, call `GetCart`, return `CartResponse`
- [ ] `POST /api/v1/cart/items` — optional auth, bind+validate `AddItemRequest`, call `AddItem`, return 200 + updated cart
- [ ] `PATCH /api/v1/cart/items/:itemId` — optional auth, bind+validate `UpdateItemRequest`, call `UpdateItemQuantity`
- [ ] `DELETE /api/v1/cart/items/:itemId` — optional auth, call `RemoveItem`
- [ ] `DELETE /api/v1/cart` — optional auth, call `ClearCart`, return 204
- [ ] `POST /api/v1/cart/apply-coupon` — require Customer role, bind+validate, call `ApplyCoupon`
- [ ] `POST /api/v1/cart/merge` — require Customer role, call `MergeCarts`

### Repository Layer (`repository/`)
- [ ] `db/queries/cart.sql`:
  - [ ] `GetCartByUserID` — SELECT cart with items (JOIN cart_items, product_variants, products)
  - [ ] `GetCartBySessionID` — SELECT for session-based carts
  - [ ] `CreateCart` — INSERT new cart
  - [ ] `AddCartItem` — INSERT or UPDATE (ON CONFLICT increment quantity)
  - [ ] `UpdateCartItemQuantity` — UPDATE quantity, unit_price_snapshot
  - [ ] `RemoveCartItem` — DELETE cart_item
  - [ ] `ClearCartItems` — DELETE all items for cart
  - [ ] `GetCartItem` — SELECT single item for validation
  - [ ] `ApplyCouponToCart` — UPDATE coupon_code on cart
  - [ ] `GetCouponByCode` — SELECT coupon for validation
  - [ ] `IncrementCouponUsage` — UPDATE used_count + 1
- [ ] Run `sqlc generate`

### Redis Operations (Guest Cart)
- [ ] `getGuestCart(sessionID)` — HGETALL `cart:session:{session_id}`
- [ ] `setGuestCart(sessionID, cart)` — HSET + EXPIRE 30 days
- [ ] `addGuestCartItem(sessionID, item)` — HSET field per item
- [ ] `removeGuestCartItem(sessionID, itemID)` — HDEL
- [ ] `clearGuestCart(sessionID)` — DEL key

### Tests
- [ ] Unit tests:
  - [ ] AddItem: stock available, stock insufficient, variant not found
  - [ ] UpdateQuantity: within stock, exceeds stock
  - [ ] RemoveItem: releases reservation
  - [ ] MergeCarts: duplicate items, guest-only items, stock conflicts
  - [ ] ApplyCoupon: valid, expired, over limit, below minimum order
- [ ] Integration tests:
  - [ ] Full cart lifecycle: add → update → apply coupon → clear
  - [ ] Guest to authenticated merge flow
  - [ ] Concurrent cart modifications (race conditions)

---

## internal/order — Order Lifecycle FSM, Order Creation, Cancellation, Refunds

### Standards & Conventions
- **Gin:** Thin handlers. All business logic in service layer.
- **FSM:** Order status transitions enforced in service, not DB. Invalid transitions return typed error.
- **Idempotency:** `X-Idempotency-Key` header required on `POST /api/v1/orders`. Duplicate keys return cached response.
- **Money:** All monetary fields use `pkg/money.Money`. Totals computed server-side, never trusted from client.
- **Events:** Publish `OrderPlaced`, `OrderCancelled`, `OrderRefundRequested` to internal event bus.
- **Audit log:** Every status change creates an immutable `order_events` row.

### Types & DTOs
- [ ] `Order` — `ID`, `UserID`, `Status`, `ShippingAddress` (JSONB), `Subtotal`, `Discount`, `Tax`, `Total`, `Currency`, `IdempotencyKey`, `Items []OrderItem`, `Events []OrderEvent`, `CreatedAt`, `UpdatedAt`
- [ ] `OrderStatus` — enum: `PendingPayment`, `Paid`, `Processing`, `Shipped`, `Delivered`, `Cancelled`, `RefundRequested`, `Refunded`
- [ ] `OrderItem` — `ID`, `OrderID`, `VariantID`, `Quantity`, `UnitPrice`, `ProductSnapshot` (JSONB: name, slug, image, options at time of order)
- [ ] `OrderEvent` — `ID`, `OrderID`, `Status`, `Notes`, `CreatedBy`, `CreatedAt`
- [ ] `CreateOrderRequest` — `shipping_address_id` (required, UUID), `idempotency_key` (from header), `coupon_code` (optional)
- [ ] `UpdateStatusRequest` — `status` (required), `notes` (optional), `tracking_number` (optional, for Shipped)
- [ ] `RefundRequest` — `reason` (required, min 10 chars)
- [ ] `OrderFilter` — `status`, `date_from`, `date_to`, `user_id` (admin), `cursor`, `limit`

### Service Layer (`service.go`)
- [ ] `CreateOrder(ctx, userID, req) (*Order, error)`:
  - [ ] Check idempotency key — if exists, return cached order
  - [ ] Fetch user's cart, validate non-empty
  - [ ] Validate shipping address belongs to user
  - [ ] For each cart item: validate stock, convert soft reservation → hard reservation
  - [ ] Compute subtotal from cart items (unit_price_snapshot * quantity)
  - [ ] Apply coupon discount if present
  - [ ] Compute tax (configurable rate or 0 for v1)
  - [ ] Compute total = subtotal - discount + tax
  - [ ] Create order + order_items in a single DB transaction
  - [ ] Snapshot product data into `product_snapshot` JSONB (name, slug, image, variant options)
  - [ ] Create initial `order_event` (status = PendingPayment)
  - [ ] Clear user's cart
  - [ ] Publish `OrderPlaced` event to event bus
  - [ ] Enqueue payment intent creation via River job
  - [ ] Store idempotency key → order_id mapping in Redis (TTL 24h)
- [ ] `GetOrder(ctx, userID, orderID) (*Order, error)` — fetch with items + events, verify ownership
- [ ] `ListOrders(ctx, userID, filter) (*PaginatedOrders, error)` — cursor pagination, own orders only
- [ ] `CancelOrder(ctx, userID, orderID) (*Order, error)`:
  - [ ] Validate status is `PendingPayment` or `Paid`
  - [ ] Transition to `Cancelled`
  - [ ] Release hard inventory reservations
  - [ ] If `Paid`: enqueue refund via payment service (River job)
  - [ ] Create `order_event` with cancellation notes
  - [ ] Publish `OrderCancelled` event
- [ ] `UpdateOrderStatus(ctx, adminID, orderID, req) (*Order, error)`:
  - [ ] Validate FSM transition from current status
  - [ ] Update status, create `order_event`
  - [ ] If `Shipped`: store tracking number in order metadata
  - [ ] Publish appropriate event
- [ ] `RequestRefund(ctx, userID, orderID, req) (*Order, error)`:
  - [ ] Validate status is `Delivered`
  - [ ] Transition to `RefundRequested`
  - [ ] Create `order_event` with reason
  - [ ] Publish `OrderRefundRequested` event
  - [ ] Enqueue refund review notification (River job)
- [ ] `ListAllOrders(ctx, filter) (*PaginatedOrders, error)` — admin/seller: all orders with advanced filters

### FSM (Order Status)
- [ ] Valid transitions:
  - `PendingPayment` → `Paid` (webhook), `Cancelled` (user/admin)
  - `Paid` → `Processing` (admin/seller), `Cancelled` (user/admin, triggers refund)
  - `Processing` → `Shipped` (admin/seller)
  - `Shipped` → `Delivered` (admin/system)
  - `Delivered` → `RefundRequested` (customer)
  - `RefundRequested` → `Refunded` (admin)
- [ ] Invalid transitions return `ErrInvalidOrderStatusTransition`

### Idempotency
- [ ] Extract `X-Idempotency-Key` from header (required on POST /orders)
- [ ] Check Redis for `idempotency:{key}` → if exists, return cached response
- [ ] On successful creation: store `idempotency:{key}` → `{order_id, status_code, response_body}` (TTL 24h)
- [ ] On duplicate: return 200 with cached body (not 201)

### Handler Layer (`handler.go`)
- [ ] `POST /api/v1/orders` — require Customer, extract idempotency key, call `CreateOrder`, return 201
- [ ] `GET /api/v1/orders` — require Customer, parse filter params, call `ListOrders`
- [ ] `GET /api/v1/orders/:id` — require Customer, verify ownership, call `GetOrder`
- [ ] `PATCH /api/v1/orders/:id/cancel` — require Customer/Admin, call `CancelOrder`
- [ ] `GET /api/v1/admin/orders` — require Admin/Seller, call `ListAllOrders`
- [ ] `PATCH /api/v1/admin/orders/:id/status` — require Admin/Seller, call `UpdateOrderStatus`
- [ ] `POST /api/v1/orders/:id/refund` — require Customer, call `RequestRefund`

### Repository Layer (`repository/`)
- [ ] `db/queries/order.sql`:
  - [ ] `CreateOrder` — INSERT into orders
  - [ ] `CreateOrderItems` — batch INSERT into order_items
  - [ ] `GetOrderByID` — SELECT with JOIN order_items, order_events
  - [ ] `GetOrderByIdempotencyKey` — SELECT for idempotency check
  - [ ] `ListOrdersByUser` — SELECT with cursor pagination, filtered by user_id
  - [ ] `ListAllOrders` — SELECT with filters (status, date range), cursor pagination
  - [ ] `UpdateOrderStatus` — UPDATE status, updated_at
  - [ ] `CreateOrderEvent` — INSERT into order_events (immutable audit log)
  - [ ] `GetOrderEvents` — SELECT ordered by created_at
  - [ ] `GetOrderItems` — SELECT for order
- [ ] Run `sqlc generate`

### Tests
- [ ] Unit tests:
  - [ ] CreateOrder: success, empty cart, invalid address, stock insufficient, idempotent duplicate
  - [ ] FSM: all valid transitions, all invalid transitions
  - [ ] CancelOrder: from PendingPayment, from Paid (triggers refund), from Shipped (rejected)
  - [ ] RequestRefund: from Delivered, from Processing (rejected)
  - [ ] Totals computation: subtotal, discount, tax, total
- [ ] Integration tests:
  - [ ] Full order lifecycle with real DB
  - [ ] Concurrent order creation with same idempotency key
  - [ ] Order creation within DB transaction (rollback on failure)

---

## internal/payment — Stripe/Paystack Integration, Webhooks, Refunds, Idempotency

### Standards & Conventions
- **Gin:** Webhook handler reads raw body for signature verification (NOT JSON-bound).
- **Provider abstraction:** `PaymentProvider` interface allows swapping gateways without changing business logic.
- **Idempotency:** All Stripe API calls include `stripe-idempotency-key` = `{order_id}:{attempt_number}`.
- **PCI compliance:** Card data never touches NimCart servers. Stripe.js handles tokenization client-side.
- **Money:** All amounts in integer cents via `pkg/money.Money`. Currency codes ISO 4217.
- **Events:** Publish `PaymentConfirmed`, `PaymentFailed`, `RefundProcessed` to event bus.

### Types & DTOs
- [ ] `Payment` — `ID`, `OrderID`, `Gateway`, `GatewayPaymentID`, `Amount` (money.Money), `Currency`, `Status`, `Metadata` (JSONB), `CreatedAt`
- [ ] `PaymentStatus` — enum: `Pending`, `Succeeded`, `Failed`, `Refunded`, `PartiallyRefunded`
- [ ] `PaymentProvider` interface:
  - [ ] `CreatePaymentIntent(ctx, amount, currency, orderID, idempotencyKey) (*PaymentIntent, error)`
  - [ ] `ConfirmPayment(ctx, paymentIntentID) error`
  - [ ] `Refund(ctx, gatewayPaymentID, amount, reason, idempotencyKey) (*RefundResult, error)`
  - [ ] `ConstructWebhookEvent(payload, signature, secret) (*WebhookEvent, error)`
- [ ] `PaymentIntent` — `ID`, `ClientSecret`, `Amount`, `Currency`, `Status`
- [ ] `WebhookEvent` — `Type`, `PaymentIntentID`, `Metadata`
- [ ] `StripeProvider` — implements `PaymentProvider` using `stripe-go` SDK
- [ ] `PaystackProvider` — implements `PaymentProvider` using Paystack API

### Service Layer (`service.go`)
- [ ] `CreatePaymentIntent(ctx, orderID, amount, currency) (*PaymentIntent, error)`:
  - [ ] Validate order exists and status is `PendingPayment`
  - [ ] Check for existing payment record (idempotency)
  - [ ] Call `provider.CreatePaymentIntent()` with idempotency key
  - [ ] Create `payments` row with status `Pending`
  - [ ] Return client secret to frontend
- [ ] `HandleWebhook(ctx, event WebhookEvent) error`:
  - [ ] Switch on event type:
    - [ ] `payment_intent.succeeded` → update payment status to `Succeeded`, publish `PaymentConfirmed`, enqueue order status update to `Paid`
    - [ ] `payment_intent.payment_failed` → update payment status to `Failed`, publish `PaymentFailed`, enqueue failure notification
  - [ ] All webhook processing via River job for durability (retry on failure)
- [ ] `Refund(ctx, orderID, amount, reason) (*Payment, error)`:
  - [ ] Validate order status allows refund (`Paid`, `Processing`, `Delivered`)
  - [ ] Validate refund amount <= original charge
  - [ ] Call `provider.Refund()` with idempotency key
  - [ ] Update payment status to `Refunded` or `PartiallyRefunded`
  - [ ] Publish `RefundProcessed` event
  - [ ] Enqueue order status update to `Refunded`
- [ ] `GetPaymentByOrderID(ctx, orderID) (*Payment, error)`

### Stripe Provider (`stripe_provider.go`)
- [ ] Initialize Stripe client with secret key from config/Vault
- [ ] `CreatePaymentIntent` — call `stripe.PaymentIntent.New()` with amount, currency, metadata
- [ ] `ConfirmPayment` — verify PaymentIntent status
- [ ] `Refund` — call `stripe.Refund.New()` with payment_intent, amount
- [ ] `ConstructWebhookEvent` — call `webhook.ConstructEvent()` with `Stripe-Signature` header validation
- [ ] Set webhook signing secret from config

### Paystack Provider (`paystack_provider.go`)
- [ ] Initialize Paystack client with secret key
- [ ] `CreatePaymentIntent` — call Paystack Initialize Transaction API
- [ ] `Refund` — call Paystack Refund API
- [ ] `ConstructWebhookEvent` — verify HMAC SHA512 signature on webhook payload

### Webhook Handler (`handler.go`)
- [ ] `POST /api/v1/webhooks/stripe`:
  - [ ] Read raw request body (NOT `c.ShouldBindJSON`)
  - [ ] Get `Stripe-Signature` header
  - [ ] Call `provider.ConstructWebhookEvent(body, signature, secret)`
  - [ ] If signature invalid: return 400
  - [ ] Enqueue webhook processing as River job (durable, retryable)
  - [ ] Return 200 immediately (Stripe expects quick 200)
- [ ] `POST /api/v1/webhooks/paystack` — same pattern with HMAC verification

### Payment Flow Handler
- [ ] `POST /api/v1/payments/intent` — require Customer, create payment intent for order, return client secret
- [ ] `GET /api/v1/payments/:orderID` — require Customer, return payment status

### Repository Layer (`repository/`)
- [ ] `db/queries/payment.sql`:
  - [ ] `CreatePayment` — INSERT into payments
  - [ ] `GetPaymentByOrderID` — SELECT by order_id
  - [ ] `GetPaymentByGatewayID` — SELECT by gateway_payment_id (for webhook lookup)
  - [ ] `UpdatePaymentStatus` — UPDATE status, metadata
  - [ ] `UpdatePaymentGatewayID` — UPDATE gateway_payment_id after intent creation
- [ ] Run `sqlc generate`

### Tests
- [ ] Unit tests:
  - [ ] CreatePaymentIntent: success, order not found, duplicate intent
  - [ ] HandleWebhook: payment succeeded, payment failed, invalid signature
  - [ ] Refund: success, amount exceeds original, invalid order status
- [ ] Integration tests:
  - [ ] Stripe webhook with mocked signature verification
  - [ ] Full payment flow: create intent → simulate webhook → order status update
  - [ ] Refund flow: full refund, partial refund
- [ ] Provider tests:
  - [ ] Stripe provider with mocked HTTP client
  - [ ] Paystack provider with mocked HTTP client

### Security Checklist
- [ ] Stripe secret key from Vault/env, never in source
- [ ] Webhook signature verified before any processing
- [ ] Idempotency keys on all gateway API calls
- [ ] No card data stored server-side
- [ ] Webhook endpoint rate-limited and behind signature check only (no JWT)

---

## internal/inventory — Stock Levels, Two-Phase Reservations, Low-Stock Alerts

### Standards & Conventions
- **Two-phase reservation model:** Soft reservations (Redis, TTL 30 min) → Hard reservations (PostgreSQL) on order creation.
- **Available stock formula:** `available = physical_stock - reserved_hard`
- **Gin:** Thin handlers. Admin/Seller role required for all admin endpoints.
- **Events:** Publish `InventoryDepleted`, `InventoryLowStock` to event bus.
- **Concurrency:** Use PostgreSQL `SELECT ... FOR UPDATE` on inventory rows during reservation to prevent overselling.

### Types & DTOs
- [ ] `Inventory` — `ID`, `VariantID`, `PhysicalStock`, `ReservedHard`, `Available` (computed), `ReorderThreshold`, `UpdatedAt`
- [ ] `SoftReservation` — `VariantID`, `Quantity`, `SessionID`, `ExpiresAt` (stored in Redis)
- [ ] `AdjustStockRequest` — `physical_stock` (required, min 0), `reason` (required, min 5 chars)
- [ ] `InventoryFilter` — `low_stock_only` (bool), `variant_id`, `cursor`, `limit`
- [ ] `ReservationRequest` — `variant_id`, `quantity`, `session_id`

### Service Layer (`service.go`)
- [ ] `GetInventory(ctx, variantID) (*Inventory, error)` — fetch stock levels, compute available
- [ ] `ListInventory(ctx, filter) (*PaginatedInventory, error)` — list all variants with stock levels
- [ ] `GetLowStock(ctx, threshold) ([]Inventory, error)` — variants where `available <= reorder_threshold`
- [ ] `AdjustStock(ctx, variantID, req, adminID) (*Inventory, error)`:
  - [ ] Update `physical_stock`
  - [ ] Create audit log entry (who, what, when, why)
  - [ ] Check if now below reorder threshold → publish `InventoryLowStock`
- [ ] `CreateSoftReservation(ctx, req) error`:
  - [ ] Check `available >= quantity` (read from DB)
  - [ ] Store in Redis: `reservation:{session_id}:{variant_id}` = quantity, TTL 30 min
  - [ ] Use Redis WATCH/MULTI for atomicity
- [ ] `ReleaseSoftReservation(ctx, sessionID, variantID) error`:
  - [ ] DEL Redis key `reservation:{session_id}:{variant_id}`
- [ ] `ReleaseAllSoftReservations(ctx, sessionID) error`:
  - [ ] SCAN and DEL all `reservation:{session_id}:*` keys
- [ ] `ConvertToHardReservation(ctx, variantID, quantity, orderID) error`:
  - [ ] Begin DB transaction
  - [ ] `SELECT ... FOR UPDATE` on inventory row
  - [ ] Validate `available >= quantity` (re-check under lock)
  - [ ] Increment `reserved_hard` by quantity
  - [ ] Release corresponding soft reservation from Redis
  - [ ] Commit transaction
  - [ ] If `available == 0` after: publish `InventoryDepleted`
- [ ] `ReleaseHardReservation(ctx, variantID, quantity) error`:
  - [ ] Decrement `reserved_hard` by quantity
  - [ ] Check if stock now available again
- [ ] `DecrementPhysicalStock(ctx, variantID, quantity) error`:
  - [ ] Called on fulfilment dispatch (item shipped)
  - [ ] Decrement `physical_stock` and `reserved_hard`
  - [ ] Create audit log entry

### Redis Operations
- [ ] `setSoftReservation(sessionID, variantID, quantity)` — SET with TTL 30 min
- [ ] `getSoftReservation(sessionID, variantID)` — GET
- [ ] `releaseSoftReservation(sessionID, variantID)` — DEL
- [ ] `getAllSoftReservations(sessionID)` — SCAN pattern `reservation:{session_id}:*`
- [ ] `getReservedSoft(variantID)` — count all soft reservations for a variant (for available calculation)

### Handler Layer (`handler.go`)
- [ ] `GET /api/v1/admin/inventory` — require Admin/Seller, parse filter, call `ListInventory`
- [ ] `PATCH /api/v1/admin/inventory/:variantId` — require Admin/Seller, bind+validate `AdjustStockRequest`, call `AdjustStock`
- [ ] `GET /api/v1/admin/inventory/low-stock` — require Admin/Seller, call `GetLowStock`

### Repository Layer (`repository/`)
- [ ] `db/queries/inventory.sql`:
  - [ ] `GetInventoryByVariantID` — SELECT single row
  - [ ] `ListInventory` — SELECT all with pagination
  - [ ] `GetLowStockInventory` — SELECT WHERE `(physical_stock - reserved_hard) <= reorder_threshold`
  - [ ] `UpdatePhysicalStock` — UPDATE physical_stock with audit log
  - [ ] `IncrementHardReservation` — UPDATE reserved_hard = reserved_hard + $1 (with FOR UPDATE lock)
  - [ ] `DecrementHardReservation` — UPDATE reserved_hard = reserved_hard - $1 (CHECK >= 0)
  - [ ] `DecrementPhysicalAndHard` — UPDATE both (on fulfilment)
  - [ ] `LockInventoryRow` — SELECT ... FOR UPDATE
  - [ ] `CreateInventoryAuditLog` — INSERT who, what, when, why
  - [ ] `CreateInventory` — INSERT initial stock for new variant
- [ ] Run `sqlc generate`

### Tests
- [ ] Unit tests:
  - [ ] CreateSoftReservation: stock available, insufficient stock, concurrent reservations
  - [ ] ConvertToHardReservation: success, stock depleted between soft and hard, transaction rollback
  - [ ] ReleaseHardReservation: success, over-release prevention
  - [ ] AdjustStock: increase, decrease, audit log creation
  - [ ] LowStock: threshold detection
- [ ] Integration tests:
  - [ ] Full reservation lifecycle: soft → hard → decrement
  - [ ] Concurrent reservation attempts on same variant (oversell prevention)
  - [ ] Transaction rollback on hard reservation failure

### Performance
- [ ] BTREE unique index on `variant_id` for lock-free reservation updates
- [ ] `SELECT ... FOR UPDATE` only during hard reservation conversion (not on reads)
- [ ] Redis soft reservations prevent DB reads on every add-to-cart
- [ ] Low-stock check cached in Redis (TTL 5 min) to avoid repeated queries

---

## internal/user — Profile, Addresses, Wishlist

### Standards & Conventions
- **Gin:** Thin handlers. All endpoints require authenticated Customer role.
- **Ownership:** Users can only access/modify their own profile, addresses, and wishlist. Enforced in service layer.
- **Validation:** `go-playground/validator/v10` on all DTOs. Address fields validated for completeness.
- **PII:** Email and phone encrypted at rest using PostgreSQL `pgcrypto` for elevated sensitivity fields.

### Types & DTOs
- [ ] `UserProfile` — `ID`, `Email`, `Name`, `AvatarURL`, `Phone`, `Role`, `Status`, `EmailVerifiedAt`, `CreatedAt`
- [ ] `UpdateProfileRequest` — `name` (optional, min 2), `avatar_url` (optional, URL), `phone` (optional, E.164 format)
- [ ] `Address` — `ID`, `UserID`, `FullName`, `Line1`, `Line2`, `City`, `State`, `Country`, `PostalCode`, `IsDefault`, `CreatedAt`
- [ ] `CreateAddressRequest` — `full_name` (required), `line1` (required), `line2` (optional), `city` (required), `state` (required), `country` (required, ISO 3166-1 alpha-2), `postal_code` (required), `is_default` (optional bool)
- [ ] `UpdateAddressRequest` — all fields optional
- [ ] `WishlistItem` — `ProductID`, `ProductName`, `Slug`, `BasePrice`, `ImageURL`, `AddedAt`
- [ ] `AddWishlistRequest` — `product_id` (required, UUID)

### Service Layer (`service.go`)
- [ ] `GetProfile(ctx, userID) (*UserProfile, error)` — fetch user by ID
- [ ] `UpdateProfile(ctx, userID, req) (*UserProfile, error)` — update name, avatar, phone
- [ ] `ListAddresses(ctx, userID) ([]Address, error)` — fetch all addresses for user
- [ ] `CreateAddress(ctx, userID, req) (*Address, error)`:
  - [ ] If `is_default = true`: unset previous default address
  - [ ] Create new address
- [ ] `UpdateAddress(ctx, userID, addressID, req) (*Address, error)`:
  - [ ] Verify address belongs to user
  - [ ] If changing `is_default` to true: unset previous default
  - [ ] Update fields
- [ ] `DeleteAddress(ctx, userID, addressID) error`:
  - [ ] Verify ownership
  - [ ] If deleting default: promote next address as default
  - [ ] Hard delete
- [ ] `GetWishlist(ctx, userID) ([]WishlistItem, error)` — fetch with product details (JOIN products)
- [ ] `AddToWishlist(ctx, userID, req) error`:
  - [ ] Validate product exists and is active
  - [ ] Check not already in wishlist (idempotent — no error if duplicate)
  - [ ] INSERT wishlist entry
- [ ] `RemoveFromWishlist(ctx, userID, productID) error`:
  - [ ] Verify in wishlist
  - [ ] DELETE

### Handler Layer (`handler.go`)
- [ ] `GET /api/v1/users/profile` — require auth, call `GetProfile`
- [ ] `PATCH /api/v1/users/profile` — require auth, bind+validate, call `UpdateProfile`
- [ ] `GET /api/v1/users/addresses` — require auth, call `ListAddresses`
- [ ] `POST /api/v1/users/addresses` — require auth, bind+validate, call `CreateAddress`, return 201
- [ ] `PUT /api/v1/users/addresses/:id` — require auth, verify ownership, call `UpdateAddress`
- [ ] `DELETE /api/v1/users/addresses/:id` — require auth, verify ownership, call `DeleteAddress`, return 204
- [ ] `GET /api/v1/users/wishlist` — require auth, call `GetWishlist`
- [ ] `POST /api/v1/users/wishlist` — require auth, bind+validate, call `AddToWishlist`, return 201
- [ ] `DELETE /api/v1/users/wishlist/:productId` — require auth, call `RemoveFromWishlist`, return 204

### Repository Layer (`repository/`)
- [ ] `db/queries/user.sql`:
  - [ ] `GetUserByID` — SELECT full profile
  - [ ] `UpdateUserProfile` — UPDATE name, avatar_url, phone
  - [ ] `ListAddresses` — SELECT by user_id, ordered by is_default DESC, created_at
  - [ ] `CreateAddress` — INSERT
  - [ ] `UpdateAddress` — UPDATE fields
  - [ ] `DeleteAddress` — DELETE
  - [ ] `UnsetDefaultAddress` — UPDATE is_default = false WHERE user_id = $1
  - [ ] `GetAddressByID` — SELECT for ownership check
  - [ ] `GetWishlist` — SELECT with JOIN products for details
  - [ ] `AddToWishlist` — INSERT (ON CONFLICT DO NOTHING for idempotency)
  - [ ] `RemoveFromWishlist` — DELETE
  - [ ] `IsInWishlist` — SELECT EXISTS for duplicate check
- [ ] Run `sqlc generate`

### Tests
- [ ] Unit tests:
  - [ ] UpdateProfile: success, empty fields
  - [ ] CreateAddress: with default, without default
  - [ ] DeleteAddress: default address (promotes next), non-default
  - [ ] AddToWishlist: success, duplicate (idempotent), product not found
  - [ ] RemoveFromWishlist: success, not in wishlist
- [ ] Integration tests:
  - [ ] Full address CRUD lifecycle
  - [ ] Wishlist add/remove with product validation
  - [ ] Ownership enforcement (user A cannot access user B's data)

---

## internal/notification — Email, In-App Events, Background Job Workers

### Standards & Conventions
- **River jobs:** All async notifications processed via River job queue for durability and retry.
- **Email:** SendGrid (primary) or SMTP via gomail (fallback). Templates stored as HTML files.
- **Event-driven:** Subscribes to event bus events: `OrderPlaced`, `PaymentConfirmed`, `OrderCancelled`, `PasswordResetRequested`, `ReviewSubmitted`.
- **No direct imports:** Other modules never import notification directly — they publish events.

### Types & DTOs
- [ ] `EmailType` — enum: `OrderConfirmation`, `OrderShipped`, `OrderCancelled`, `PasswordReset`, `WelcomeEmail`, `ReviewReminder`, `RefundProcessed`
- [ ] `EmailJob` — `To`, `Subject`, `Template`, `Data` (map for template variables)
- [ ] `InAppNotification` — `ID`, `UserID`, `Title`, `Message`, `Type`, `Read`, `CreatedAt`
- [ ] `NotificationEvent` — maps event bus events to notification actions

### Service Layer (`service.go`)
- [ ] `SendEmail(ctx, emailJob) error` — render template, send via SendGrid/SMTP
- [ ] `CreateInAppNotification(ctx, userID, title, message, type) error` — INSERT notification
- [ ] `GetUnreadNotifications(ctx, userID) ([]InAppNotification, error)`
- [ ] `MarkAsRead(ctx, userID, notificationID) error`

### Event Handlers (subscribed to event bus)
- [ ] `OnOrderPlaced(event)`:
  - [ ] Enqueue River job: send order confirmation email
  - [ ] Create in-app notification: "Your order #X has been placed"
- [ ] `OnPaymentConfirmed(event)`:
  - [ ] Enqueue River job: send payment receipt email
  - [ ] Create in-app notification: "Payment confirmed for order #X"
- [ ] `OnOrderCancelled(event)`:
  - [ ] Enqueue River job: send cancellation email
  - [ ] Create in-app notification: "Order #X has been cancelled"
- [ ] `OnPasswordResetRequested(event)`:
  - [ ] Enqueue River job: send password reset email with token link
- [ ] `OnReviewSubmitted(event)`:
  - [ ] Enqueue River job: notify admin of pending review moderation
- [ ] `OnOrderShipped(event)`:
  - [ ] Enqueue River job: send shipping confirmation with tracking number
- [ ] `OnRefundProcessed(event)`:
  - [ ] Enqueue River job: send refund confirmation email

### River Job Workers
- [ ] `EmailWorker` — processes `email` job kind:
  - [ ] Render HTML template from `templates/` directory
  - [ ] Send via SendGrid API (or SMTP fallback)
  - [ ] MaxAttempts: 3, with exponential backoff
  - [ ] On permanent failure: dead-letter to `failed_emails` table
- [ ] `InAppNotificationWorker` — processes `notification` job kind:
  - [ ] Create in-app notification record
  - [ ] MaxAttempts: 2

### Email Templates (`templates/`)
- [ ] `order_confirmation.html` — order summary, items, total, shipping address
- [ ] `payment_receipt.html` — payment details, amount, gateway reference
- [ ] `order_cancelled.html` — cancellation reason, refund timeline
- [ ] `password_reset.html` — reset link (token), expiry warning (1 hour)
- [ ] `welcome_email.html` — account created, getting started
- [ ] `order_shipped.html` — tracking number, estimated delivery
- [ ] `refund_processed.html` — refund amount, expected timeline
- [ ] `review_reminder.html` — prompt to review purchased product

### Handler Layer (`handler.go`)
- [ ] `GET /api/v1/notifications` — require auth, return unread notifications
- [ ] `PATCH /api/v1/notifications/:id/read` — require auth, mark as read
- [ ] `PATCH /api/v1/notifications/read-all` — require auth, mark all as read

### Repository Layer (`repository/`)
- [ ] `db/queries/notification.sql`:
  - [ ] `CreateNotification` — INSERT in-app notification
  - [ ] `GetUnreadNotifications` — SELECT WHERE read = false, ORDER BY created_at DESC
  - [ ] `MarkNotificationRead` — UPDATE read = true
  - [ ] `MarkAllNotificationsRead` — UPDATE read = true WHERE user_id = $1
  - [ ] `CreateFailedEmail` — INSERT into failed_emails for dead-letter
- [ ] Run `sqlc generate`

### Tests
- [ ] Unit tests:
  - [ ] Event handler mapping: correct job enqueued for each event type
  - [ ] Email rendering: template variables populated correctly
  - [ ] Retry logic: job retried on transient failure
- [ ] Integration tests:
  - [ ] Full flow: event published → job enqueued → email sent (mock SMTP)
  - [ ] Dead-letter: permanently failed email persisted to failed_emails

---

## internal/review — Product Ratings, Review Submission, Moderation

### Standards & Conventions
- **Gin:** Thin handlers. Public read endpoints, authenticated write endpoints.
- **Purchase verification:** Customer can only review products from orders with status `Delivered`.
- **One review per product per user:** Enforced via unique constraint `(product_id, user_id)`.
- **Moderation:** Reviews start as `Pending`, visible publicly only after admin approval (`Approved`).
- **Rating:** Integer 1-5. Average rating computed and cached per product.

### Types & DTOs
- [ ] `Review` — `ID`, `ProductID`, `UserID`, `OrderID`, `Rating` (1-5), `Title`, `Body`, `Status`, `ModeratedBy`, `ModeratorNotes`, `CreatedAt`, `UpdatedAt`
- [ ] `ReviewStatus` — enum: `Pending`, `Approved`, `Rejected`
- [ ] `CreateReviewRequest` — `rating` (required, min 1, max 5), `title` (required, min 3, max 100), `body` (required, min 10, max 2000)
- [ ] `ModerateReviewRequest` — `status` (required, oneof=approved rejected), `moderator_notes` (optional)
- [ ] `ReviewSummary` — `ProductID`, `AverageRating`, `TotalReviews`, `RatingDistribution` (map[int]int: count per star)
- [ ] `ReviewFilter` — `product_id`, `status` (admin), `cursor`, `limit`

### Service Layer (`service.go`)
- [ ] `ListApprovedReviews(ctx, productID, cursor, limit) (*PaginatedReviews, error)`:
  - [ ] Only return `Approved` reviews
  - [ ] Include reviewer name (JOIN users)
  - [ ] Cursor pagination
- [ ] `GetReviewSummary(ctx, productID) (*ReviewSummary, error)`:
  - [ ] Compute average rating, total count, distribution per star
  - [ ] Cache in Redis (TTL 10 min, invalidate on new approved review)
- [ ] `CreateReview(ctx, userID, productID, orderID, req) (*Review, error)`:
  - [ ] Validate order exists, belongs to user, status is `Delivered`
  - [ ] Validate order contains the product (variant → product)
  - [ ] Check no existing review for this product+user (unique constraint)
  - [ ] Create review with status `Pending`
  - [ ] Publish `ReviewSubmitted` event (triggers admin notification)
- [ ] `ModerateReview(ctx, adminID, reviewID, req) (*Review, error)`:
  - [ ] Update status to `Approved` or `Rejected`
  - [ ] Set `moderated_by` and `moderator_notes`
  - [ ] If approved: invalidate review summary cache
  - [ ] Publish `ReviewApproved` or `ReviewRejected` event
- [ ] `DeleteReview(ctx, adminID, reviewID) error`:
  - [ ] Hard delete (abuse removal)
  - [ ] Invalidate review summary cache
- [ ] `ListAllReviews(ctx, filter) (*PaginatedReviews, error)` — admin: all reviews with status filter

### Handler Layer (`handler.go`)
- [ ] `GET /api/v1/products/:id/reviews` — public, parse cursor/limit, call `ListApprovedReviews`
- [ ] `POST /api/v1/products/:id/reviews` — require Customer, extract order_id from body or query, bind+validate, call `CreateReview`, return 201
- [ ] `PATCH /api/v1/admin/reviews/:id` — require Admin, bind+validate `ModerateReviewRequest`, call `ModerateReview`
- [ ] `DELETE /api/v1/admin/reviews/:id` — require Admin, call `DeleteReview`, return 204
- [ ] `GET /api/v1/admin/reviews` — require Admin, parse filter, call `ListAllReviews`
- [ ] `GET /api/v1/products/:id/reviews/summary` — public, call `GetReviewSummary`

### Repository Layer (`repository/`)
- [ ] `db/queries/review.sql`:
  - [ ] `ListReviewsByProduct` — SELECT WHERE product_id AND status = 'approved', cursor pagination
  - [ ] `GetReviewSummary` — SELECT AVG(rating), COUNT(*), grouping by rating
  - [ ] `CreateReview` — INSERT with unique constraint (product_id, user_id)
  - [ ] `GetReviewByID` — SELECT single review
  - [ ] `UpdateReviewStatus` — UPDATE status, moderated_by, moderator_notes
  - [ ] `DeleteReview` — DELETE
  - [ ] `ListAllReviews` — SELECT with status filter, cursor pagination
  - [ ] `HasUserReviewedProduct` — SELECT EXISTS (product_id, user_id)
  - [ ] `GetUserDeliveredOrderForProduct` — SELECT from orders JOIN order_items WHERE status = 'delivered' (purchase verification)
- [ ] Run `sqlc generate`

### Tests
- [ ] Unit tests:
  - [ ] CreateReview: success, not purchased, already reviewed, invalid rating
  - [ ] ModerateReview: approve, reject, cache invalidation
  - [ ] DeleteReview: success, cache invalidation
  - [ ] GetReviewSummary: correct averages, empty reviews
- [ ] Integration tests:
  - [ ] Full review lifecycle: create → approve → visible in listing
  - [ ] Purchase verification: cannot review without delivered order
  - [ ] Unique constraint: second review from same user rejected
   - [ ] Cache invalidation on approval

---

## internal/analytics — Admin Dashboard Metrics, Revenue Charts, KPIs

### Standards & Conventions
- **Gin:** Thin handlers. All endpoints require Admin role.
- **Read-only:** Analytics module performs no mutations. Reads aggregated data from orders, payments, products.
- **Caching:** Analytics results cached in Redis (TTL 30 min) to avoid repeated expensive queries.
- **Date ranges:** All endpoints accept `period` param: `7d`, `30d`, `90d`, or custom `from`/`to` dates.

### Types & DTOs
- [ ] `AnalyticsSummary` — `TotalRevenue` (money.Money), `TotalOrders` (int64), `TotalCustomers` (int64), `AverageOrderValue` (money.Money), `ConversionRate` (float64, optional v1)
- [ ] `RevenuePoint` — `Date` (string), `Revenue` (money.Money), `OrderCount` (int64)
- [ ] `StatusDistribution` — `Status` (order status string), `Count` (int64)
- [ ] `TopProduct` — `ProductID`, `ProductName`, `Slug`, `Revenue` (money.Money), `UnitsSold` (int64)
- [ ] `CategoryBreakdown` — `CategoryID`, `CategoryName`, `Revenue` (money.Money), `Percentage` (float64)
- [ ] `AnalyticsFilter` — `period` (string: 7d/30d/90d/custom), `from` (string, date), `to` (string, date)

### Service Layer (`service.go`)
- [ ] `GetSummary(ctx, filter) (*AnalyticsSummary, error)`:
  - [ ] Aggregate total revenue, order count, customer count for period
  - [ ] Compute AOV = total_revenue / total_orders
  - [ ] Compute trend vs previous period of equal length
- [ ] `GetRevenueChart(ctx, filter) ([]RevenuePoint, error)`:
  - [ ] Group revenue by day/week/month depending on period length
  - [ ] Return ordered time series
- [ ] `GetOrdersByStatus(ctx, filter) ([]StatusDistribution, error)`:
  - [ ] COUNT grouped by order status
- [ ] `GetTopProducts(ctx, filter, limit) ([]TopProduct, error)`:
  - [ ] JOIN order_items + orders + products
  - [ ] SUM revenue per product, ORDER BY DESC, LIMIT
  - [ ] Default limit: 10
- [ ] `GetSalesByCategory(ctx, filter) ([]CategoryBreakdown, error)`:
  - [ ] JOIN order_items → product_variants → products → categories
  - [ ] SUM revenue per category, compute percentage of total

### Handler Layer (`handler.go`)
- [ ] `GET /api/v1/admin/analytics/summary` — require Admin, parse period param, call `GetSummary`
- [ ] `GET /api/v1/admin/analytics/revenue` — require Admin, parse period/from/to, call `GetRevenueChart`
- [ ] `GET /api/v1/admin/analytics/orders-by-status` — require Admin, call `GetOrdersByStatus`
- [ ] `GET /api/v1/admin/analytics/top-products` — require Admin, parse limit (default 10, max 50), call `GetTopProducts`
- [ ] `GET /api/v1/admin/analytics/sales-by-category` — require Admin, call `GetSalesByCategory`

### Repository Layer (`repository/`)
- [ ] `db/queries/analytics.sql`:
  - [ ] `GetRevenueSummary` — SELECT SUM(total), COUNT(*) FROM orders WHERE paid status, filtered by date range
  - [ ] `GetCustomerCount` — SELECT COUNT(*) FROM users WHERE created_at in range
  - [ ] `GetRevenueTimeSeries` — SELECT DATE(created_at), SUM(total), COUNT(*) GROUP BY date, ordered
  - [ ] `GetOrdersByStatus` — SELECT status, COUNT(*) GROUP BY status
  - [ ] `GetTopProducts` — SELECT product_id, product_name, SUM(oi.unit_price * oi.quantity) as revenue, SUM(oi.quantity) as sold, GROUP BY, ORDER BY DESC, LIMIT
  - [ ] `GetSalesByCategory` — SELECT category_id, category_name, SUM(revenue), GROUP BY, compute percentage
  - [ ] All queries accept date range parameters via `sqlc.narg()`
- [ ] Run `sqlc generate`

### Caching
- [ ] Cache all analytics responses in Redis with TTL 30 minutes
- [ ] Cache keys: `analytics:summary:{period}`, `analytics:revenue:{period}`, etc.
- [ ] Invalidate cache on report refresh (optional admin force-refresh param `?refresh=true`)

### Tests
- [ ] Unit tests:
  - [ ] GetSummary: correct aggregates, empty data, date range filtering
  - [ ] GetRevenueChart: correct grouping by day/week/month, sorted correctly
  - [ ] GetTopProducts: correct ordering, respects limit, empty results
  - [ ] GetSalesByCategory: correct percentages (sum to 100%), empty categories
- [ ] Integration tests:
  - [ ] Full analytics query with seeded test data
   - [ ] Cache hit/miss behavior
   - [ ] Date range boundary validation

---

## internal/coupon — Admin Coupon CRUD

### Standards & Conventions
- **Gin:** Thin handlers. All endpoints require Admin role.
- **sqlc:** All queries from `db/queries/coupon.sql`. Already shared with cart module's coupon validation.
- **Soft delete:** Coupons are soft-deleted via `deleted_at` — don't hard delete to preserve order history references.

### Types & DTOs
- [ ] `Coupon` — `ID`, `Code`, `Type` (percent/fixed), `Value`, `MinOrderAmount`, `MaxUses`, `UsedCount`, `ExpiresAt`, `CreatedAt`
- [ ] `CreateCouponRequest` — `code` (required, unique, uppercase alphanumeric), `type` (required, oneof=percent fixed), `value` (required, positive), `min_order_amount` (optional), `max_uses` (optional, 0=unlimited), `expires_at` (optional)
- [ ] `UpdateCouponRequest` — all fields optional (pointer types); `code` immutable after creation
- [ ] `CouponFilter` — `type`, `active` (bool), `cursor`, `limit`

### Service Layer (`service.go`)
- [ ] `ListCoupons(ctx, filter) (*PaginatedCoupons, error)` — cursor pagination
- [ ] `CreateCoupon(ctx, req) (*Coupon, error)` — validate code uniqueness (uppercase, no special chars)
- [ ] `UpdateCoupon(ctx, couponID, req) (*Coupon, error)` — code field is immutable
- [ ] `DeleteCoupon(ctx, couponID) error` — soft delete (set `deleted_at`)

### Handler Layer (`handler.go`)
- [ ] `GET /api/v1/admin/coupons` — require Admin, parse filter, call `ListCoupons`
- [ ] `POST /api/v1/admin/coupons` — require Admin, bind+validate, call `CreateCoupon`, return 201
- [ ] `PUT /api/v1/admin/coupons/:id` — require Admin, bind+validate, call `UpdateCoupon`
- [ ] `DELETE /api/v1/admin/coupons/:id` — require Admin, call `DeleteCoupon`, return 204

### Repository Layer (`repository/`)
- [ ] `db/queries/cart.sql` (shared with cart module for coupon queries):
  - [ ] `ListCoupons` — SELECT with type filter, `WHERE deleted_at IS NULL`, cursor pagination
  - [ ] `CreateCoupon` — INSERT
  - [ ] `UpdateCoupon` — UPDATE fields (except code and used_count)
  - [ ] `SoftDeleteCoupon` — UPDATE deleted_at = now()
  - [ ] `GetCouponByID` — SELECT by id
- [ ] Run `sqlc generate`

### Tests
- [ ] Unit tests:
  - [ ] CreateCoupon: success, duplicate code, invalid type
  - [ ] UpdateCoupon: success, code cannot change, not found
  - [ ] DeleteCoupon: success, soft delete verified
  - [ ] ListCoupons: active filter, type filter, pagination
