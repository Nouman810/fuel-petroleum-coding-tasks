# Task 06: Product Catalog

**Depends on:** Task 02 (auth & role-based routing)

Dealers order fuel and lubricants, and Admin needs to define what's orderable before that can happen. This task is independent of dealers/stations — it can be built right after auth, in parallel with Task 03.

Build Admin-only CRUD for products: name, category (MS petrol, HSD diesel, or lubricant), unit of measure, and price. Give Admin a list view of the catalog. There's no Dealer-facing UI in this task — dealers will browse this catalog when placing orders in Task 07, not before.

Keep category as a fixed set (MS / HSD / lubricant) rather than a free-text field, since order and reporting logic downstream will branch on it.

See full architecture plan: E:/Fuel_Petroleum/Planning_Tasks/Fuel_Petroleum-Plan-DealerFuelStationMgmt/spec.md
