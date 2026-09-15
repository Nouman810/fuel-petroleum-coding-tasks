# Execution Plan: Product Catalog

## Change Class
FEATURE

## Summary

Add Admin-only CRUD for products (name, category, unit, price) as the standalone catalog that Task 07's ordering flow will read from. No Dealer access, no scoping — this is a flat, network-wide list Admin manages, structurally identical in access-control shape to `Dealer` (`requireRole('ADMIN')`, no `dealerScope`).

## Approach and Trade-offs

**Model:** A new `Product` table with a `category` enum (`ProductCategory`: `MS` | `HSD` | `LUBRICANT`), mirroring the `EquipmentType` enum precedent from Task 05. Category stays a fixed enum (not free text) because Task 07 (orders) and Task 10 (reporting, "by product... and by lubricant specifically") both branch logic on it — a free-text field would let category values drift and break that branching.

**No relations yet.** `Product` has no FK to or from any existing table. `Order` (Task 07) will add its own `productId` FK pointing at `Product` when that task lands — building it now would be speculative, since Order's exact cascade/restrict behavior isn't decided yet.

**Equipment.product stays a plain string.** Task 05 left a note that this field might become a real FK once Product Catalog exists. Confirmed with the user this task does **not** do that migration — it's out of scope, deferred to whenever it's actually needed (most likely alongside Task 07, when Order's own Product FK pattern is decided).

**Full CRUD, including delete.** The raw prompt says "CRUD" explicitly. Unlike `Dealer`/`Station` (which have no delete route in this codebase) or `Equipment` (which has delete with a 409-on-FK-conflict guard), `Product` currently has zero incoming FKs, so there's no FK-conflict case to handle. There is still a not-found case: both `PATCH /products/:id` and `DELETE /products/:id` wrap their `prisma.product.update()`/`.delete()` call in `try/catch` and map Prisma's `P2025` ("record not found") to `404 { error: 'Product not found' }`, matching the same catch shape used in `dealers.js:81-83` (`PATCH /dealers/:id`) and `stations.js:85-88`. Without this catch, Express 5's default error handler would return an unhandled `500` instead of the `404` the test plan requires — `backend/src/app.js` registers no custom error-handling middleware. (Once Task 07 adds `Order.productId`, that task will need to decide whether to add `onDelete: Restrict` and a 409 path here — noted for that task, not handled now.)

**No uniqueness constraint on name.** Not requested by the raw prompt or spec.md; two products can share a name (e.g., different unit/pack sizes are plausible in a real catalog). Not adding an unrequested constraint.

**Price and no negative/zero guard beyond `positive()`.** `price` uses `z.coerce.number().positive()`, matching the `capacity`/`currentMeterReading` numeric-coercion pattern from Task 05 (bare `z.coerce.number()` silently accepts `""` as `0` — always pair coercion with an explicit bound).

## Files to Change

### Backend
- `backend/prisma/schema.prisma` — add `ProductCategory` enum (`MS`, `HSD`, `LUBRICANT`) and `Product` model.
- `backend/prisma/migrations/20260915150000_add_product/migration.sql` — new migration, generated via `npx prisma migrate diff --from-schema-datamodel <old-schema-snapshot> --to-schema-datamodel prisma/schema.prisma --script` (real tool output, not hand-written), then applied locally via `npx prisma migrate deploy`. (The next available timestamp slot: `20260915000000_add_user_auth`, `20260915120000_add_dealer`, `20260915130000_add_station`, `20260915140000_add_equipment` already exist, so this migration must sort after all of them.)
- `backend/src/routes/products.js` (new) — `POST /products`, `GET /products`, `GET /products/:id`, `PATCH /products/:id`, `DELETE /products/:id`. All routes `router.use('/products', authenticate, requireRole('ADMIN'))`, matching `dealers.js`'s router-level guard (no `dealerScope` — Product isn't dealer-owned). `PATCH`/`DELETE` catch `P2025` → `404` (see Approach).
- `backend/src/routes/products.test.js` (new) — test cases per Test Plan below.
- `backend/src/app.js` — add `const productsRouter = require('./routes/products'); app.use(productsRouter);` after the existing router mounts.

### Frontend
- `frontend/src/api/products.js` (new) — `listProducts()`, `getProduct(id)`, `createProduct(payload)`, `updateProduct(id, payload)`, `deleteProduct(id)`. Same `apiFetch` wrapper pattern as `api/dealers.js` (list/get/create/update), plus `deleteProduct(id)` following `api/equipment.js`'s `deleteEquipment(id)` precedent (dealers.js has no delete function) — all ending in `res.json()`, DELETE returns `200` + the deleted row, not a bare `204`.
- `frontend/src/routes/ProductListPage.jsx` (new) — list + inline create-form + inline edit + delete, modeled on `DealerListPage.jsx`'s list/create-form shape combined with `StationEquipmentPage.jsx`'s per-row edit/delete pattern (Product needs edit+delete, which `DealerListPage` doesn't have but `StationEquipmentPage`'s `EquipmentRow` does).
- `frontend/src/routes/ProductListPage.test.jsx` (new) — test cases per Test Plan below.
- `frontend/src/routes/AdminShell.jsx` — add `import ProductListPage from './ProductListPage';`, add `<Route path="products" element={<ProductListPage />} />`, add `<p><Link to="products">Manage Products</Link></p>` to `AdminHome` alongside the existing Dealers/Stations/Equipment links.
- `frontend/src/routes/AdminShell.test.jsx` — two new cases: `/admin/products` resolves to `ProductListPage`'s content; `/admin` home renders a "Manage Products" link (same two-case pattern used for Equipment in Task 05).
- `frontend/vite.config.js` — add `'/products': 'http://localhost:3000'` to the dev proxy map.

### Documentation
- None required. No API-spec or ERD doc file exists in this repo to update (confirmed: `docs/` does not exist at the project root; `spec.md` in Planning_Tasks already documents the `PRODUCT` entity and is not a living doc this task needs to touch).

## Database Schema

```prisma
enum ProductCategory {
  MS
  HSD
  LUBRICANT
}

model Product {
  id        String          @id @default(uuid())
  name      String
  category  ProductCategory
  unit      String
  price     Decimal         @db.Decimal(12, 2)
  createdAt DateTime        @default(now())
  updatedAt DateTime        @updatedAt

  @@map("products")
}
```

Migration SQL (generated via `prisma migrate diff`, folder `backend/prisma/migrations/20260915150000_add_product/migration.sql`):
```sql
-- CreateEnum
CREATE TYPE "ProductCategory" AS ENUM ('MS', 'HSD', 'LUBRICANT');

-- CreateTable
CREATE TABLE "products" (
    "id" TEXT NOT NULL,
    "name" TEXT NOT NULL,
    "category" "ProductCategory" NOT NULL,
    "unit" TEXT NOT NULL,
    "price" DECIMAL(12,2) NOT NULL,
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "products_pkey" PRIMARY KEY ("id")
);
```
(This SQL is a projection of the schema above; the actual committed migration file will be the real `prisma migrate diff` output, not hand-typed, per this repo's established convention.)

## Test Plan

### Backend (`backend/src/routes/products.test.js`)
1. `POST /products` (Admin) creates a product with name/category/unit/price, returns `201` with `Number(price)` compared (Decimal serializes as string).
2. `POST /products` with a malformed body (missing `name`, or `category` not one of the enum values) returns `400`.
3. `POST /products` with `price: ""` (empty string) returns `400` (proves `.positive()` bound, not bare coercion).
4. `POST /products` with a Dealer token returns `403`.
5. `POST /products` with no Authorization header returns `401`.
6. `GET /products` (Admin) returns all created products.
7. `GET /products` with a Dealer token returns `403`.
8. `GET /products/:id` (Admin) returns the product.
9. `GET /products/:id` for a non-existent id returns `404`.
10. `GET /products/:id` with a Dealer token returns `403`.
11. `PATCH /products/:id` (Admin) updates name/category/unit/price, returns `200` with updated fields.
12. `PATCH /products/:id` with a Dealer token returns `403`.
13. `PATCH /products/:id` for a non-existent id returns `404` (via explicit `P2025` catch, not the framework default `500`).
14. `DELETE /products/:id` (Admin) returns `200` with the deleted row's JSON; a subsequent `GET /products/:id` on that id returns `404`.
15. `DELETE /products/:id` with a Dealer token returns `403`.
16. `DELETE /products/:id` for a non-existent id returns `404` (via explicit `P2025` catch, not the framework default `500`).

Run locally: `cd backend && npx jest src/routes/products.test.js --no-cache` (requires the live local Postgres at `C:\Users\it.admin\pg-fuel-petroleum`), then the full suite `npx jest --no-cache`, then re-seed (`npx prisma db seed`) as routine hygiene (each test file scopes its own cleanup to its fixture ids/emails, so the dev seed accounts are not actually at risk from this task's tests, but re-seeding after a full run is harmless and matches established practice from prior tasks).

### Frontend (`frontend/src/routes/ProductListPage.test.jsx`, `frontend/src/routes/AdminShell.test.jsx`)
1. Renders one row per product returned by `listProducts()`, with name/category/unit/price visible.
2. Submitting the create form calls `createProduct` with the entered name/category/unit/price.
3. Submitting an edit on a row calls `updateProduct(id, payload)` with the edited fields.
4. Clicking Delete on a row calls `deleteProduct(id)`.
5. `AdminShell.test.jsx`: `/admin/products` resolves to `ProductListPage`'s content (mocking `api/products.js`).
6. `AdminShell.test.jsx`: `/admin` home renders a clickable "Manage Products" link pointing at `/admin/products`.

Run locally: `cd frontend && npx vitest run`.

### Lint
`cd backend && npx eslint .` and `cd frontend && npx eslint .`, both must exit 0.

## Acceptance Criteria

1. `POST /products` (Admin) creates a product, returns `201`.
2. `POST /products` with a malformed body returns `400`.
3. `POST /products` with `price` as an empty string returns `400` (not silently coerced to `0`).
4. `POST /products` with a Dealer token returns `403`.
5. `POST /products` unauthenticated returns `401`.
6. `GET /products` (Admin) returns all products; Dealer token returns `403`.
7. `GET /products/:id` returns the product for Admin, `404` for a non-existent id, `403` for a Dealer token.
8. `PATCH /products/:id` (Admin) updates fields, returns `200`; Dealer token returns `403`; non-existent id returns `404`.
9. `DELETE /products/:id` (Admin) returns `200` with the deleted row and the id is subsequently `404`; Dealer token returns `403`; non-existent id returns `404`.
10. Frontend product list page renders products, and its create/edit/delete forms call the correct `api/products.js` functions with the correct payload/id.
11. `AdminHome` renders a clickable "Manage Products" link, and `/admin/products` resolves to the product list page.
12. Lint passes with zero errors in both services.
13. CI runs lint + test for both services on the PR, with a live Postgres service for the backend job, and is green.

## Branch

`feature/fuel_petroleum-XXX-product-catalog` (created from `story/3-dealer-fuel-station-management-system`)
