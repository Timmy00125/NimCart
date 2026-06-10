# Frontend — Angular 22 + SpartanUI + TailwindCSS

## Standards & Conventions (Angular 22 Best Practices)
- **Standalone components only** — no NgModules. `standalone: true` is the default in Angular 20+.
- **Signals for state management** — use `signal()`, `computed()`, `effect()`. NgRx Signal Store for feature/global state.
- **`inject()` function** — use instead of constructor injection.
- **`input()` and `output()` functions** — use instead of `@Input()` and `@Output()` decorators.
- **`OnPush` change detection** — set `changeDetection: ChangeDetectionStrategy.OnPush` on all components.
- **Native control flow** — use `@if`, `@for`, `@switch` instead of `*ngIf`, `*ngFor`, `*ngSwitch`.
- **Lazy loading** — all feature routes use `loadComponent` / `loadChildren` with dynamic imports.
- **Reactive Forms** — typed forms with composable validators. No template-driven forms.
- **`NgOptimizedImage`** — use for all static images.
- **Accessibility** — WCAG 2.1 AA. All SpartanUI components are ARIA-compliant. Test with axe-core.
- **No `ngClass`/`ngStyle`** — use `class` and `style` bindings instead.
- **No `@HostBinding`/`@HostListener`** — use `host` object in decorator.

---

## Project Scaffolding (M0)

### Angular Workspace
- [ ] Create Angular workspace: `ng new frontend --standalone --style=css --routing --ssr`
- [ ] Enable strict mode in `tsconfig.json`: `"strict": true`, `"noImplicitOverride": true`
- [ ] Configure `angular.json`:
  - [ ] Set `budgets` for bundle size warnings
  - [ ] Enable `sourceMap` for development
  - [ ] Configure `assets` and `styles` paths

### TailwindCSS Setup
- [ ] Install: `npm install -D tailwindcss @tailwindcss/postcss postcss autoprefixer`
- [ ] Create `tailwind.config.js` with:
  - [ ] `content` paths: `./src/**/*.{html,ts}`
  - [ ] Extend theme with NimCart design tokens (colors, spacing, typography)
  - [ ] Import SpartanUI Tailwind preset
- [ ] Create `postcss.config.js` with Tailwind plugin
- [ ] Add `@tailwind base; @tailwind components; @tailwind utilities;` to `styles.css`

### SpartanUI Setup
- [ ] Install brain primitives: `npm install @spartan-ng/brain`
- [ ] Install CLI: `npm install -D @spartan-ng/cli`
- [ ] Import Tailwind preset: `@import "@spartan-ng/brain/hlm-tailwind-preset.css"` in `styles.css`
- [ ] Add required components via CLI (`ng g @spartan-ng/cli:ui`):
  - [ ] Button, Card, Badge, Input, Label, Select, Checkbox, RadioGroup
  - [ ] Dialog, AlertDialog, Sheet, ScrollArea
  - [ ] Table, Paginator
  - [ ] Menubar, NavigationMenu
  - [ ] FormField, Toast (Sonner), Skeleton, Combobox, Command
  - [ ] Separator, Avatar, Tabs, Accordion

### NgRx Signal Store
- [ ] Install: `npm install @ngrx/signals @ngrx/operators`
- [ ] Create global stores:
  - [ ] `AuthStore` — `isAuthenticated`, `currentUser`, `tokens`
  - [ ] `CartStore` — `items`, `subtotal`, `discount`, `total`, `itemCount`
- [ ] Feature stores per lazy-loaded feature (provided at component/route level)

### Environment Configuration
- [ ] Create `src/environments/environment.ts` (dev) and `environment.prod.ts`
- [ ] Define: `apiUrl`, `stripePublishableKey`, `sentryDsn`

### ESLint & Prettier
- [ ] Install and configure `@angular-eslint` with recommended rules
- [ ] Install Prettier with Angular plugin
- [ ] Create `.eslintrc.json` with Angular + TypeScript rules
- [ ] Create `.prettierrc` with project formatting rules
- [ ] Add `lint` and `format` scripts to `package.json`

### Testing Setup
- [ ] Configure Jest: `npm install -D jest @angular-builders/jest jest-preset-angular`
- [ ] Configure Angular Testing Library: `npm install -D @testing-library/angular @testing-library/jest-dom`
- [ ] Set up Playwright: `npm install -D @playwright/test`
- [ ] Create `playwright.config.ts` with base URL, test directory, browser projects
- [ ] Add test scripts to `package.json`: `test`, `test:watch`, `test:coverage`, `e2e`

---

## Core Module (`src/app/core/`)

### AuthService
- [ ] `login(email, password): Observable<AuthResponse>` — POST `/api/v1/auth/login`
- [ ] `register(data): Observable<AuthResponse>` — POST `/api/v1/auth/register`
- [ ] `refresh(): Observable<AuthResponse>` — POST `/api/v1/auth/refresh`
- [ ] `logout(): Observable<void>` — POST `/api/v1/auth/logout`
- [ ] `forgotPassword(email): Observable<void>` — POST `/api/v1/auth/forgot-password`
- [ ] `resetPassword(token, password): Observable<void>` — POST `/api/v1/auth/reset-password`
- [ ] `getMe(): Observable<User>` — GET `/api/v1/auth/me`
- [ ] Store access token in memory (Signal), refresh token in HttpOnly cookie (server-managed)

### ApiService
- [ ] Wrapper around `HttpClient` with base URL from environment
- [ ] Methods: `get<T>`, `post<T>`, `put<T>`, `patch<T>`, `delete<T>`
- [ ] Automatic `Content-Type: application/json` header
- [ ] Error normalization: convert RFC 7807 responses to typed `ApiError`

### TokenInterceptor
- [ ] Intercept all outgoing requests: attach `Authorization: Bearer {access_token}` from AuthStore
- [ ] On 401 response: call `refresh()`, retry original request once
- [ ] On refresh failure: redirect to `/login` with `returnUrl`
- [ ] Prevent concurrent refresh calls (queue pending requests)

### AuthGuard
- [ ] `canActivate` — check `AuthStore.isAuthenticated()`, redirect to `/login` if not
- [ ] Support `returnUrl` query param for post-login redirect

### AdminGuard
- [ ] `canActivate` — check `AuthStore.currentUser()?.role === 'admin'`, redirect to `/` if not

### RoleGuard
- [ ] `canActivate` — check user role against route `data.allowedRoles`

---

## Shared Module (`src/app/shared/`)

### SpartanUI Wrapper Components
- [ ] Re-export all SpartanUI styled components for use across features
- [ ] Create custom themed wrappers where needed

### Shared Components
- [ ] `ProductCardComponent` — card with image, name, price, rating, add-to-cart button
- [ ] `ProductSkeletonComponent` — skeleton loading state for product cards
- [ ] `StarRatingComponent` — accessible keyboard-navigable star rating (display + input modes)
- [ ] `OrderTimelineComponent` — vertical timeline from order_events
- [ ] `ImageGalleryComponent` — thumbnail strip + zoomed main image
- [ ] `SearchBarComponent` — debounced search with HlmCombobox + HlmCommand
- [ ] `EmptyStateComponent` — reusable empty state with icon, title, action
- [ ] `LoadingSpinnerComponent` — centered spinner with optional message
- [ ] `PriceDisplayComponent` — formatted price with currency (uses Intl.NumberFormat)
- [ ] `PaginationComponent` — cursor-based pagination controls wired to API

### Shared Pipes
- [ ] `CurrencyPipe` — format money with symbol and decimals
- [ ] `TimeAgoPipe` — relative time display ("2 hours ago")
- [ ] `TruncatePipe` — truncate text with ellipsis

### Shared Directives
- [ ] `LazyLoadDirective` — intersection observer for lazy-loading images

---

## App Configuration

### app.routes.ts
- [ ] Define root routes with lazy loading:
  - [ ] `''` → `HomeComponent` (eager, landing page)
  - [ ] `'products'` → lazy load `CatalogComponent`
  - [ ] `'products/:slug'` → lazy load `ProductDetailComponent`
  - [ ] `'cart'` → lazy load `CartComponent`
  - [ ] `'checkout'` → lazy load `CheckoutComponent` (AuthGuard)
  - [ ] `'orders'` → lazy load `OrdersComponent` (AuthGuard)
  - [ ] `'account'` → lazy load `AccountComponent` (AuthGuard)
  - [ ] `'login'` → lazy load `LoginComponent`
  - [ ] `'register'` → lazy load `RegisterComponent`
  - [ ] `'forgot-password'` → lazy load `ForgotPasswordComponent`
  - [ ] `'reset-password'` → lazy load `ResetPasswordComponent`
  - [ ] `'admin'` → lazy load `AdminComponent` + `admin.routes.ts` (AdminGuard)
  - [ ] `'**'` → `NotFoundComponent`

### app.component.ts
- [ ] Root layout: header (navbar), router-outlet, footer
- [ ] Provide global stores: `AuthStore`, `CartStore`
- [ ] Initialize auth state on app load (check for existing session)

### Layout Components
- [ ] `HeaderComponent` — logo, navigation menu (HlmNavigationMenu), search bar, cart icon with badge, user menu
- [ ] `FooterComponent` — links, copyright
- [ ] `MobileNavComponent` — hamburger menu using HlmSheet for mobile

---

## Feature: Home / Landing Page

### Component: `HomeComponent`
- [ ] Hero section: full-width banner with headline, CTA button ("Shop Now")
- [ ] Featured categories section: grid of category cards (image + name), link to catalog with filter
- [ ] Featured products section: horizontal scroll or grid of `ProductCardComponent` (top 8 by sales/rating)
- [ ] Promotional banner section: seasonal offers, free shipping threshold
- [ ] New arrivals section: latest 4 products
- [ ] Trust badges: secure payment, free returns, fast shipping

### State
- [ ] `featuredProducts` signal — loaded from `GET /api/v1/products?sort=featured&limit=8`
- [ ] `newArrivals` signal — loaded from `GET /api/v1/products?sort=newest&limit=4`
- [ ] `categories` signal — loaded from `GET /api/v1/categories`
- [ ] `isLoading` computed signal — true while any data is loading

### Data Fetching
- [ ] Use `rxMethod` from NgRx Signal Store or `effect()` with `HttpClient`
- [ ] Fetch featured products, new arrivals, and categories on component init
- [ ] Skeleton loading states while data loads (`ProductSkeletonComponent`)

### SEO & Meta
- [ ] Set page title: "NimCart — Your One-Stop Shop"
- [ ] Set meta description for search engines
- [ ] Open Graph tags for social sharing

### Accessibility
- [ ] All images have descriptive `alt` text
- [ ] CTA buttons have clear accessible names
- [ ] Skip-to-content link
- [ ] Color contrast meets WCAG AA

### Tests
- [ ] Component test: renders hero, featured products, categories
- [ ] Component test: shows skeleton while loading
- [ ] Component test: handles empty featured products gracefully

---

## Feature: Catalog (Product Listing, Filters, Search)

### Component: `CatalogComponent`
- [ ] Product grid: responsive grid (2 cols mobile, 3 tablet, 4 desktop) of `ProductCardComponent`
- [ ] Filter sidebar (desktop) / filter sheet (mobile via `HlmSheet`):
  - [ ] Category filter: tree select from categories
  - [ ] Price range: min/max inputs
  - [ ] Tags: checkbox list
  - [ ] Sort: select (newest, price low-high, price high-low, rating)
- [ ] Active filters display: removable filter chips/badges
- [ ] Search results mode: when `q` query param present, show "Results for '{query}'" header
- [ ] Empty state: "No products found" with suggestion to clear filters
- [ ] Infinite scroll or "Load More" button with cursor pagination

### State: `CatalogStore` (NgRx Signal Store)
- [ ] `products` signal — current page of products
- [ ] `filters` signal — `{ category_id, min_price, max_price, tags, sort }`
- [ ] `searchQuery` signal — from route query param `q`
- [ ] `cursor` signal — current pagination cursor
- [ ] `hasMore` signal — whether more pages exist
- [ ] `isLoading` signal
- [ ] `totalCount` signal — for display ("Showing X of Y products")
- [ ] Methods: `loadProducts()`, `updateFilters()`, `clearFilters()`, `loadMore()`, `setSort()`

### Routing & Query Params
- [ ] Sync filters to URL query params for shareability and back-button support
- [ ] `?category=electronics&min_price=1000&max_price=5000&sort=price_asc&cursor=xxx`
- [ ] On query param change: update store, re-fetch products

### Data Fetching
- [ ] `GET /api/v1/products` with filter params and cursor
- [ ] `GET /api/v1/products/search?q={query}` when search mode
- [ ] Debounce filter changes (300ms) before API call
- [ ] Cancel in-flight requests on filter change (switchMap)

### Sub-Components
- [ ] `FilterSidebarComponent` — category tree, price range, tags, sort
- [ ] `ActiveFiltersComponent` — removable filter badges
- [ ] `ProductGridComponent` — responsive grid layout

### Tests
- [ ] Component test: renders product grid with mock data
- [ ] Component test: filter changes trigger API call
- [ ] Component test: pagination loads more products
- [ ] Component test: search mode shows correct header
- [ ] Component test: empty state when no products

---

## Feature: Product Detail

### Component: `ProductDetailComponent`
- [ ] Image gallery: main image + thumbnail strip (`ImageGalleryComponent`)
  - [ ] Click thumbnail to swap main image
  - [ ] Click main image to open zoom dialog (`HlmDialog`)
  - [ ] Keyboard navigation between images
- [ ] Product info sections:
  - [ ] Name, slug, category breadcrumb
  - [ ] Price display (base price or variant price override)
  - [ ] Average rating + review count (link to reviews section)
  - [ ] Description (rich text from product.description)
  - [ ] Status badge (only show Active products to customers)
- [ ] Variant selector:
  - [ ] Render variant options from `options` JSONB (e.g., Size: S/M/L, Color: Red/Blue)
  - [ ] Button group or select per option dimension
  - [ ] On variant selection: update price, stock status, images
  - [ ] Disable unavailable variants (out of stock)
- [ ] Quantity selector: numeric input with +/- buttons, max = available stock
- [ ] Add to cart button:
  - [ ] Call `CartStore.addItem(variantId, quantity)`
  - [ ] Show success toast (HlmToast/Sonner)
  - [ ] Disable if out of stock
  - [ ] Loading state while adding
- [ ] Add to wishlist button: heart icon, toggle state
- [ ] Stock status indicator: "In Stock" (green), "Low Stock" (orange, < 5), "Out of Stock" (red)
- [ ] Product metadata: render `metadata` JSONB as key-value pairs or specs table

### Reviews Section
- [ ] Review summary: average rating, total count, star distribution bar chart
- [ ] Review list: paginated list of approved reviews
  - [ ] Reviewer name, rating stars, title, body, date
- [ ] Write review form (visible only if user has purchased and delivered):
  - [ ] Star rating input (`StarRatingComponent`)
  - [ ] Title input, body textarea
  - [ ] Submit button → `POST /api/v1/products/:id/reviews`
  - [ ] Success message: "Review submitted for moderation"
- [ ] "Load more reviews" with cursor pagination

### State: `ProductDetailStore` (NgRx Signal Store)
- [ ] `product` signal — full product with variants and images
- [ ] `selectedVariant` computed signal — based on selected options
- [ ] `selectedOptions` signal — `{ size: 'M', color: 'Red' }`
- [ ] `quantity` signal — selected quantity
- [ ] `reviews` signal — paginated approved reviews
- [ ] `reviewSummary` signal — average, count, distribution
- [ ] `isLoading` signal
- [ ] `isAddingToCart` signal
- [ ] Methods: `loadProduct(slug)`, `selectOption(key, value)`, `setQuantity(n)`, `addToCart()`, `loadReviews()`, `submitReview(data)`

### Routing
- [ ] Route: `/products/:slug`
- [ ] Load product by slug on route param change
- [ ] Set page title to product name for SEO

### Tests
- [ ] Component test: renders product info, images, variants
- [ ] Component test: variant selection updates price and stock
- [ ] Component test: add to cart calls store method
- [ ] Component test: review form visible only for eligible users
- [ ] Component test: skeleton loading state

---

## Feature: Cart

### CartStore (Global, provided at root)
- [ ] State:
  - [ ] `items` signal — `CartItem[]`
  - [ ] `subtotal` computed signal — sum of (item.unitPrice * item.quantity)
  - [ ] `discount` computed signal — from applied coupon
  - [ ] `total` computed signal — subtotal - discount
  - [ ] `itemCount` computed signal — sum of quantities
  - [ ] `couponCode` signal — applied coupon code (nullable)
  - [ ] `isLoading` signal
  - [ ] `isCartOpen` signal — controls drawer visibility
- [ ] Methods:
  - [ ] `loadCart()` — GET `/api/v1/cart`, populate state
  - [ ] `addItem(variantId, quantity)` — POST `/api/v1/cart/items`, update state
  - [ ] `updateQuantity(itemId, quantity)` — PATCH `/api/v1/cart/items/:itemId`
  - [ ] `removeItem(itemId)` — DELETE `/api/v1/cart/items/:itemId`
  - [ ] `clearCart()` — DELETE `/api/v1/cart`
  - [ ] `applyCoupon(code)` — POST `/api/v1/cart/apply-coupon`
  - [ ] `mergeCarts()` — POST `/api/v1/cart/merge` (called post-login)
  - [ ] `openCart()` / `closeCart()` — toggle drawer

### Cart Drawer Component (`CartDrawerComponent`)
- [ ] Slide-in from right using `HlmSheet`
- [ ] Header: "Your Cart (X items)" with close button
- [ ] Item list:
  - [ ] Product image (thumbnail), name, variant options
  - [ ] Quantity controls: - / display / + buttons
  - [ ] Unit price and line total
  - [ ] Remove button (trash icon)
  - [ ] Stock warning if quantity > available
- [ ] Coupon section:
  - [ ] Input field + "Apply" button
  - [ ] Display applied coupon with remove option
  - [ ] Error message for invalid coupon
- [ ] Totals section:
  - [ ] Subtotal, Discount (if coupon), Total
  - [ ] "Proceed to Checkout" button → navigate to `/checkout`
  - [ ] "Continue Shopping" link
- [ ] Empty state: "Your cart is empty" with "Browse Products" CTA
- [ ] Loading skeleton while cart loads

### Cart Page Component (`CartComponent`)
- [ ] Full-page version for mobile/larger screens (alternative to drawer)
- [ ] Same content as drawer but in page layout
- [ ] "Clear Cart" button with confirmation dialog (`HlmAlertDialog`)

### Behavior
- [ ] On auth change (login): call `mergeCarts()`, then `loadCart()`
- [ ] On auth change (logout): call `loadCart()` (fetches guest cart)
- [ ] Real-time stock validation: items show warning if stock decreased since add
- [ ] Debounce quantity updates (300ms) before API call

### Tests
- [ ] Store test: addItem, removeItem, updateQuantity update state correctly
- [ ] Store test: computed signals (subtotal, total, itemCount)
- [ ] Component test: renders items, handles empty cart
- [ ] Component test: coupon apply/remove
- [ ] Component test: quantity change calls API
- [ ] Component test: clear cart shows confirmation

---

## Feature: Checkout (Multi-Step)

### CheckoutStore (Feature-scoped)
- [ ] State:
  - [ ] `step` signal — current step (1-4)
  - [ ] `selectedAddressId` signal
  - [ ] `addresses` signal — user's saved addresses
  - [ ] `cart` signal — current cart (from CartStore)
  - [ ] `paymentIntentClientSecret` signal — from payment service
  - [ ] `isProcessing` signal
  - [ ] `orderId` signal — set on successful order creation
  - [ ] `error` signal — error message if any
- [ ] Methods:
  - [ ] `loadAddresses()` — GET `/api/v1/users/addresses`
  - [ ] `selectAddress(id)` — set selected address
  - [ ] `createOrder()` — POST `/api/v1/orders` with idempotency key
  - [ ] `createPaymentIntent()` — POST `/api/v1/payments/intent`
  - [ ] `confirmPayment()` — Stripe confirmCardPayment
  - [ ] `goToStep(n)` — navigate between steps
  - [ ] `reset()` — clear all state after order success

### Step 1: Shipping Address
- [ ] Display saved addresses as radio cards (`HlmRadioGroup`)
  - [ ] Each card: full name, address lines, city, state, postal code
  - [ ] Default address pre-selected
- [ ] "Add New Address" button → inline form or dialog
  - [ ] Reactive form: full_name, line1, line2, city, state, country, postal_code
  - [ ] Validators: required fields, postal code format
- [ ] "Continue" button → go to step 2

### Step 2: Order Summary & Review
- [ ] Cart items summary: product name, variant, quantity, unit price, line total
- [ ] Shipping address display
- [ ] Totals: subtotal, discount, tax, total
- [ ] Coupon display (if applied)
- [ ] "Edit Cart" link → navigate to `/cart`
- [ ] "Continue to Payment" button → create order (POST /api/v1/orders), go to step 3

### Step 3: Payment
- [ ] Stripe Elements integration:
  - [ ] Load Stripe.js with publishable key from environment
  - [ ] Render Card Element (card number, expiry, CVC)
  - [ ] Create payment intent on step entry → get client secret
- [ ] "Pay {total}" button:
  - [ ] Call `stripe.confirmCardPayment(clientSecret, { payment_method: { card } })`
  - [ ] Loading state during confirmation
  - [ ] Handle errors: card declined, insufficient funds, etc.
  - [ ] On success: go to step 4
- [ ] Security notice: "Your payment info is encrypted and secure"

### Step 4: Order Confirmation
- [ ] Success message: "Order placed successfully!"
- [ ] Order number and summary
- [ ] Estimated delivery date
- [ ] "View Order" button → navigate to `/orders/:id`
- [ ] "Continue Shopping" button → navigate to `/products`
- [ ] Clear cart and checkout state

### Checkout Stepper UI
- [ ] Visual step indicator: 1. Address → 2. Review → 3. Payment → 4. Confirmation
- [ ] Completed steps show checkmark
- [ ] Current step highlighted
- [ ] Cannot skip ahead (must complete each step)
- [ ] "Back" button on steps 2-3

### Idempotency
- [ ] Generate unique idempotency key (crypto.randomUUID()) on checkout start
- [ ] Include in `X-Idempotency-Key` header on order creation
- [ ] If user refreshes during checkout: detect in-progress order, resume or restart

### Error Handling
- [ ] Network errors: retry button
- [ ] Payment errors: show Stripe error message, allow retry
- [ ] Stock errors (item out of stock during checkout): redirect to cart with message
- [ ] Session expiry: redirect to login with returnUrl

### Tests
- [ ] Store test: step navigation, state transitions
- [ ] Component test: address selection, form validation
- [ ] Component test: payment flow with mocked Stripe
- [ ] Component test: order confirmation renders correctly
- [ ] E2E test: full checkout flow (Playwright)

---

## Feature: Orders (History & Detail)

### Component: `OrderListComponent`
- [ ] Order list table (desktop) / card list (mobile):
  - [ ] Order number, date, status badge (color-coded), total, item count
  - [ ] Click row → navigate to order detail
- [ ] Status filter tabs: All, Pending, Paid, Shipped, Delivered, Cancelled
- [ ] Pagination: "Load More" with cursor
- [ ] Empty state: "No orders yet" with "Start Shopping" CTA
- [ ] Loading skeleton

### Component: `OrderDetailComponent`
- [ ] Order header: order number, date, status badge
- [ ] Order timeline (`OrderTimelineComponent`):
  - [ ] Vertical timeline from `order_events`
  - [ ] Each event: status, date, notes, who performed
  - [ ] Current status highlighted
  - [ ] Icons per status (clock, check, truck, package, etc.)
- [ ] Order items table:
  - [ ] Product image (thumbnail), name, variant options
  - [ ] Quantity, unit price, line total
  - [ ] Click product name → navigate to product detail
- [ ] Shipping address card
- [ ] Payment summary: subtotal, discount, tax, total
- [ ] Actions:
  - [ ] "Cancel Order" button (visible if status is PendingPayment or Paid)
  - [ ] Confirmation dialog before cancellation (`HlmAlertDialog`)
  - [ ] "Request Refund" button (visible if status is Delivered)
  - [ ] Refund reason form (textarea, min 10 chars)
- [ ] Tracking number display (if status is Shipped)

### State: `OrdersStore` (NgRx Signal Store)
- [ ] `orders` signal — paginated order list
- [ ] `selectedOrder` signal — full order detail
- [ ] `statusFilter` signal
- [ ] `cursor` signal
- [ ] `hasMore` signal
- [ ] `isLoading` signal
- [ ] Methods: `loadOrders(filter?)`, `loadOrder(id)`, `cancelOrder(id)`, `requestRefund(id, reason)`, `loadMore()`

### Routing
- [ ] `/orders` → `OrderListComponent`
- [ ] `/orders/:id` → `OrderDetailComponent`
- [ ] Both behind AuthGuard

### Tests
- [ ] Component test: order list renders with mock data
- [ ] Component test: status filter changes trigger re-fetch
- [ ] Component test: order detail shows timeline, items, actions
- [ ] Component test: cancel button visible only for eligible statuses
- [ ] Component test: refund form validation

---

## Feature: Account (Profile, Addresses, Wishlist, Security)

### Component: `AccountComponent`
- [ ] Tab layout using `HlmTabs`:
  - [ ] Profile tab
  - [ ] Addresses tab
  - [ ] Wishlist tab
  - [ ] Security tab (password change)

### Profile Tab
- [ ] Display current profile: avatar, name, email, phone
- [ ] Edit form (Reactive Forms):
  - [ ] Name input (required, min 2)
  - [ ] Phone input (optional, E.164 format validator)
  - [ ] Avatar upload: file input with preview, upload to S3 via presigned URL
  - [ ] Email display (read-only, cannot change in v1)
- [ ] Save button → PATCH `/api/v1/users/profile`
- [ ] Success toast on save

### Addresses Tab
- [ ] List of saved addresses as cards:
  - [ ] Full name, address lines, city, state, country, postal code
  - [ ] "Default" badge on default address
  - [ ] Edit button → open edit dialog/form
  - [ ] Delete button → confirmation dialog
- [ ] "Add New Address" button → inline form or dialog:
  - [ ] Reactive form: full_name, line1, line2, city, state, country (select), postal_code
  - [ ] "Set as default" checkbox
  - [ ] Validators: required fields, postal code format
- [ ] Set as default: toggle on any address → PATCH with is_default=true

### Wishlist Tab
- [ ] Grid of wishlist product cards:
  - [ ] Product image, name, price, stock status
  - [ ] "Add to Cart" button (if in stock)
  - [ ] "Remove" button (heart icon toggle)
- [ ] Empty state: "Your wishlist is empty" with "Browse Products" CTA
- [ ] Loading skeleton

### Security Tab
- [ ] Change password form:
  - [ ] Current password input
  - [ ] New password input (min 8, max 128)
  - [ ] Confirm new password input (must match)
  - [ ] Password strength indicator
  - [ ] Submit → POST `/api/v1/auth/reset-password` (or dedicated change-password endpoint)
- [ ] Success: log out all sessions, redirect to login

### State: `AccountStore` (Feature-scoped)
- [ ] `profile` signal
- [ ] `addresses` signal
- [ ] `wishlist` signal
- [ ] `isLoading` signal
- [ ] `isSaving` signal
- [ ] Methods: `loadProfile()`, `updateProfile(data)`, `loadAddresses()`, `addAddress(data)`, `updateAddress(id, data)`, `deleteAddress(id)`, `loadWishlist()`, `removeFromWishlist(productId)`, `addToCartFromWishlist(productId)`

### Tests
- [ ] Component test: profile form pre-filled with current data
- [ ] Component test: address CRUD operations
- [ ] Component test: wishlist add/remove
- [ ] Component test: password change validation (match, min length)
- [ ] Component test: form validation errors displayed

---

## Feature: Auth (Login, Register, Forgot/Reset Password)

### Component: `LoginComponent`
- [ ] Email input (required, email format validator)
- [ ] Password input (required) with show/hide toggle
- [ ] "Remember me" checkbox (extends refresh token TTL — optional v1)
- [ ] "Login" button → call `AuthStore.login(email, password)`
- [ ] Loading state during login
- [ ] Error display: invalid credentials, account suspended
- [ ] "Forgot password?" link → `/forgot-password`
- [ ] "Don't have an account? Register" link → `/register`
- [ ] On success: redirect to `returnUrl` query param or `/`
- [ ] Rate limit display: "Too many attempts, please wait" after 429 response

### Component: `RegisterComponent`
- [ ] Name input (required, min 2)
- [ ] Email input (required, email format)
- [ ] Password input (required, min 8, max 128) with strength indicator
- [ ] Confirm password input (must match)
- [ ] Terms & conditions checkbox (required)
- [ ] "Create Account" button → call `AuthStore.register(data)`
- [ ] Loading state
- [ ] Error display: email already exists, validation errors
- [ ] "Already have an account? Login" link → `/login`
- [ ] On success: auto-login, redirect to `/`

### Component: `ForgotPasswordComponent`
- [ ] Email input (required, email format)
- [ ] "Send Reset Link" button → call `AuthService.forgotPassword(email)`
- [ ] Always show success message: "If an account exists with that email, we've sent a reset link" (prevent enumeration)
- [ ] "Back to Login" link

### Component: `ResetPasswordComponent`
- [ ] Extract `token` from query params
- [ ] New password input (required, min 8) with strength indicator
- [ ] Confirm password input (must match)
- [ ] "Reset Password" button → call `AuthService.resetPassword(token, password)`
- [ ] Success: redirect to `/login` with success message
- [ ] Error: invalid/expired token → show "Link expired" message with "Request new link" option

### State: `AuthStore` (Global, provided at root)
- [ ] State:
  - [ ] `isAuthenticated` computed signal — `!!accessToken()`
  - [ ] `currentUser` signal — `User | null`
  - [ ] `accessToken` signal — stored in memory only
  - [ ] `isLoading` signal
- [ ] Methods:
  - [ ] `login(email, password)` — POST login, store tokens, set user
  - [ ] `register(data)` — POST register, store tokens, set user
  - [ ] `refresh()` — POST refresh, update access token
  - [ ] `logout()` — POST logout, clear tokens, redirect to `/login`
  - [ ] `loadUser()` — GET /auth/me, set currentUser (called on app init)
  - [ ] `initialize()` — check for existing session on app load

### Guards
- [ ] `AuthGuard` — redirect to `/login?returnUrl=...` if not authenticated
- [ ] `GuestGuard` — redirect to `/` if already authenticated (for login/register pages)

### Tests
- [ ] Component test: login form validation, submit, error display
- [ ] Component test: register form validation, password match, submit
- [ ] Component test: forgot password always shows success
- [ ] Component test: reset password token validation
- [ ] Store test: login sets tokens and user
- [ ] Store test: logout clears state
- [ ] Guard test: redirects unauthenticated users
- [ ] E2E test: full login flow (Playwright)

---

## Feature: Admin Dashboard

### Admin Layout Component
- [ ] Sidebar navigation (collapsible on mobile):
  - [ ] Dashboard (analytics)
  - [ ] Products
  - [ ] Orders
  - [ ] Inventory
  - [ ] Users
  - [ ] Reviews
- [ ] Top bar: admin name, logout button, notification bell
- [ ] Main content area: `<router-outlet>`
- [ ] Responsive: sidebar collapses to hamburger menu on mobile

### Admin Routes (`admin.routes.ts`)
- [ ] `''` → `AdminDashboardComponent` (analytics)
- [ ] `'products'` → `AdminProductsComponent`
- [ ] `'products/:id'` → `AdminProductDetailComponent`
- [ ] `'orders'` → `AdminOrdersComponent`
- [ ] `'orders/:id'` → `AdminOrderDetailComponent`
- [ ] `'inventory'` → `AdminInventoryComponent`
- [ ] `'users'` → `AdminUsersComponent`
- [ ] `'reviews'` → `AdminReviewsComponent`
- [ ] All routes behind AdminGuard

### Shared Admin Components
- [ ] `AdminDataTableComponent<T>` — reusable data table with:
  - [ ] Column definitions (header, cell template, sortable)
  - [ ] Server-side pagination (cursor-based)
  - [ ] Sort controls
  - [ ] Row actions (edit, delete, view)
  - [ ] Loading skeleton
  - [ ] Empty state
- [ ] `AdminStatCardComponent` — metric card with label, value, trend indicator
- [ ] `AdminConfirmDialogComponent` — reusable confirmation dialog for destructive actions

### Tests
- [ ] Component test: admin layout renders sidebar and router-outlet
- [ ] Guard test: non-admin users redirected
- [ ] Component test: data table pagination, sorting

---

## Feature: Admin Products

### Component: `AdminProductsComponent`
- [ ] Data table of all products:
  - [ ] Columns: image thumbnail, name, category, status badge, price, stock, seller, created date
  - [ ] Status filter: All, Draft, Pending Review, Active, Archived
  - [ ] Search by product name
  - [ ] Sort by: name, price, created date
  - [ ] Cursor pagination
- [ ] Row actions:
  - [ ] View → navigate to product detail
  - [ ] Edit → open edit dialog or navigate to edit page
  - [ ] Approve/Reject (for Pending Review products)
  - [ ] Archive (soft delete)
- [ ] "Create Product" button → open create form

### Component: `AdminProductFormComponent` (Create/Edit)
- [ ] Reactive form fields:
  - [ ] Name (required), Description (textarea), Category (select from tree)
  - [ ] Base price (number, cents), Currency (select)
  - [ ] Status (select, admin only for transitions)
  - [ ] Metadata (key-value editor for JSONB)
- [ ] Variant management:
  - [ ] Add variant: SKU, price override, options (dynamic key-value)
  - [ ] Remove variant
  - [ ] Edit variant
- [ ] Image upload:
  - [ ] Drag-and-drop or file select
  - [ ] Upload via presigned URL
  - [ ] Preview thumbnails, reorder, delete
- [ ] Save button → POST/PUT API call
- [ ] Validation: required fields, price > 0, SKU uniqueness

### Component: `AdminProductDetailComponent`
- [ ] Full product info display
- [ ] Status transition buttons with confirmation:
  - [ ] Draft → Pending Review
  - [ ] Pending Review → Active (approve)
  - [ ] Pending Review → Draft (reject with reason)
  - [ ] Active → Archived
- [ ] Edit button → open form
- [ ] Delete button → confirmation dialog, soft delete

### State: `AdminProductsStore`
- [ ] `products` signal, `filters` signal, `cursor` signal, `isLoading` signal
- [ ] Methods: `loadProducts()`, `createProduct(data)`, `updateProduct(id, data)`, `transitionStatus(id, status)`, `deleteProduct(id)`

### Tests
- [ ] Component test: table renders with mock data
- [ ] Component test: form validation
- [ ] Component test: status transition buttons visible for correct statuses
- [ ] Component test: delete confirmation

---

## Feature: Admin Orders

### Component: `AdminOrdersComponent`
- [ ] Data table of all orders:
  - [ ] Columns: order number, customer name/email, status badge, total, items count, date
  - [ ] Filters: status, date range (from/to), customer
  - [ ] Sort by: date (newest/oldest), total
  - [ ] Cursor pagination
- [ ] Row click → navigate to order detail
- [ ] Export to CSV button (optional v1)

### Component: `AdminOrderDetailComponent`
- [ ] Order header: order number, customer info, date, status badge
- [ ] Order timeline (all order_events)
- [ ] Order items table: product, variant, quantity, unit price, line total
- [ ] Shipping address
- [ ] Payment info: gateway, amount, status, gateway reference
- [ ] Status transition controls:
  - [ ] Select new status from valid transitions (based on FSM)
  - [ ] Notes textarea (optional)
  - [ ] Tracking number input (for Shipped status)
  - [ ] "Update Status" button with confirmation
- [ ] Refund section:
  - [ ] View refund requests
  - [ ] Approve/reject refund with notes
  - [ ] Refund amount input (capped at original charge)

### State: `AdminOrdersStore`
- [ ] `orders` signal, `selectedOrder` signal, `filters` signal, `cursor` signal, `isLoading` signal
- [ ] Methods: `loadOrders(filter)`, `loadOrder(id)`, `updateStatus(id, status, notes)`, `processRefund(orderId, amount)`

### Tests
- [ ] Component test: table renders with filters
- [ ] Component test: status transition shows only valid options
- [ ] Component test: refund amount validation

---

## Feature: Admin Inventory

### Component: `AdminInventoryComponent`
- [ ] Data table of inventory levels:
  - [ ] Columns: product name, variant (SKU, options), physical stock, reserved, available, reorder threshold
  - [ ] Color-coded stock status: green (healthy), orange (low), red (depleted)
  - [ ] Sort by: product name, stock level
  - [ ] Cursor pagination
- [ ] Filter tabs: All, Low Stock, Out of Stock
- [ ] Row actions:
  - [ ] "Adjust Stock" → open adjustment dialog
  - [ ] View audit history

### Stock Adjustment Dialog (`HlmDialog`)
- [ ] Current stock display
- [ ] New physical stock input (number, min 0)
- [ ] Reason input (required, min 5 chars) — e.g., "Received shipment", "Damaged goods"
- [ ] Preview: old stock → new stock
- [ ] Save button → PATCH `/api/v1/admin/inventory/:variantId`
- [ ] Confirmation for significant changes (>50% reduction)

### Low Stock Alerts
- [ ] Dedicated view/section for variants below reorder threshold
- [ ] `GET /api/v1/admin/inventory/low-stock`
- [ ] Display: variant, current available, threshold, suggested reorder quantity

### Audit Log
- [ ] View audit history per variant:
  - [ ] Date, action, old stock, new stock, reason, performed by
  - [ ] Paginated list

### State: `AdminInventoryStore`
- [ ] `inventory` signal, `filter` signal, `cursor` signal, `isLoading` signal
- [ ] Methods: `loadInventory(filter)`, `adjustStock(variantId, newStock, reason)`, `loadLowStock()`, `loadAuditLog(variantId)`

### Tests
- [ ] Component test: table renders with stock levels
- [ ] Component test: stock adjustment form validation
- [ ] Component test: low stock filter
- [ ] Component test: color-coded stock status

---

## Feature: Admin Users

### Component: `AdminUsersComponent`
- [ ] Data table of all users:
  - [ ] Columns: name, email, role badge, status badge, email verified, created date
  - [ ] Filters: role (customer, seller, admin), status (active, suspended, deactivated)
  - [ ] Search by name or email
  - [ ] Sort by: name, email, created date
  - [ ] Cursor pagination
- [ ] Row click → user detail dialog/page

### User Detail Dialog
- [ ] Full profile display: name, email, phone, avatar, role, status
- [ ] Role management:
  - [ ] Change role select (customer, seller, admin)
  - [ ] Confirmation dialog for role changes
- [ ] Status management:
  - [ ] Suspend/Deactivate/Reactivate buttons
  - [ ] Confirmation dialog with reason input
- [ ] Address list (read-only)
- [ ] Order history summary (count, total spent)

### Seller Approval
- [ ] Pending sellers section: list of users with role=seller awaiting approval
- [ ] Approve/Reject buttons with notes
- [ ] Email notification on approval/rejection

### State: `AdminUsersStore`
- [ ] `users` signal, `selectedUser` signal, `filters` signal, `cursor` signal, `isLoading` signal
- [ ] Methods: `loadUsers(filter)`, `updateUserRole(id, role)`, `updateUserStatus(id, status)`, `approveSeller(id)`

### Tests
- [ ] Component test: table renders with filters
- [ ] Component test: role change confirmation
- [ ] Component test: status change with reason

---

## Feature: Admin Analytics

### Component: `AdminDashboardComponent`
- [ ] KPI stat cards (top row):
  - [ ] Total Revenue (with trend vs previous period)
  - [ ] Total Orders (with trend)
  - [ ] Total Customers (with trend)
  - [ ] Average Order Value
  - [ ] Conversion Rate (optional v1)
- [ ] Date range selector: Today, Last 7 Days, Last 30 Days, Last 90 Days, Custom

### Charts Section
- [ ] Revenue over time (line chart): daily/weekly/monthly revenue
- [ ] Orders by status (pie/donut chart): distribution of order statuses
- [ ] Top selling products (bar chart): top 10 products by revenue
- [ ] Sales by category (bar chart): revenue per category

### Recent Activity
- [ ] Recent orders table (last 10): order number, customer, total, status, date
- [ ] Recent reviews pending moderation (last 5)
- [ ] Low stock alerts (top 5)

### API Endpoints (Backend)
- [ ] `GET /api/v1/admin/analytics/summary` — KPI data
- [ ] `GET /api/v1/admin/analytics/revenue?period=7d` — revenue time series
- [ ] `GET /api/v1/admin/analytics/orders-by-status` — status distribution
- [ ] `GET /api/v1/admin/analytics/top-products?limit=10` — top products
- [ ] `GET /api/v1/admin/analytics/sales-by-category` — category breakdown

### State: `AdminAnalyticsStore`
- [ ] `summary` signal — KPI data
- [ ] `revenueChart` signal — time series data
- [ ] `ordersByStatus` signal
- [ ] `topProducts` signal
- [ ] `dateRange` signal
- [ ] `isLoading` signal
- [ ] Methods: `loadSummary(range)`, `loadRevenue(range)`, `loadOrdersByStatus()`, `loadTopProducts()`, `setDateRange(range)`

### Tests
- [ ] Component test: stat cards render with mock data
- [ ] Component test: date range change triggers re-fetch
- [ ] Component test: charts render with data

---

## E2E Tests (Playwright)

### Critical Flows
- [ ] Auth flow: register → login → logout
- [ ] Browse flow: home → catalog → product detail → add to cart
- [ ] Checkout flow: cart → address → payment → order confirmation
- [ ] Admin flow: login as admin → manage products → approve product
- [ ] Search flow: search bar → results → product detail

### Performance
- [ ] Lighthouse CI: score >= 85 for Performance, Accessibility, Best Practices, SEO
- [ ] First Contentful Paint < 1.8s
- [ ] Time to Interactive < 2.5s (4G mobile)
- [ ] Cumulative Layout Shift < 0.1
