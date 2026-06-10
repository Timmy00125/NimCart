# NimCart — Product Requirements Document

**Version:** 1.0.0
**Status:** Draft – Awaiting Engineering Sign-off
**Date:** June 2026

| Field | Detail |
|---|---|
| Document Type | Product Requirements Document (PRD) |
| Product Name | NimCart |
| Version | 1.0.0 |
| Status | Draft |
| Backend | Go 1.25 · Gin · sqlc · pgx/v5 |
| Frontend | Angular 22 · SpartanUI · TailwindCSS |
| Database | PostgreSQL 16 · Atlas (migrations) |
| Architecture | Modular Monolith |
| Target Audience | SME Retailers, B2C Shoppers |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Goals & Non-Goals](#2-goals--non-goals)
3. [Stakeholders & User Personas](#3-stakeholders--user-personas)
4. [Technology Stack](#4-technology-stack)
5. [Architecture — Modular Monolith](#5-architecture--modular-monolith)
6. [Module Specifications](#6-module-specifications)
    - 6.1 Auth, 6.2 Catalog, 6.3 Cart, 6.4 Order, 6.5 Payment, 6.6 Inventory, 6.7 User, 6.8 Review, 6.9 Analytics, 6.10 Notification, 6.11 Coupon Management
7. [Database Schema](#7-database-schema)
8. [Frontend Specifications](#8-frontend-specifications)
9. [Non-Functional Requirements](#9-non-functional-requirements)
10. [Security Considerations](#10-security-considerations)
11. [Development Workflow](#11-development-workflow)
12. [Milestones & Delivery Plan](#12-milestones--delivery-plan)
13. [Open Questions & Risks](#13-open-questions--risks)
14. [Glossary](#14-glossary)

---

## 1. Executive Summary

NimCart is a full-stack eCommerce platform built as a modular monolith. It is designed to let small-to-medium retailers launch, manage, and scale an online storefront without adopting the operational complexity of microservices. By co-locating all business domains inside a single deployable binary while enforcing strict module boundaries, NimCart achieves the developer ergonomics of a monolith and the codebase maintainability of a service-oriented system.

**Key Design Principles:**

- **Modular Monolith** — all modules deploy together, but boundaries are enforced at the package level.
- **Type-safe database access** via sqlc (compiled queries from SQL, no runtime reflection).
- **Thin API layer** — business logic lives in domain services, not HTTP handlers.
- **Angular + SpartanUI** for an accessible, component-driven storefront.
- **Event-driven intra-module communication** through a lightweight internal event bus.

---

## 2. Goals & Non-Goals

### 2.1 Goals

- Deliver a production-ready eCommerce experience covering catalog, cart, checkout, payments, orders, and user accounts.
- Expose a clean REST API (v1) from a single Go binary for all frontend and third-party consumers.
- Maintain < 100 ms p95 API response time under moderate traffic (500 concurrent users).
- Achieve ≥ 80% unit/integration test coverage per module.
- Provide a seamless Angular SPA shopping experience with SSR-ready architecture.
- Support role-based access control (Admin, Seller, Customer, Guest).

### 2.2 Non-Goals (v1)

- No microservices decomposition — modules are NOT independently deployable in v1.
- No mobile-native apps (Android/iOS). The Angular app is responsive and serves mobile browsers.
- No multi-tenancy — a single seller organisation per deployment.
- No real-time bidding/auction functionality.
- No AI-powered product recommendation engine (planned for v2).

---

## 3. Stakeholders & User Personas

| Persona | Role | Primary Needs |
|---|---|---|
| Customer | B2C Shopper | Browse products, manage cart/wishlist, place & track orders, manage returns. |
| Admin | Platform Operator | Manage sellers, approve products, view platform analytics, configure system settings. |
| Seller / Store Manager | Merchant | CRUD product catalog, view orders, manage inventory, access sales reports. |
| Guest | Anonymous Visitor | Browse catalog, add to cart, sign up to checkout. |

---

## 4. Technology Stack

### 4.1 Backend

| Concern | Technology | Version | Rationale |
|---|---|---|---|
| Language | Go | 1.25 | Static typing, compiled binary, excellent concurrency primitives, single-artifact deployment. |
| HTTP Framework | Gin | 1.10+ | Minimal overhead, middleware chaining, widely adopted in production Go APIs. |
| DB Driver | pgx/v5 | 5.x | High-performance, context-aware PostgreSQL driver; native binary protocol. |
| Query Codegen | sqlc | 1.27+ | Generates fully type-safe Go from raw SQL. Eliminates runtime reflection; SQL remains the source of truth. |
| Migrations | Atlas (ariga.io) | Latest | Declarative schema migrations with auto-diff and CI integration. Replaces hand-written SQL migration files. |
| Validation | go-playground/validator | v10 | Struct-tag based request validation with i18n-ready error messages. |
| Auth | JWT (golang-jwt/jwt) | v5 | Stateless access + refresh token pair; RS256 signing. |
| Password Hashing | bcrypt (x/crypto) | stdlib | Industry-standard adaptive hashing for stored credentials. |
| Config | Viper | 1.x | Multi-source config (env vars, YAML, flags) with hot reload support. |
| Logging | zap (Uber) | 2.x | Structured JSON logging; zero-allocation hot path. |
| Tracing/Metrics | OpenTelemetry Go SDK | 1.x | Vendor-neutral observability; exportable to Tempo, Prometheus, Jaeger. |
| Testing | testify + gomock | Latest | Assertions, mocks, and table-driven test helpers. |
| Task Queue | River (riverqueue) | 0.x | Postgres-backed job queue — no extra broker (Redis/RabbitMQ) required in v1. |
| File Storage | AWS S3 SDK (minio-compatible) | v2 | Object storage for product images; uses presigned URLs. |
| Email | SendGrid / SMTP via gomail | Latest | Transactional email for order confirmations, password resets. |

### 4.2 Frontend

| Concern | Technology | Version | Rationale |
|---|---|---|---|
| Framework | Angular | 22 | Signals-based reactivity, standalone components, built-in SSR support via Angular Universal. |
| UI Component Library | SpartanUI (spartan.ng) | Latest | Headless + styled Angular component library built on Radix UI primitives. Accessible, themeable, ARIA-compliant. |
| Styling | TailwindCSS | 3.x | Utility-first CSS; co-located with SpartanUI tokens for consistent design system. |
| State Management | NgRx (Signal Store) | 18+ | Lightweight reactive state per feature; fine-grained signal-based reactivity. |
| HTTP Client | Angular HttpClient + interceptors | Built-in | Centralised auth token injection, error normalisation, retry logic. |
| Forms | Angular Reactive Forms | Built-in | Typed forms with composable validators. |
| Routing | Angular Router (lazy modules) | Built-in | Feature-module lazy loading to minimise initial bundle. |
| Testing | Jest + Angular Testing Library | Latest | Component and integration tests with user-centric queries. |
| E2E Testing | Playwright | Latest | Cross-browser end-to-end tests for critical purchase flows. |
| Build | Angular CLI + esbuild | Built-in | Fast incremental builds; production tree-shaking. |

### 4.3 Database & Infrastructure

| Concern | Technology | Version | Rationale |
|---|---|---|---|
| Database | PostgreSQL | 16+ | ACID, JSONB, full-text search, row-level security, partial indexes — handles all v1 persistence needs. |
| Connection Pooling | PgBouncer | 1.22+ | Transaction-mode pooling between Go instances and Postgres; prevents connection exhaustion. |
| Caching | Redis (go-redis) | 7.x | Session cache, rate-limit counters, idempotency keys, hot product caching. |
| Object Storage | MinIO (dev) / AWS S3 (prod) | Latest | S3-compatible product image and asset storage. |
| Container Runtime | Docker + Docker Compose | Latest | Local development parity; single-command stack startup. |
| CI/CD | GitHub Actions | N/A | Lint, test, build, migrate, and deploy pipeline. |
| Secrets | HashiCorp Vault / env injection | Latest | Secrets never in source control; rotated via Vault dynamic credentials. |

---

## 5. Architecture — Modular Monolith

### 5.1 Philosophy

NimCart is a single deployable Go binary whose internal structure mirrors a service-oriented architecture. Each business domain is a Go module package with a clearly defined public interface (API surface), internal business logic, and its own sqlc-generated repository layer. Cross-module communication is forbidden via direct function calls to internal packages; all inter-module interaction goes through either the public API types or the internal event bus.

### 5.2 Repository Structure

```
nimcart/
├── cmd/
│   └── server/             # main.go — wires modules, starts Gin router
├── internal/
│   ├── auth/               # JWT issuance, RBAC middleware
│   │   ├── handler.go
│   │   ├── service.go
│   │   └── repository/     # sqlc-generated queries
│   ├── catalog/            # Products, categories, search
│   ├── cart/               # Cart & wishlist
│   ├── order/              # Order lifecycle FSM
│   ├── payment/            # Payment gateway integration
│   ├── inventory/          # Stock levels, reservations
│   ├── user/               # User profile, addresses
│   ├── notification/       # Email + in-app events
│   ├── review/             # Product ratings & reviews
│   ├── analytics/          # Admin dashboard metrics & KPIs
│   └── platform/           # Shared: DB pool, logger, config, event bus
├── db/
│   ├── migrations/         # Atlas HCL migration files
│   ├── queries/            # Raw .sql files consumed by sqlc
│   └── schema/             # Atlas schema definitions
├── pkg/
│   ├── apierr/             # Typed API error responses
│   ├── pagination/         # Cursor-based pagination helpers
│   └── money/              # Monetary value type (no float arithmetic)
├── frontend/               # Angular workspace (standard Angular CLI)
├── sqlc.yaml
├── atlas.hcl
└── docker-compose.yml
```

### 5.3 Module Boundary Rules

- Each module exposes only types in its top-level package (e.g., `catalog.Product`, `catalog.Service`).
- Internal sub-packages (`catalog/internal/...`) are unreachable by other modules at compile time.
- No module may import another module's repository or DB types — only the module's exported service interface.
- Cross-module state changes are driven by events published to the internal event bus, consumed asynchronously.
- Boundary violations are caught at CI time via `depguard` rules in golangci-lint.

### 5.4 Internal Event Bus

A lightweight, in-process, typed event bus (using Go generics) handles domain events such as `OrderPlaced`, `PaymentConfirmed`, and `InventoryDepleted`. Event handlers run in separate goroutines. For durability, events that must not be lost (e.g., `PaymentConfirmed` → trigger fulfilment) are persisted to a River job queue table in PostgreSQL before the HTTP response is returned.

### 5.5 API Design

- All endpoints are prefixed `/api/v1/`.
- JSON request/response bodies with `snake_case` field names.
- Errors follow RFC 7807 (Problem Details): `{ type, title, status, detail, instance }`.
- Pagination uses cursor-based (keyset) pagination via an opaque `cursor` field.
- All mutating endpoints (POST/PUT/PATCH/DELETE) require idempotency keys (`X-Idempotency-Key` header) for payment and order flows.

---

## 6. Module Specifications

### 6.1 Auth Module

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/auth/register` | POST | Public | Register a new customer account. Hashes password with bcrypt cost 12. |
| `/api/v1/auth/login` | POST | Public | Issue access (15 min) + refresh (7 days) JWT pair. |
| `/api/v1/auth/refresh` | POST | Refresh Token | Rotate refresh token; issue new access token. |
| `/api/v1/auth/logout` | POST | Bearer | Revoke refresh token (stored in Redis blacklist). |
| `/api/v1/auth/forgot-password` | POST | Public | Send password reset link via email (token valid 1 hour). |
| `/api/v1/auth/reset-password` | POST | Public | Validate token, update password hash. |
| `/api/v1/auth/me` | GET | Bearer | Return authenticated user's profile. |

#### Roles & Permissions

| Role | Permissions |
|---|---|
| Guest | Browse catalog, view products, add to session cart. |
| Customer | All Guest + place orders, manage own profile/addresses, write reviews. |
| Seller | All Customer + CRUD own products, manage own inventory, view own orders. |
| Admin | All Seller + platform-wide management, approve/reject sellers, access analytics. |

---

### 6.2 Catalog Module

Products have a status FSM: `Draft → PendingReview → Active → Archived`.

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/products` | GET | Public | List active products. Supports filter (category, price range, tags), sort, and cursor pagination. |
| `/api/v1/products/:slug` | GET | Public | Fetch single product by slug including variants and images. |
| `/api/v1/products/search` | GET | Public | Full-text search using PostgreSQL `tsvector` + `plainto_tsquery`. |
| `/api/v1/products` | POST | Seller/Admin | Create product (creates in Draft status). |
| `/api/v1/products/:id` | PUT | Seller/Admin | Update product fields. |
| `/api/v1/products/:id/status` | PATCH | Admin | Transition product status. |
| `/api/v1/products/:id` | DELETE | Admin | Soft delete product (sets `archived_at`). |
| `/api/v1/categories` | GET | Public | Fetch category tree. |
| `/api/v1/categories` | POST | Admin | Create category. |
| `/api/v1/products/:id/images` | POST | Seller/Admin | Upload product images; returns presigned S3 URL. |

#### Product Variants

Products support variants (e.g., Size: S/M/L, Colour: Red/Blue). Each variant has its own SKU, price override, and inventory count. Variant options are stored as `JSONB` for flexibility.

---

### 6.3 Cart Module

Session cart for guests (stored in Redis with a 30-day TTL) and a persistent cart for authenticated users (stored in PostgreSQL). On login, the guest cart is merged into the user cart.

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/cart` | GET | Optional | Retrieve cart (session or user). Validates stock in real time. |
| `/api/v1/cart/items` | POST | Optional | Add item to cart. Checks availability and reserves soft stock. |
| `/api/v1/cart/items/:itemId` | PATCH | Optional | Update quantity. Validates against available stock. |
| `/api/v1/cart/items/:itemId` | DELETE | Optional | Remove item. Releases soft stock reservation. |
| `/api/v1/cart` | DELETE | Optional | Clear entire cart. |
| `/api/v1/cart/apply-coupon` | POST | Customer | Apply discount coupon code. Returns updated cart totals. |
| `/api/v1/cart/merge` | POST | Customer | Merge guest cart into authenticated user cart (called post-login). |

---

### 6.4 Order Module

Orders are state machines with the following states:

- `PENDING_PAYMENT` — order created, awaiting payment confirmation.
- `PAID` — payment confirmed by payment webhook.
- `PROCESSING` — warehouse has picked up the order.
- `SHIPPED` — tracking number assigned, in transit.
- `DELIVERED` — confirmed delivery.
- `CANCELLED` — cancelled before shipment (triggers inventory release + refund).
- `REFUND_REQUESTED` — customer requested refund post-delivery.
- `REFUNDED` — refund processed.

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/orders` | POST | Customer | Create order from cart. Locks inventory, creates payment intent. |
| `/api/v1/orders` | GET | Customer | List own orders with pagination. |
| `/api/v1/orders/:id` | GET | Customer | Fetch order detail including line items, shipping, timeline. |
| `/api/v1/orders/:id/cancel` | PATCH | Customer/Admin | Cancel order if in `PENDING_PAYMENT` or `PAID` state. |
| `/api/v1/admin/orders` | GET | Admin/Seller | List all orders with advanced filters. |
| `/api/v1/admin/orders/:id/status` | PATCH | Admin/Seller | Transition order status with optional notes. |
| `/api/v1/orders/:id/refund` | POST | Customer | Request refund with reason. |

---

### 6.5 Payment Module

Abstracts payment gateway interactions behind an internal `PaymentProvider` interface, allowing gateway swaps without changing business logic. v1 targets Stripe as the primary gateway with Paystack as the alternative for NGN markets.

| Concern | Detail |
|---|---|
| Gateway | Stripe (primary) / Paystack (NGN fallback) |
| Flow | Create `PaymentIntent` server-side → confirm client-side → webhook → update order status. |
| Idempotency | All Stripe calls include a `stripe-idempotency-key` derived from `order_id + attempt_number`. |
| Webhooks | `POST /api/v1/webhooks/stripe` — validates `Stripe-Signature` header before processing. |
| Refunds | Initiated via payment module; amount capped at original charge. |
| Currencies | USD (primary), NGN (Paystack), EUR. |
| PCI Compliance | Card data never touches NimCart servers; tokenised via Stripe.js / Paystack inline. |

---

### 6.6 Inventory Module

Manages stock levels per product variant with a two-phase reservation model to prevent overselling.

- **Available Stock** = Physical Stock − Hard Reservations.
- **On add-to-cart:** create a soft reservation (Redis TTL 30 min).
- **On order creation:** convert soft reservation to hard reservation in DB.
- **On order cancellation/payment timeout:** release hard reservation.
- **On fulfilment dispatch:** decrement physical stock.

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/admin/inventory` | GET | Admin/Seller | List inventory levels with low-stock alerts. |
| `/api/v1/admin/inventory/:variantId` | PATCH | Admin/Seller | Manually adjust stock level with audit log. |
| `/api/v1/admin/inventory/low-stock` | GET | Admin/Seller | Return variants below reorder threshold. |

---

### 6.7 User Module

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/users/profile` | GET | Customer | Get own profile. |
| `/api/v1/users/profile` | PATCH | Customer | Update name, avatar, phone. |
| `/api/v1/users/addresses` | GET | Customer | List saved addresses. |
| `/api/v1/users/addresses` | POST | Customer | Add new address. |
| `/api/v1/users/addresses/:id` | PUT | Customer | Update address. |
| `/api/v1/users/addresses/:id` | DELETE | Customer | Delete address. |
| `/api/v1/users/wishlist` | GET | Customer | Fetch wishlist items. |
| `/api/v1/users/wishlist` | POST | Customer | Add product to wishlist. |
| `/api/v1/users/wishlist/:productId` | DELETE | Customer | Remove from wishlist. |

---

### 6.8 Review Module

Customers can submit one review per purchased product after order status reaches `DELIVERED`. Reviews are moderated — they appear in `PENDING` state until an admin approves or rejects them.

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/products/:id/reviews` | GET | Public | List approved reviews for a product (paginated). |
| `/api/v1/products/:id/reviews/summary` | GET | Public | Get average rating, total count, star distribution. |
| `/api/v1/products/:id/reviews` | POST | Customer | Submit review. Validates purchase eligibility. |
| `/api/v1/admin/reviews` | GET | Admin | List all reviews with status filter. |
| `/api/v1/admin/reviews/:id` | PATCH | Admin | Approve or reject a review with optional moderator notes. |
| `/api/v1/admin/reviews/:id` | DELETE | Admin | Hard delete review (abuse removal). |

---

### 6.9 Analytics Module

Provides aggregated metrics for the admin dashboard. All endpoints require Admin authentication.

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/admin/analytics/summary` | GET | Admin | Returns KPIs: total revenue, order count, customer count, AOV. Supports `period` param (7d/30d/90d/custom). |
| `/api/v1/admin/analytics/revenue` | GET | Admin | Revenue time series (daily/weekly/monthly) for charting. |
| `/api/v1/admin/analytics/orders-by-status` | GET | Admin | Distribution of orders across statuses. |
| `/api/v1/admin/analytics/top-products` | GET | Admin | Top products by revenue (limit param, default 10). |
| `/api/v1/admin/analytics/sales-by-category` | GET | Admin | Revenue breakdown per category, with percentages. |

Analytics data is pre-computed via materialized views or cached queries in Redis (TTL 30 min).

---

### 6.10 Notification Module

Handles email delivery and in-app notifications via River background jobs. Subscribes to domain events from the internal event bus.

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/notifications` | GET | Customer | List unread in-app notifications for current user. |
| `/api/v1/notifications/:id/read` | PATCH | Customer | Mark a single notification as read. |
| `/api/v1/notifications/read-all` | PATCH | Customer | Mark all notifications as read.

---

### 6.11 Coupon Management

Coupons are created and managed by admins. The cart module validates and applies coupons at checkout; the order module increments `used_count` on successful order placement.

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/admin/coupons` | GET | Admin | List all coupons with pagination. Supports filter by type and active/expired. |
| `/api/v1/admin/coupons` | POST | Admin | Create a new coupon (code, type, value, min_order_amount, max_uses, expires_at). |
| `/api/v1/admin/coupons/:id` | PUT | Admin | Update coupon fields. |
| `/api/v1/admin/coupons/:id` | DELETE | Admin | Soft-delete coupon (sets `deleted_at`).

---

## 7. Database Schema

All tables include `created_at TIMESTAMPTZ DEFAULT now()` and `updated_at` managed by a trigger. Soft deletes use `deleted_at TIMESTAMPTZ`. UUIDs are used for primary keys (`gen_random_uuid()`).

### 7.1 Key Tables

| Table | Primary Key | Key Columns | Notes |
|---|---|---|---|
| `users` | `id UUID` | `email`, `password_hash`, `role`, `status`, `email_verified_at` | `role: ENUM(guest, customer, seller, admin)` |
| `addresses` | `id UUID` | `user_id FK`, `full_name`, `line1`, `line2`, `city`, `state`, `country`, `postal_code`, `is_default` | |
| `categories` | `id UUID` | `name`, `slug`, `parent_id` (self FK), `depth`, `path` (ltree) | ltree extension for hierarchical queries |
| `products` | `id UUID` | `seller_id FK`, `category_id FK`, `name`, `slug`, `description`, `status`, `base_price`, `currency`, `metadata JSONB` | `status: ENUM(draft, pending, active, archived)` |
| `product_variants` | `id UUID` | `product_id FK`, `sku`, `price_override`, `options JSONB`, `images JSONB[]` | `options: {"size":"M","colour":"Red"}` |
| `inventory` | `id UUID` | `variant_id FK`, `physical_stock`, `reserved_hard`, `reorder_threshold` | `available = physical − reserved_hard` |
| `carts` | `id UUID` | `user_id FK (nullable)`, `session_id`, `expires_at` | session cart for guests |
| `cart_items` | `id UUID` | `cart_id FK`, `variant_id FK`, `quantity`, `unit_price_snapshot` | price snapshot at time of add |
| `orders` | `id UUID` | `user_id FK`, `status`, `shipping_address JSONB`, `subtotal`, `discount`, `tax`, `total`, `currency`, `idempotency_key` | `status: ENUM(...)` |
| `order_items` | `id UUID` | `order_id FK`, `variant_id FK`, `quantity`, `unit_price`, `product_snapshot JSONB` | snapshot prevents stale data |
| `order_events` | `id UUID` | `order_id FK`, `status`, `notes`, `created_by FK` | immutable audit log |
| `payments` | `id UUID` | `order_id FK`, `gateway`, `gateway_payment_id`, `amount`, `currency`, `status`, `metadata JSONB` | |
| `reviews` | `id UUID` | `product_id FK`, `user_id FK`, `order_id FK`, `rating (1–5)`, `title`, `body`, `status`, `moderated_by FK` | `status: ENUM(pending, approved, rejected)` |
| `coupons` | `id UUID` | `code (unique)`, `type (percent/fixed)`, `value`, `min_order_amount`, `max_uses`, `used_count`, `expires_at` | |
| `jobs` (River) | `id BIGINT` | `kind`, `args JSONB`, `state`, `queue`, `priority`, `scheduled_at`, `attempt` | Managed by River library |

### 7.2 Indexes

- `products`: GIN index on `to_tsvector(name || ' ' || description)` for full-text search.
- `products`: BTREE on `(category_id, status)` for catalog listing.
- `orders`: BTREE on `(user_id, created_at DESC)` for customer order history.
- `inventory`: BTREE on `variant_id` (unique) for lock-free reservation updates.
- `coupons`: BTREE on `code` (unique) with partial index `WHERE deleted_at IS NULL`.

---

## 8. Frontend Specifications

### 8.1 Application Structure

```
src/app/
├── core/                        # Singleton services: AuthService, ApiService, TokenInterceptor
├── shared/                      # Shared components, pipes, directives (SpartanUI wrappers)
├── features/
│   ├── home/                    # Landing page, hero, featured products
│   ├── catalog/                 # Product listing, filters, search results
│   ├── product-detail/          # Product page: images, variants, add-to-cart, reviews
│   ├── cart/                    # Cart drawer/page, coupon application
│   ├── checkout/                # Multi-step: address → shipping → payment → confirm
│   ├── orders/                  # Order history, order detail, tracking timeline
│   ├── account/                 # Profile, addresses, wishlist, password change
│   ├── auth/                    # Login, register, forgot/reset password
│   └── admin/                   # Admin dashboard (lazy-loaded, admin role guard)
│       ├── products/
│       ├── orders/
│       ├── inventory/
│       ├── users/
│       └── analytics/
└── app.routes.ts                # Root router with lazy-loaded feature paths
```

### 8.2 SpartanUI Component Mapping

| UI Feature | SpartanUI Component(s) | Notes |
|---|---|---|
| Navigation & Menus | `HlmMenubar`, `HlmNavigationMenu`, `HlmSheet` (mobile nav) | Responsive hamburger nav collapses to Sheet on mobile. |
| Product Cards | `HlmCard`, `HlmBadge` | Skeleton loading state via `HlmSkeleton`. |
| Data Tables (Admin) | `HlmTable`, `HlmPaginator` | Sorting and pagination wired to API cursor params. |
| Forms | `HlmInput`, `HlmLabel`, `HlmSelect`, `HlmCheckbox`, `HlmRadioGroup`, `HlmFormField` | Reactive Forms with typed controls; inline error display. |
| Modals & Alerts | `HlmDialog`, `HlmAlertDialog`, `HlmToast` (Sonner) | Confirm destructive actions via AlertDialog. |
| Cart Drawer | `HlmSheet` | Slides in from right, updates reactively via Signal Store. |
| Image Gallery | Custom + `HlmScrollArea` | Thumbnail strip + zoomed main image. |
| Order Timeline | Custom stepper (TailwindCSS) | Maps `order_events` to a vertical timeline. |
| Rating Input | Custom star component + `HlmButton` | Accessible keyboard-navigable star rating. |
| Search | `HlmCombobox` + `HlmCommand` | Debounced search with command-palette style. |

### 8.3 State Management Strategy

- **Global auth state:** NgRx Signal Store (`isAuthenticated`, `currentUser`, `tokens`).
- **Cart state:** NgRx Signal Store with computed selectors for item count and totals. Synced with server on auth change.
- **Product catalog:** local Component Store per listing page; server-driven filtering.
- **Checkout flow:** Component Store scoped to checkout feature; cleared on order success.
- **Admin tables:** server-side pagination/sort; no client-side data caching.

### 8.4 Authentication Flow

1. User submits login form → `AuthService.login()` → `POST /api/v1/auth/login`.
2. Server returns `{ access_token, refresh_token, expires_in }`.
3. Access token stored in memory; refresh token stored in an `HttpOnly` cookie. Never `localStorage`.
4. `TokenInterceptor` attaches `Bearer` access token to every API request.
5. On 401 response: interceptor calls `/api/v1/auth/refresh`, retries original request once.
6. On refresh failure: redirect to `/login` with `returnUrl`.

---

## 9. Non-Functional Requirements

| Category | Requirement | Target |
|---|---|---|
| Performance | API p95 response time | < 100 ms under 500 concurrent users (excluding payment gateway calls). |
| Performance | Frontend Time to Interactive | < 2.5 s on 4G mobile (Lighthouse score ≥ 85). |
| Scalability | Horizontal scaling | App is stateless; scale horizontally behind a load balancer. DB reads offloaded to read replica. |
| Availability | Uptime SLA | 99.5% monthly (excludes planned maintenance windows). |
| Security | Input validation | All request bodies validated server-side via go-playground/validator before touching business logic. |
| Security | Rate limiting | Login: 10 req/min/IP (Redis token bucket). General API: 1 000 req/min/user. |
| Security | SQL injection | Eliminated by design — sqlc generates parameterised queries; no string interpolation. |
| Security | CORS | Strict allowlist; credentials mode enabled only for same-site origins. |
| Security | HTTPS | TLS 1.2+ enforced; HSTS header with 1-year max-age. |
| Observability | Structured logging | All requests logged with `trace_id`, `user_id`, latency, status via zap. |
| Observability | Distributed tracing | OpenTelemetry spans exported to Tempo; `trace_id` injected in error responses. |
| Reliability | Idempotent mutations | Payment and order creation require `X-Idempotency-Key`; duplicates return cached response. |
| Reliability | Background jobs | All async work (emails, webhook processing) runs via River jobs with retry + dead-letter queue. |
| Accessibility | WCAG compliance | WCAG 2.1 AA. SpartanUI components are ARIA-compliant; tested with axe-core. |
| Testing | Coverage | ≥ 80% line coverage per module (unit + integration). E2E Playwright tests for checkout & auth. |

---

## 10. Security Considerations

### 10.1 Authentication & Authorisation

- JWTs signed with RS256 (asymmetric); private key stored in Vault, public key distributed to app instances.
- Access tokens have 15-minute TTL. Refresh tokens are 7-day rotating tokens stored server-side (Redis) for revocation.
- All admin routes guarded by both JWT middleware and role-assertion middleware.
- Row-level ownership checks: customers can only read/mutate their own orders, addresses, and reviews — enforced in the service layer, not just the router.

### 10.2 Data Protection

- Passwords hashed with bcrypt, cost factor 12. Never stored in plaintext or logged.
- PII (email, phone, address) encrypted at rest using PostgreSQL `pgcrypto` for fields with elevated sensitivity.
- Payment card data never stored. Stripe tokenisation handles card capture client-side.
- Database credentials rotated via HashiCorp Vault dynamic secrets.

### 10.3 API Security

- CSRF protection via `SameSite=Strict` cookie policy for the refresh token cookie.
- All uploads (product images) scanned for MIME type mismatch and capped at 10 MB.
- Stripe webhook signature verified with `stripe.ConstructEvent` before processing.
- All endpoints behind rate-limiting middleware (Redis token bucket).

---

## 11. Development Workflow

### 11.1 Local Development

```bash
# Bring up Postgres, Redis, MinIO, PgBouncer
docker compose up -d

# Apply database migrations
atlas migrate apply --env dev

# Regenerate sqlc query code after SQL changes
sqlc generate

# Run Go backend (hot reload via air)
air

# Run Angular frontend
cd frontend && ng serve
```

### 11.2 sqlc Workflow

```
db/queries/catalog.sql   ← write SQL here
       ↓
sqlc generate            ← run after any SQL change
       ↓
internal/catalog/repository/  ← generated Go types + functions
```

### 11.3 Atlas Migration Workflow

```bash
# Edit db/schema/*.hcl, then diff against live DB
atlas migrate diff --env dev add_product_tags

# Review generated migration file in db/migrations/
# Then apply
atlas migrate apply --env dev
```

### 11.4 CI/CD Pipeline (GitHub Actions)

| Stage | Steps | Gate |
|---|---|---|
| Lint | golangci-lint (Go), ESLint + Prettier (Angular) | PR block on failure |
| Test | `go test ./... -race`; `ng test` (Jest) | PR block on < 80% coverage |
| Build | `go build ./cmd/server`; `ng build --configuration production` | PR block on build failure |
| Migration Check | `atlas migrate lint` — detects destructive migrations | PR block on destructive change without approval |
| Docker Build | `docker buildx build` — multi-arch (amd64, arm64) | Main branch only |
| Deploy (Staging) | Push image, `atlas migrate apply`, docker rollout | Auto on merge to `main` |
| Deploy (Prod) | Same as staging + manual approval gate | Manual trigger |

### 11.5 Branching Strategy

- `main` — production-ready; protected, requires PR + 1 approval.
- `develop` — integration branch; all feature branches merge here first.
- `feature/<ticket-id>-short-description` — standard feature branches.
- `hotfix/<ticket-id>-description` — hotfixes branched from `main`, merged to both `main` and `develop`.

---

## 12. Milestones & Delivery Plan

| Milestone | Scope | Target |
|---|---|---|
| **M0 — Foundation** | Project scaffolding, Docker Compose, DB schema, Atlas migrations, sqlc setup, CI pipeline skeleton, SpartanUI + Tailwind Angular workspace. | Week 1–2 |
| **M1 — Auth & User** | Auth module (register, login, JWT, RBAC), User module (profile, addresses). Angular: auth pages, route guards. | Week 3–4 |
| **M2 — Catalog** | Catalog module (CRUD products, categories, variants, images, full-text search). Angular: product listing, product detail page. | Week 5–7 |
| **M3 — Cart & Checkout** | Cart module (guest + user, merge, coupon). Order module (create order, state machine). Angular: cart drawer, checkout flow. | Week 8–10 |
| **M4 — Payments** | Payment module (Stripe integration, webhooks, idempotency). Order status updates. Angular: Stripe Elements integration, order confirmation. | Week 11–12 |
| **M5 — Inventory & Reviews** | Inventory module (reservations, low-stock alerts). Review module (submit, moderation). Angular: admin inventory table, review UI. | Week 13–14 |
| **M6 — Admin Dashboard** | Admin: order management, product approval, user management, sales metrics. Angular: admin feature module. | Week 15–16 |
| **M7 — Hardening** | Performance profiling, security audit, E2E Playwright tests, Lighthouse optimisation, OpenTelemetry instrumentation, documentation. | Week 17–18 |
| **M8 — Launch** | Staging deployment, UAT, production deployment. | Week 19–20 |

---

## 13. Open Questions & Risks

| ID | Item | Impact | Mitigation |
|---|---|---|---|
| R-01 | Paystack webhook reliability in NGN market | Medium | Implement webhook retry queue via River jobs with exponential backoff. |
| R-02 | sqlc codegen churn if schema changes frequently early on | Low | Lock schema early; treat schema changes as a formal migration PR review step. |
| R-03 | Atlas HCL learning curve for team | Low | Provide team onboarding doc; use `atlas schema inspect` to bootstrap from existing DB. |
| R-04 | SpartanUI component gaps (not all Radix primitives ported to Angular) | Medium | Audit required components in M0; build custom wrappers for any gaps. |
| R-05 | River job queue durability — in-Postgres jobs vs dedicated message broker | Medium | Acceptable for v1 traffic. Evaluate Kafka/RabbitMQ in v2 if throughput demands it. |
| R-06 | Modular boundary enforcement — Go tooling doesn't block internal imports by default | High | Enforce via `depguard` rules in golangci-lint configuration and PR code review checklist. |
| R-07 | GDPR / data residency requirements | Medium | Flag for legal review before EU launch; design for data-region tagging in schema from the start. |

---

## 14. Glossary

| Term | Definition |
|---|---|
| **Modular Monolith** | A single-deployable application whose internal code is organised into discrete, loosely-coupled domain modules with enforced boundaries. |
| **sqlc** | A Go code generator that produces fully type-safe database access code from raw SQL queries. No ORM reflection at runtime. |
| **pgx/v5** | A high-performance, context-aware PostgreSQL driver for Go using the native binary wire protocol. |
| **Atlas** | A database schema management tool that enables declarative HCL-based migrations with automatic diff and CI integration. |
| **River** | A Postgres-backed background job library for Go. Stores jobs in a DB table, eliminating the need for a separate message broker in v1. |
| **SpartanUI** | An Angular component library providing headless, accessible UI primitives (built on Radix UI concepts) with Tailwind-based styling. |
| **PgBouncer** | A lightweight connection pooler for PostgreSQL that runs in transaction mode, reducing active DB connections under load. |
| **Soft Reservation** | A temporary hold on inventory stored in Redis when an item is added to cart, expiring after 30 minutes if no order is placed. |
| **Hard Reservation** | A durable inventory hold written to PostgreSQL when an order is created, released only on cancellation or fulfilment. |
| **RBAC** | Role-Based Access Control — permissions are assigned to roles (Guest, Customer, Seller, Admin) rather than individual users. |
| **RFC 7807** | IETF standard for HTTP API error responses (Problem Details): a JSON object with `type`, `title`, `status`, `detail`, and `instance` fields. |
| **OpenTelemetry** | A vendor-neutral observability framework for generating traces, metrics, and logs, exportable to any compatible backend. |
| **depguard** | A golangci-lint plugin that enforces import restrictions between packages, used here to prevent cross-module internal imports. |

---

*NimCart PRD v1.0 — © 2026. Draft document, subject to change.*
