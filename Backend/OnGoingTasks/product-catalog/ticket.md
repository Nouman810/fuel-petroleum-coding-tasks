# [Task 06] Product Catalog

**Story:** Fuel_Petroleum#3 (Dealer & Fuel Station Management System)

## Problem

Dealers will eventually place orders for fuel and lubricants (Task 07), but there is currently no catalog of what's orderable. Admin needs a way to define the product list — what it is, what category it falls into, how it's measured, and what it costs — before ordering can exist.

## Solution

Added an Admin-only CRUD surface for products: create, list, view, update, and delete. Category is a fixed set (MS / HSD / Lubricant) rather than free text, so downstream ordering and reporting can reliably branch on it. The catalog is network-wide, not tied to any individual dealer or station.

## Benefits

- Unblocks Task 07 (Order Management) — orders will reference this catalog.
- Unblocks Task 10 (Reporting) — product-wise and lubricant-wise breakdowns need a fixed category field to group on.
- Prevents category drift (e.g. "diesel" vs "HSD" vs "Diesel") that a free-text field would allow.

## Acceptance Criteria

- [x] Admin can create a product (name, category, unit, price)
- [x] Admin can list all products
- [x] Admin can view a single product
- [x] Admin can update a product
- [x] Admin can delete a product
- [x] Dealers cannot access any product endpoint (403)
- [x] Unauthenticated requests are rejected (401)
- [x] Admin dashboard has a "Manage Products" entry
- [x] Lint passes on both services
- [x] CI is green on the PR
