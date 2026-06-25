# Komodo UI — TODO

> **Current Version:** V1

Priority guide: **[H]** = blocking real backend simulation · **[M]** = important, not blocking · **[L]** = low priority (docs, testing)

Sections are ordered by dependency — auth must come before cart, cart before checkout, etc.

---

## V1 (Current)

### Auth Flow
> All auth endpoints return 501. No JWT handling exists. Protected routes are unprotected.

- **[L]** Wire `komodo-access-api` `GET /v1/me/access` for component-level feature gating — once access-api ships the public route, fetch the user's active flags + role from BFF on session load and store in `UserState`; gate Svelte components on the flags instead of client-side `if (env === 'prod')` checks; blocked on access-api V2 route implementation
- **[H]** Implement `POST /api/user/login` BFF route — exchange credentials with `komodo-auth-api`; store JWT in `httpOnly; Secure; SameSite=Strict` cookie
- **[H]** Implement `POST /api/user/logout` BFF route — call `komodo-auth-api` `/oauth/revoke`, clear session cookie
- **[H]** Implement `POST /api/user/register` BFF route — create user via `komodo-user-api` `POST /v1/profile`
- **[H]** Add JWT validation to root `+layout.server.ts` — validate cookie on every server load, hydrate session into `event.locals`
- **[H]** Build `/login` page — login form wired to `/api/user/login`; redirect to previous route on success
- **[H]** Build `/signup` page — registration form wired to `/api/user/register`
- **[H]** Add `+layout.server.ts` guards to `/profile`, `/profile/settings`, and `/checkout` that redirect unauthenticated users to `/login`
- **[H]** Build `src/lib/server/auth.ts` — `AuthClient` wrapping `komodo-auth-api` (token exchange, revoke, introspect)
- **[M]** Wire `UserState` to server-loaded session — currently just sets a boolean flag with no data
- **[M]** Implement `GET /api/user/preferences` + `PUT /api/user/preferences` BFF routes — proxy to `komodo-user-api`
- **[M]** Build combined signup/login page — single email entry; check email existence via BFF, then render login form (password) or signup form (full registration) in-place; no separate `/login` and `/signup` routes needed

### Auth API alignment (2026-06-12 — auth-api PRD/HLD frozen; see `apis/komodo-auth-api/docs/`)
> **Komodo is passwordless: passkeys (WebAuthn) + email OTP. No password forms anywhere.** The
> items above that mention "credentials"/"password" (login BFF route, combined signup/login page)
> must be reworked to OTP + passkey flows before being built.

- **[H]** BFF-owned guest sessions — issue a signed anonymous session cookie (`httpOnly; Secure; SameSite`) for guest UX state (cart continuity); auth-api never mints guest tokens (decision 2026-06-12); pass the session ID to the cart BFF for guest carts and merge-on-login
- **[H]** Token response migration camelCase → snake_case (`access_token`, `token_type`, `expires_in`) for all OTP-verify/token consumers
- **[H]** Audit guest-OTP assumptions — `POST /v1/otp/verify` now returns 401 (account not found) for any email without a resolved `USER#<id>`; route guest checkout away from OTP verify entirely
- **[H]** Passkey ceremonies (V1) — WebAuthn register/login via `navigator.credentials` against auth-api `/v1/passkeys/*` begin/complete endpoints; registration requires an authenticated (OTP-verified) session; endpoints land with auth-api TODO Phase 3b
- **[M]** Login page = email entry → OTP code entry or passkey assertion; no password fields
- **[L]** Social IdP (Google/Apple) via `authorization_code` is V2 — do not build against `GET /v1/oauth/authorize` (returns 501)

> **Read-only audit (2026-06-16, auth-api Phase 5 cross-repo closure).** Findings against current code:
> - **Token migration (#35) is a regenerate, not a hand-edit.** `scripts/gen-types.ts` already maps
>   `auth → apis/komodo-auth-api/openapi.yaml`; the committed `src/lib/types/generated/auth.ts` is
>   **stale** — camelCase (`accessToken`/`tokenType`/`expiresIn`/`grantType`/`refreshToken`), paths
>   missing the `/v1` prefix (`/oauth/token`), and it still surfaces `authorization_code`/`redirectUri`
>   as active. Re-running the type-gen against the now-snake_case OpenAPI 3.1 spec flips all token
>   fields to snake_case (delivering #35) and drops the V2 `authorization_code` surface in one shot.
>   Do the regenerate first; everything below cascades from it.
> - **Zero current blast radius.** `src/lib/server/auth.ts` is **empty (0 bytes)** and no live code
>   consumes the token types — only `src/lib/types/auth.ts` re-exports them. There is no OTP-verify
>   call wired anywhere yet. So #35 and #36 are **forward constraints to honor when the `AuthClient`
>   (#24, "Build `src/lib/server/auth.ts`") is built**, not present-day breakages — nothing to migrate
>   or 401-audit until that client exists.
> - **[H] NEW — bare-UUID subject standard (auth-api decision, 2026-06-16).** JWT `sub` and every
>   user identifier auth-api puts on the wire are **bare UUIDs** — never `USER#`-prefixed (the `USER#`
>   in #36 is an internal auth-api/DynamoDB key detail the UI never sees). When the BFF hydrates a
>   session from the token, treat `sub` as a bare UUID; do not add/strip a `USER#` prefix.

---

## Product Pages
> Server load works (shop-items-api wired). The page itself is empty.

- **[H]** Build `/shop/[id]/+page.svelte` — render product images, name, price, description, variant selector, and add-to-cart using data already returned by `+page.server.ts`
- **[H]** Implement `ShopItem.svelte` — reusable product card for listings, search results, and homepage
- **[H]** Implement `PricingLabel.svelte` — price with sale/original price and currency formatting
- **[H]** Implement `QuantitySelector.svelte` — increment/decrement with min=1, max=stock
- **[H]** Implement `AddToCart.svelte` — calls cart BFF and updates `CartState`; handles out-of-stock state
- **[M]** Build `/shop/service/[id]` page — same pattern as product; wire `GET /api/shop/service/[id]` to `komodo-shop-items-api`
- **[M]** Build `/shop/service/repair/[id]` page — repair service detail: accepted device types, turnaround estimate, pricing, warranty on repair, and "Book Repair" CTA
- **[M]** Build repair booking flow — multi-step form: (1) device info + serial number, (2) issue description, (3) inbound shipping label generation, (4) confirmation; wire to `komodo-order-reservations-api`
- **[M]** Fix `Header.svelte` nav links — currently point to `/products` and `/services` (routes that don't exist)
- **[M]** Fix cart badge in `Header.svelte` — hardcoded to "0"; bind to `CartState.itemCount`
- **[M]** Display certification badges on product/service detail pages — render from `ShopItem` data
- **[M]** Display top 6 key features per product/service — structured feature highlight section on detail page
- **[M]** Display recommended products/tools — wire to `komodo-shop-items-api` `GET /suggestions` on detail + cart pages
- **[M]** Display real-time ecom stats on product detail pages — wire to `komodo-statistics-api` public endpoints via BFF: show "X users have this in cart" (`GET /v1/stats/items/{itemId}/in-cart`), "Y people bought this recently" (`GET /v1/stats/items/{itemId}/recently-bought`), and "frequently bought with" pairings (`GET /v1/stats/items/{itemId}/frequently-bought-with`); implement `GET /api/shop/stats/[itemId]` BFF route; display as social-proof callouts on the product page

---

## Home Page
> Renders a Hero with hardcoded Unsplash images. No real content.

- **[H]** Build featured products section — call `komodo-shop-items-api` or use curated static list to show real products using `ShopItem.svelte`
- **[M]** Replace hardcoded Unsplash images with real product/brand imagery
- **[M]** Build `Footer.svelte` — navigation links, legal, social
- **[M]** Build scrolling promotional banner component — configurable slides with CTA links, auto-play with pause-on-hover

---

## Cart
> All cart BFF routes return 501. `CartState` has structure but no methods.

- **[H]** Build `src/lib/server/cart.ts` — `CartClient` wrapping all `komodo-cart-api` guest + auth endpoints
- **[H]** Implement `GET /api/shop/cart` BFF route — fetch cart from `komodo-cart-api` (`GET /v1/cart` for auth, `GET /cart/{cartId}` for guest)
- **[H]** Implement `POST /api/shop/cart` BFF route — add item; call `komodo-cart-api` `POST /v1/cart/items`
- **[H]** Implement `PUT /api/shop/cart` BFF route — update quantity; call `komodo-cart-api` `PUT /v1/cart/items/{itemId}`
- **[H]** Implement `DELETE /api/shop/cart` BFF route — remove item or clear; call `komodo-cart-api` `DELETE /v1/cart/items/{itemId}`
- **[H]** Add `addItem()`, `updateQuantity()`, `removeItem()`, `clear()`, and `subtotal` derived to `CartState`
- **[H]** Persist guest cart ID in `localStorage`; pass `X-Session-ID` header on guest cart requests
- **[H]** Build `/shop/cart` page — line items, quantity controls, subtotal, proceed-to-checkout CTA
- **[M]** Call `POST /v1/cart/merge` after login with stored `guest_cart_id` to merge guest → authenticated cart
- **[M]** Add optimistic UI for add/remove — update `CartState` immediately, reconcile with server response

---

## Checkout Flow
> All 3 checkout pages are empty. Checkout BFF returns 501.

- **[H]** Build `/checkout` page — 3-step form: (1) shipping address, (2) payment method, (3) review & confirm
- **[H]** Implement `POST /api/checkout` BFF route — call `komodo-cart-api` `POST /v1/cart/checkout` for token, then `komodo-order-api` `POST /v1/orders` to place the order
- **[H]** Build `/checkout/complete` page — order confirmation with order ID, summary, and estimated delivery
- **[H]** Implement `CheckoutForm.svelte` — multi-step form component with address, payment, and confirmation steps
- **[H]** Build `src/lib/server/checkout.ts` — orchestrates cart-api checkout token + order-api order placement
- **[L]** Confetti animation behind order complete modal — celebratory burst on `/checkout/complete`; honor `prefers-reduced-motion`
- **[M]** Pre-fill checkout address from user's saved addresses (`komodo-user-api` `GET /v1/addresses`)
- **[M]** Pre-fill payment from user's saved methods (`komodo-user-api` `GET /v1/payments`)
- **[M]** Wire `POST /api/payments/intent` BFF route — create payment intent via `komodo-payments-api`
- **[M]** Implement `PaymentProcessors.svelte` — credit card form (Stripe Elements or equivalent)
- **[M]** Add payment plan option to checkout — display available installment plans at payment step; call `POST /api/payments/plans` via BFF after order is placed if user selects a plan
- **[M]** Build `/profile/payments/plans` page — list active payment plans with installment schedule, next due date, remaining balance, and status badge
- **[M]** Add sliding swipe-to-confirm submit button on order placement — draggable slider component that must be fully dragged to trigger order submission; prevents accidental orders; implement as `SlideToConfirm.svelte` in `src/lib/components/`

---

## User Profile
> Profile pages are empty stubs. All user BFF routes return 501.

- **[H]** Build `src/lib/server/user.ts` — `UserClient` wrapping `komodo-user-api` (profile, addresses, payments, preferences)
- **[H]** Implement `GET/PUT/DELETE /api/user/profile` BFF routes — proxy to `komodo-user-api`
- **[H]** Build `/profile` page — display name, email, order history summary, account actions
- **[M]** Build `/profile/settings` page — editable profile form, address book, saved payment methods, preferences toggles
- **[M]** Implement `GET/POST/PUT/DELETE /api/address` BFF routes — confirm correct target (`komodo-user-api` for saved addresses; `komodo-address-api` for validation only)
- **[M]** Implement payment method CRUD in profile settings — wire to `komodo-user-api` `GET/PUT/DELETE /v1/payments`
- **[M]** Implement `ProfileSettings.svelte`, `PaymentMethods.svelte` components

---

## Orders
> Order BFF routes all return 501. No order pages exist.

- **[M]** Build `src/lib/server/order.ts` — `OrderClient` wrapping `komodo-order-api`
- **[M]** Implement `GET /api/orders/history/[userID]` BFF route — proxy to `komodo-order-api` `GET /v1/orders`
- **[M]** Implement `GET /api/orders/[id]/details` BFF route — proxy to `komodo-order-api` `GET /v1/orders/{orderId}`
- **[M]** Implement `POST /api/orders/[id]/returns` BFF route — proxy to `komodo-order-api` (`POST /v1/orders/returns`); returns are folded into order-api, no standalone `order-returns-api`
- **[M]** Build order detail page — items, status timeline, tracking info, return/cancel CTA
- **[M]** Implement `OrderHistory.svelte` — table of past orders with status badges

---

## Search
> `/shop/search` page is empty. Search BFF returns 501.

- **[M]** Build `src/lib/server/search.ts` — `SearchClient` wrapping `komodo-search-api`
- **[M]** Implement `GET /api/shop/search` BFF route — proxy to `komodo-search-api`
- **[M]** Build `/shop/search` page — results grid, filters sidebar, pagination
- **[M]** Implement `ItemFiltering.svelte` — category, price range, rating filter controls
- **[M]** Implement `Pagination.svelte` — page controls for search results
- **[M]** Wire search input in `Header.svelte` to navigate to `/shop/search?q=...`
- **[M]** Implement `Search.svelte` — debounced typeahead with instant results dropdown

---

## Component Library Strategy
> The stack is pre-configured for shadcn-svelte (CVA + clsx + tailwind-merge + lucide-svelte + tw-animate-css + Tailwind v4 + Svelte 5) — but **no library is mandated**. Pick per component as design preferences emerge. Adopting one library doesn't preclude others on the same page.
>
> File scaffolds for many of the components below already exist as 0-byte stubs in `ui/src/lib/components/{primitives,forms,feedback,navigation,overlays,display,...}` — implementation is "fill the stub", not "create the file."

**Library options to evaluate:**
- **shadcn-svelte** — copy-in primitives, Tailwind-styled, Bits UI underneath; full ownership/mutation
- **Bits UI** — headless primitives only; what shadcn wraps. Use directly if you want zero opinion on styling.
- **Melt UI** — alternative builder-pattern headless lib
- **Skeleton UI** — opinionated Tailwind-native component library (broader catalog, more aesthetic lock-in)
- **Flowbite Svelte** — Tailwind component library; large catalog, design language is its own
- **svelte-headlessui** — Radix-style headless primitives port
- **Specialty libs** — `svelte-dnd-action` (drag/drop), `embla-carousel-svelte` (carousels), `vaul-svelte` (mobile drawers/action sheets), `media-chrome` or `Plyr` (video), `paneforge` (resizable panels), TanStack Table (sort/expand/select)
- **Custom** — hand-built when no off-the-shelf primitive fits or styling needs are highly bespoke

**Per-component viable options** (informational; not enforced):

| Component | Viable options |
|-----------|---------------|
| Input, Select, Checkbox, Radio, Switch, Slider | shadcn-svelte · Bits UI · Melt UI · Skeleton · Flowbite · custom |
| Modal/Dialog, Popover, Tooltip, Hover Card | shadcn-svelte · Bits UI · svelte-headlessui · custom |
| Toast, Alert, Snackbar | shadcn-svelte (Sonner) · Skeleton (Toast) · custom |
| Tabs, Accordion, Breadcrumbs | shadcn-svelte · Bits UI · Skeleton · Flowbite · custom |
| Sidebar, Hamburger, Dropdown Menu | shadcn-svelte (Sheet + DropdownMenu) · Bits UI · custom |
| DatePicker, DateRangePicker, TimePicker | shadcn-svelte (Calendar) · Bits UI · `svelte-flatpickr` · custom |
| Combobox, PredictiveText, SearchSuggestions | shadcn-svelte (Command) · Bits UI · Melt UI · custom |
| Table (generic/sorted/expandable/checkbox) | shadcn-svelte shell + TanStack Table · Skeleton · Flowbite · custom |
| Carousel, ScrollingBanner, StoryPhotos | shadcn-svelte (Carousel) · `embla-carousel-svelte` · `swiper` · custom |
| ActionSheet, mobile Drawer | shadcn-svelte (Drawer/`vaul-svelte`) · custom |
| ResizingCanvas, Resizable panels | shadcn-svelte (Resizable/`paneforge`) · custom |
| Spinner, Skeleton (loading) | shadcn-svelte · lucide `Loader` icon · Skeleton lib · custom CSS |
| Badge, Pills, BestSellerTag | shadcn-svelte (Badge) · Skeleton · custom (likely thin) |
| AddToCart, QuantitySelector, PricingLabel, ShopItem | wrap shadcn `Button`/`Input` · custom (business logic dominates) |
| CurrencyInput, MaskedInput, ConfirmInput, PasswordInput | wrap shadcn `Input` + formatter · custom |
| Workflow, Timeline, Tile, OperatingHours | compose shadcn `Card` + `Separator` · custom |
| BulkSelect, BulkEditBar | shadcn-svelte `Checkbox` + custom action bar · custom |
| LongPressButton, SlideToConfirm | wrap shadcn `Button` + custom interaction · custom |
| DragDropList | `svelte-dnd-action` · custom (no shadcn equivalent) |
| VideoPlayer | `media-chrome` · `Plyr` · `vidstack` · custom |
| WheelDiscount, 3D modeling | custom (Threlte/GSAP — already deps) |
| PercentageGauge | custom SVG · `svelte-radial-gauge` |
| CallButton, ShareButton, Advertisement slot | custom (too thin to justify a primitive) |
| Fun Mode (cursor/confetti/audio) | custom + `canvas-confetti` |

> One pragmatic adoption path: install the shadcn-svelte commodities (Button, Input, Dialog, Tooltip, Popover, DropdownMenu, Sheet, Tabs, Accordion, Calendar, Switch, Slider, Sonner, Badge, Skeleton, Pagination, Carousel, Resizable, Drawer, HoverCard, Command, Table) in one batch, then evaluate Skeleton/Flowbite/custom on case-by-case basis as design surfaces.

---

## Core UI Components
> 69 of 70 components are empty stubs. These block every page above.

- **[H]** `Input.svelte` — text input with label, validation error state, helper text
- **[H]** `Select/DropdownSelector.svelte` — controlled select with options list
- **[H]** `CheckboxGroup.svelte` / `RadioGroup.svelte` — form group controls
- **[H]** `Spinner.svelte` / `Skeleton.svelte` — loading states for async content
- **[H]** `Toast.svelte` / `Alert.svelte` — error and confirmation feedback; wire to `NotificationsState`
- **[M]** `Modal.svelte` — overlay dialog for confirmations and inline forms
- **[M]** `Badge.svelte` — status labels (order status, stock status)
- **[M]** `Tooltip.svelte` — hover hints
- **[M]** `Breadcrumbs.svelte` — navigation trail; wire to `AppState.breadcrumbs` (restoration is a TODO in AppState)
- **[M]** `Sidebar.svelte` — collapsible side panel; wire to `AppState.sidebarOpen`
- **[M]** `Hamburger.svelte` + `DropdownMenu.svelte` — mobile nav and account menu in Header
- **[M]** `DatePicker.svelte` — for service scheduling flows
- **[L]** `Tabs.svelte`, `Accordion.svelte` — secondary layout components
- **[L]** `VideoPlayer.svelte` (`<video>`) — controlled player wrapper with poster, captions, and lazy-load
- **[L]** `OperatingHours.svelte` (`<hours-operation>`) — weekly hours table; highlights "open now"; timezone-aware
- **[L]** `CallButton.svelte` (`<button-call>`) — `tel:` deep-link button with click-to-call analytics hook
- **[L]** `ShareButton.svelte` (`<button-share>`) — native Web Share API fallback to copy-link
- **[L]** `ToggleSwitch.svelte` (`<button-switch>`) — iOS-style binary switch with keyboard + ARIA
- **[L]** `PopoutMenu.svelte` (`<menu-popout>`) — anchored floating menu with focus trap and outside-click close
- **[L]** `Combobox.svelte` (`<combobox>`, `<option>`) — searchable fixed-options select (e.g. US states)
- **[L]** `ConfirmInput.svelte` (`<input-confirm>`) — paired inputs requiring identical values; horizontal or vertical
- **[L]** `CurrencyInput.svelte` (`<currency>`) — auto-formats to currency, optional ranges (`$10–150`), increments, multi-currency
- **[L]** `CurrencyText.svelte` — display-only currency formatter (read-only counterpart)
- **[L]** `DateRangePicker.svelte` (`<date-range>`) — preset ranges (7/30/90d), past/future toggle, weekend-disable, min/max
- **[L]** `DateTimeText.svelte` (`<date-time>`) — locale-aware date+time formatter
- **[L]** `PercentageGauge.svelte` (`<chart-gauge>`) — radial gauge meter with percentage fill
- **[L]** `Timeline.svelte` (`<timeline>`, `<timeline-item>`) — vertical or horizontal connected event sequence
- **[L]** `MaskedInput.svelte` (`<input-masked>`) — masks all but last entered char (password/PII pattern)
- **[L]** `Menu.svelte` (`<menu>`, `<menu-option>`) — two variants: simple (compact vertical) and complex (expanded readable)
- **[L]** `Popover.svelte` (`<popover>`) — floating text box anchored to a UI element
- **[L]** `Snackbar.svelte` (`<snackbar-alert>`) — bottom-corner fly-in alert (success/warn/error); shorter-lived than Toast
- **[L]** `Tile.svelte` (`<tile>`) — large card with formatted heading, subhead, body for high-readability layouts
- **[L]** `Workflow.svelte` (`<workflow>`, `<flow-section>`, `<flow-header>`) — grouped repeated rows with header/subhead/body and optional row buttons
- **[L]** `Table.svelte` family (`<table>`, `<row>`, `<col>`, `<cell>`) — generic, sorted, expandable (click-to-expand), and checkbox variants; static or dynamic population with "Show More" trigger
- **[L]** `TimePicker.svelte` (`<time-picker>`) — hours/minutes/seconds picker
- **[L]** `ActionSheet.svelte` (`<action-sheet>`) — iOS-style vertical row picker for large data sets
- **[L]** `PredictiveText.svelte` (`<prediction-group>`, `<prediction>`) — autocomplete + suggestion API integration; auto-fill on selection
- **[L]** `SearchSuggestions.svelte` (`<suggestion-group>`, `<suggestion>`) — combined static + API results; sortable and limited
- **[L]** `BulkSelect.svelte` (`<select>`, `<select-bar>`, `<option>`) — multi-row selection with toggled action bar (Apple Photos pattern)
- **[L]** `VolumeSlider.svelte` — volume icon with on-hover slider
- **[L]** `Pills.svelte` (`<pill-group>`, `<pill>`) — compact tag/filter chips
- **[L]** `AddressDisplay.svelte` (`<address>`) — auto-formats structured address into readable text
- **[L]** `LongPressButton.svelte` — hold-to-confirm button (Apple Pay pattern); prevents accidental destructive actions
- **[L]** `PasswordInput.svelte` — text input with hide/show toggle
- **[L]** `DragDropList.svelte` (`<drag-drop-group>`, `<drag-drop-row>`) — reorderable rows via drag and drop with keyboard fallback
- **[L]** `StoryPhotos.svelte` — full-screen swipeable photo viewer (Instagram Stories pattern)
- **[L]** `ResizingCanvas.svelte` (`<canvas>`) — auto-scales viewport to data set size; auto-scales images to visible bounds
- **[L]** `DefaultImage.svelte` — image with built-in fallback (placeholder + optional goofy/playful image on load failure)
- **[L]** `SynopsisModal.svelte` (`<summary>`) — quick-view popover with item summary on hover
- **[L]** `Advertisement.svelte` (`<advertisement>`) — slot for 1st/3rd-party ad injection; lazy-loaded
- **[L]** `BestSellerTag.svelte` (`<tag>`) — overlay tag for product cards (Best Seller / Featured / Customer Favorite)
- **[L]** `CustomerRating.svelte` (`<cust-rating>`) — popularity / rating display with filter integration
- **[L]** `WheelDiscount.svelte` (`<wheel-discount>`) — animated wheel-spinner popup that generates discounts

---

## State Management
> Stores are declared but have no data flow end-to-end.

- **[H]** `UserState` — hydrate `#profile` from root server load (`event.locals.user`); wire `login()`/`logout()` to real session data
- **[H]** `CartState` — implement `addItem()`, `updateQuantity()`, `removeItem()`, `clear()`, `itemCount` and `subtotal` derived; sync with cart BFF on page load
- **[M]** `NotificationsState` — implement `add()`, `dismiss()`, `clear()`; drive `Toast.svelte`
- **[M]** `item.svelte.ts` — add selected variant, quantity, and loaded product state for product detail page
- **[M]** Fix `AppState` breadcrumb restoration — currently a `// TODO` comment

---

## Payments BFF
> All payment routes return 501.

- **[M]** Build `src/lib/server/payments.ts` — `PaymentsClient` wrapping `komodo-payments-api`
- **[M]** Implement `POST /api/payments/intent` — create payment intent; proxy to `komodo-payments-api`
- **[M]** Implement `GET/POST/DELETE /api/payments/methods` — saved card management; proxy to `komodo-payments-api`
- **[M]** Implement `GET /api/payments/transactions` — transaction history; proxy to `komodo-payments-api`

---

## Security
> Auth not yet implemented — these become relevant once sessions exist.

- **[H]** Set `httpOnly; Secure; SameSite=Strict` on session cookie when auth is wired
- **[M]** Add Content Security Policy headers in `hooks.server.ts` — restrict script/style/img sources
- **[M]** Add rate limiting on BFF write routes (`/api/user/login`, `/api/checkout`, `/api/payments/*`)
- **[M]** Add `.env.example` documenting all required env vars (`AUTH_API_URL`, `USER_API_URL`, `CART_API_URL`, `SHOP_ITEMS_API_URL`, etc.)
- **[M]** Implement BFF-layer field encryption/decryption for sensitive PII — encrypt fields in the SvelteKit server layer (never in the browser) using keys fetched from AWS Secrets Manager before passing to backend APIs; decrypt on read
- **[L]** Add Subresource Integrity (SRI) hashes for any external CDN assets
- **[L]** Replace hardcoded Unsplash URLs with a proxied image route to avoid leaking behavior to third-party

---

## Adapter & Build
> Currently `adapter-static`. Must switch to `adapter-node` for real server-side BFF calls to work in production.

- **[H]** Switch `svelte.config.js` from `adapter-static` → `adapter-node` when backend services are live (comment swap is already prepared in the file)
- **[M]** Add production `Dockerfile` optimized for `adapter-node` (current one is dev-only hot-reload)
- **[M]** Set `ORIGIN` env var for `adapter-node` CSRF validation

---

## Testing
- **[L]** Write unit tests for all BFF API routes (happy path + error cases) — Vitest
- **[L]** Write component tests for all implemented UI components — `@testing-library/svelte`
- **[L]** Write integration tests for BFF → API service flows — Vitest with MSW or real LocalStack endpoints
- **[L]** Write E2E tests for core flows: login → add to cart → checkout → order confirmation — Playwright
- **[L]** Add client-side error boundary logging to `/api/log` route
- **[L]** Implement log rehydration from `localStorage` in `handler.worker.ts` — replay any queued log entries on worker init (`lib/logger/common/handler.worker.ts:11`)
- **[L]** Add Core Web Vitals tracking (LCP, FID, CLS)

---

## Analytics & Telemetry
> V1 strategy: structured logs land in CloudWatch per service. No separate analytics API or ingestion Lambda. Data lake is a future concern.
> Client-side logging uses `komodo-forge-sdk-ts` `logger` — single worker thread (Web Worker in browser, node-worker on Fargate, fetch on Lambda). All four channels (runtime, clickstream, interaction, telemetry) route through one handler. Browser channels are no-ops until `addListener` is called, keeping the main thread free.

- **[M]** Wire `logger` from `komodo-forge-sdk-ts` in UI — call `logger.init(...)` in root `+layout.ts`; register `addListener.runtime` pointing at `/api/log` BFF route; defer clickstream/interaction/telemetry listeners until backend is live
- **[M]** Implement `/api/log` BFF route — receives log events from the SDK's worker, writes a structured `slog`-compatible JSON line server-side so they land in the SvelteKit CloudWatch stream; Node backend can log freely, browser traffic is batched by the worker before hitting this route

---

## Support & Chat
> Support chat UI not yet implemented.

- **[M]** Build chat widget — send/receive messages against `komodo-support-api` `/v1/chat/message`; guest and authenticated modes
- **[M]** Display full chat session history — fetch prior messages on widget open; paginate if session is long
- **[L]** Wire escalation CTA in chat widget — call `/v1/chat/escalate` and confirm ticket creation to user

---

## Founders Page
> Static brand page — no backend dependency.

- **[L]** Build `/founders` page — static page with brand story, founder bios, and company background; no server load function needed; content can be hardcoded initially and moved to a CMS later

---

## Documentation
- **[L]** Add `.env.example` with all required env vars and descriptions
- **[L]** Document component API (props, slots, events) for shared components once implemented
- **[L]** Update `docs/architecture.md` once BFF wiring is complete

## Performance — Homepage
> Goal: LCP < 150ms. Pre-render + hero image optimization account for ~80% of the gain.

- **[H]** Pre-render `/` at build time — homepage content is not user-specific; statically render at build and serve from CloudFront with zero server latency; trigger a CloudFront invalidation + rebuild when shop-items data changes (event-bus `shop_item.updated` → CI job)
- **[H]** Optimize hero image for LCP — convert to AVIF/WebP, size to viewport, add `fetchpriority="high"` on the `<img>`, host on CloudFront (not a third-party origin), add `<link rel="preload">` in `<head>`; the hero is almost certainly the LCP element and will dominate the score until fixed
- **[M]** Lazy-load Threlte/GSAP/Lenis after LCP — none of these should be in the initial bundle; dynamic-import them in `onMount` after first paint so they don't block parse or delay interactivity
- **[M]** Parallel BFF fetch + Redis cache for featured products — in `+page.server.ts`, fetch all homepage data in parallel; add a 60s Redis cache in front of the shop-items-api call so DynamoDB is only hit once per minute across all users
- **[L]** Add `<link rel="preconnect">` hints for API and CDN origins in `app.html` — saves ~100ms TLS handshake on first asset fetch

---

## SDK Extractions (komodo-forge-sdk-ts)

- **[M]** Replace UI's `src/lib/server/common/client.ts` with `BaseAdapter` from `komodo-forge-sdk-ts` (`src/api/adapters/base/adapter.ts`) — extraction is already done in the SDK; UI just needs to delete the duplicate and re-import. Verify RFC 7807 `ProblemDetail` parity before deleting.
- **[M]** Add versioned barrel exports to `komodo-forge-sdk-ts` dist — `dist/index.ts` re-exports from the current stable version (e.g. `v1`) as the default import path; older/newer versions remain importable via `dist/v1`, `dist/v2`, etc.; forces a clear default while preserving backward and forward compatibility

---

## UX Behavior & Polish
> Cross-cutting interaction patterns that touch many pages.

- **[M]** `lastLocation` cookie — persist last visited route on every navigation; restore on return (e.g. after login or external bounce-back). Set in `+layout.ts`/`hooks.client.ts`; honor on `/login` redirect.
- **[M]** Click-outside-to-close utility — Svelte action (`use:clickOutside`) shared by Modal, Popover, PopoutMenu, DropdownMenu; closes UI on outside click and Escape keypress.
- **[M]** Infinite scroll / lazy loader module — `IntersectionObserver`-based loader for product grids, search results, reviews; reusable `<InfiniteScroll>` wrapper.
- **[M]** Pagination smooth-scroll without reload — fade-swap content and scroll target into view rather than full page reload.
- **[M]** Dedupe results in search — collapse repeated products (same SKU, different size/color) into a single result with variant indicator.
- **[M]** Sort by quality — order products lowest→highest quality with a one-click reverse toggle.
- **[L]** Disappearing scroll bar — overlay scrollbar that fades when idle; native + styled fallback.
- **[L]** Disappearing header/footer on scroll — auto-hide chrome on scroll-down, restore on scroll-up to maximize screen real estate.
- **[L]** Edge-bound 4-side navigation — clicking/swiping/scrolling toward a screen edge navigates to a side-bound page or flow (alternative to header nav).
- **[L]** Image-driven variant binding — clicking a product image auto-updates the selected variant (color/finish), keeping image set and option state in sync.
- **[L]** Console startup logo — print a stylized ASCII Komodo logo + build/version banner to `console.log` automatically on customer site load; use CSS console styling (`%c`) for color; place in root `+layout.svelte` `onMount` or `app.html`; skip in test/mock builds if noisy.

---

## Engagement & Social Proof
> Conversion-driving features. Most depend on backend APIs that are scaffolded but not wired.

- **[M]** Real-time inventory live counts on product detail pages — wire `komodo-shop-inventory-api` SSE/WebSocket; show "X left in stock" with live decrement.
- **[M]** "Best Seller" / "Customer Favorites" tags + filters — wire `komodo-shop-items-api` flags and `komodo-statistics-api` popularity feed; renders `BestSellerTag.svelte`.
- **[M]** "Add to Cart" CTA inside customer review section — convert browsing → buying without leaving review context.
- **[M]** Searchable reviews & questions — full-text search over a product's reviews and Q&A; wire to `komodo-loyalty-api` `/v1/items/{itemId}/reviews` (reviews live in loyalty-api, not shop-items-api).
- **[M]** Recommendation tool module — wires to `komodo-shop-items-api` `/suggestions` with user-state context (budget, history, prior carts).
- **[M]** Compare tool module — side-by-side product spec comparison (up to 4 items); persists comparison set in `localStorage`.
- **[M]** Points/Rewards display — wire `komodo-loyalty-api`; show points balance in header, points-earned preview on product/cart pages.
- **[L]** Gift card balance checker — form (number + PIN) → BFF route → `komodo-payments-api`; display balance with redeem CTA.
- **[L]** 3D Virtual Modeling — Threlte-rendered product/location tours with reference objects (coke can, bicycle) for size comparison; deferred until commerce flows are stable.
- **[L]** Wheel-spinner discount popup — uses `WheelDiscount.svelte`; backed by promotions-api code generation.
- **[L]** "Fun Mode" — global toggle (`<button-fun>`) that activates cursor change, ambient music, confetti on actions, and click sounds. Honors `prefers-reduced-motion`.

---

## Tools & Calculators
- **[M]** Shipping cost calculator — standalone widget on product/cart pages; wire `komodo-shipping-api` rate quotes by destination ZIP.
- **[L]** Labor cost calculator (`<calculator>`) — service-pricing estimator (hours × rate × multipliers) for repair / install services.

---

## Internationalization
> No prior i18n infra. Strategy: ship language toggle + per-page i18n key bundle delivered alongside data to avoid a large client-side dictionary.

- **[M]** `Translate.svelte` (`<translate>`) — renders i18n key with current locale; English / Spanish toggle in header drives `LocaleState`.
- **[M]** Per-page i18n bundle endpoint — BFF route (`/api/i18n/[page]`) returns the key/value map for the current page + locale; cached with stale-while-revalidate.
- **[M]** API responses include i18n keys for dynamic text (status labels, error messages, badges) instead of hard-coded strings — UI resolves via the bundle. Coordinate with each backend API service to return i18n keys alongside response data.
- **[L]** Locale persistence — `locale` cookie (`httpOnly: false; SameSite=Lax`) read in `+layout.server.ts` and injected into `event.locals.locale`.
