# Product Catalog

## Change Class
FEATURE

## Problem

Dealers will eventually place orders for fuel and lubricants (Task 07), but there is currently no catalog of what's orderable. Admin needs a way to define the product list — what it is, what category it falls into, how it's measured, and what it costs — before ordering can exist.

## Solution

Add an Admin-only CRUD surface for products:
- **Create** a product: name, category, unit of measure, price.
- **List** all products (Admin view).
- **View** a single product.
- **Update** a product's fields.
- **Delete** a product.

Category is a **fixed enum** (`MS` petrol / `HSD` diesel / `LUBRICANT`), not free text, because downstream order and reporting logic (Tasks 07, 10) will branch on it. This mirrors the `EquipmentType` enum precedent from Task 05.

There is no Dealer-facing UI in this task. Dealers will browse the catalog when placing orders in Task 07.

Products are **not scoped to a dealer or station** — this is a single, network-wide catalog Admin maintains, same shape as `Dealer` (Admin-only, no `dealerScope` needed).

## Out of Scope

- **Equipment.product → Product FK migration.** Task 05 added `Equipment.product` as a plain string with a Q&A note that it would "become a real FK once Product Catalog exists." Confirmed with the user: this migration is explicitly deferred, not part of this task. `Equipment.product` remains a plain string for now.
- Dealer-facing product browsing (Task 07).
- Order placement (Task 07).

## Data Model

New `Product` model (from `spec.md`'s ERD, `PRODUCT` entity):
```
PRODUCT {
    uuid id
    string name
    enum category "MS|HSD|LUBRICANT"
    string unit
    decimal price
}
```
No relations to existing models yet — `Order` (Task 07) will add its own `productId` FK to `Product` when that task lands.

## Access Control

Admin-only for all five operations (create/list/get/update/delete). No Dealer access at all in this task — matches the `dealers.js` router pattern (`router.use('/products', authenticate, requireRole('ADMIN'))`), not the `dealerScope` pattern used by Equipment/Station (those are Dealer-owned resources; Product is not).

## Applicable Rules

No `standards_location` is configured for this project (`config_hints.json` → `standards_location: ""`), so there are no rule files to reference. Following the established patterns already in this codebase instead: `backend/src/routes/dealers.js` (Admin-only CRUD shape, zod validation, `requireRole('ADMIN')`), `backend/src/routes/equipment.js` (enum-based type field, migration conventions), and the corresponding frontend `DealerListPage.jsx` (list + inline create-form UI pattern).
