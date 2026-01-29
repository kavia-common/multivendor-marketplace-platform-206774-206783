# Multi-vendor Marketplace — PostgreSQL Schema

This container uses PostgreSQL as the primary relational database for a multi-vendor marketplace (vendors/stores/products/orders/reviews, etc.).

## Connecting

Per container convention, connection details live in `db_connection.txt`.

Example:

- `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

## Extensions

- `pgcrypto` is enabled for `gen_random_uuid()`.

## Enums (types)

- `user_role`: `admin | vendor | customer`
- `store_status`: `pending | approved | rejected | suspended`
- `order_status`: `pending | payment_pending | paid | processing | shipped | delivered | cancelled | refunded`
- `payment_status`: `pending | authorized | captured | failed | refunded | cancelled`
- `fulfillment_status`: `unfulfilled | partial | fulfilled | returned`
- `coupon_type`: `percentage | fixed_amount | free_shipping`
- `coupon_scope`: `platform | store`
- `subscription_status`: `active | paused | cancelled | expired`
- `inventory_change_type`: `restock | sale | adjustment | return | cancel`

## Core identity

### `users`
Primary identity table.

Key fields:
- `id` (uuid PK)
- `email` (unique)
- `password_hash` (nullable; can be managed by auth provider)
- `role` (enum `user_role`)
- `is_email_verified`, `is_active`
- `metadata` (jsonb)
- `created_at`, `updated_at`

### `vendor_profiles`
One-to-one with `users` for vendor accounts.
- `user_id` unique FK → `users(id)` (cascade delete)

### `customer_profiles`
One-to-one with `users` for customer accounts.
- `user_id` unique FK → `users(id)` (cascade delete)
- `default_shipping_address_id` FK → `addresses(id)` (set null)
- `default_billing_address_id` FK → `addresses(id)` (set null)

### `addresses`
Reusable shipping/billing addresses.
- Optional `user_id` FK → `users(id)` (set null)

## Stores / catalog

### `stores`
Vendor storefronts, with approval workflow.
- `owner_vendor_id` FK → `vendor_profiles(id)` (cascade delete)
- `slug` unique (global)
- `status` enum `store_status`

### `store_members`
Optional staff membership for stores.
- unique `(store_id, user_id)`

### `categories`
Product categories (self-referencing tree).
- `parent_id` FK → `categories(id)` (set null)

### `products`
Store-owned product entity.
- `store_id` FK → `stores(id)` (cascade delete)
- store-scoped unique `(store_id, slug)`
- optional `category_id` FK → `categories(id)` (set null)
- `attributes` jsonb (flexible attributes)

### `product_variants`
Sellable variants/SKUs per product.
- `product_id` FK → `products(id)` (cascade delete)
- unique `(product_id, sku)`
- price fields stored as integer cents

### `product_images`
Images at product or variant level.
- `product_id` FK → `products(id)` (cascade)
- `variant_id` FK → `product_variants(id)` (set null)

## Inventory

### `inventory_items`
One row per variant.
- `variant_id` unique FK → `product_variants(id)` (cascade)

### `inventory_movements`
Append-only audit for inventory changes.

## Cart

### `carts`
One active cart per customer.
- unique `(customer_id)`

### `cart_items`
Cart lines.
- unique `(cart_id, variant_id)`
- `unit_price_cents` is a snapshot at time of add/update

## Orders / payments / fulfillment

### `orders`
Customer order (can contain items from multiple stores).
- `order_number` unique
- status fields: `status`, `payment_status`, `fulfillment_status`
- totals stored in cents: `subtotal_cents`, `discount_cents`, `shipping_cents`, `tax_cents`, `total_cents`

### `order_store_groups`
One per `(order_id, store_id)` to track per-vendor totals & fulfillment.
- unique `(order_id, store_id)`

### `order_items`
Line items with snapshots of `sku/title/unit_price_cents`.
- references to products/variants are optional (`SET NULL`) to preserve history

### `payments`
Provider-agnostic payment records.
- unique `(provider, provider_payment_id)` (when provider_payment_id is present)

### `shipments`
Tracking per store-group.

## Coupons

### `coupons`
- `code` unique
- `scope`: platform-wide or store-specific
- CHECK ensures store-scoped coupons have `store_id`, platform ones must have `store_id IS NULL`

### `order_coupons`
Join table of coupons applied to orders.

## Reviews & engagement

### `reviews`
Product reviews.
- optional `order_item_id` can link to a purchase
- `is_verified_purchase` flag supported

### `wishlists`, `wishlist_items`
Basic wishlist functionality.

### `recommendation_events`
Event stream to support recommendations / AI features.

## Admin / security

### `audit_logs`
General-purpose audit trail for admin/security actions.

## Timestamps / triggers

A trigger function maintains `updated_at`:

- `set_updated_at()` and per-table triggers for:
  - `users`, `vendor_profiles`, `customer_profiles`, `addresses`, `stores`
  - `products`, `product_variants`, `inventory_items`
  - `carts`, `cart_items`
  - `orders`, `order_store_groups`, `payments`, `shipments`, `reviews`

## Indexes (non-exhaustive)

Performance indexes exist for common access patterns:
- `users(role)`
- `stores(owner_vendor_id)`
- `products(store_id)`, `products(category_id)`
- `orders(customer_id)`
- `order_store_groups(store_id)`
- `order_items(order_id)`, `order_items(store_id)`
- `reviews(product_id)`
- `cart_items(cart_id)`
