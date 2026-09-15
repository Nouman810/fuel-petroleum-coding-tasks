# Task 05: Equipment Management

**Story:** Fuel_Petroleum#3 (Dealer & Fuel Station Management System)

## Problem

A fuel station is more than an address — it requires physical equipment infrastructure (tanks that store fuel, dispensers that pump it, nozzles that meter the sale) before business operations can proceed. Dealers need the ability to catalog their equipment; admins need visibility across the network for support and troubleshooting. Without equipment records, downstream features like shift-closing sales recording (Task 09) have no place to attach transaction data.

## Solution

Introduced an Equipment entity supporting three types: **Tanks** (hold fuel by product), **Dispensers** (stations contain them), and **Nozzles** (attach to dispensers, each tracks a meter baseline). Dealers can manage equipment scoped to their own stations:

- **Create equipment:** Dealers add tanks with capacity and product type, dispensers, and nozzles (specifying which dispenser each attaches to and its baseline meter reading).
- **Edit equipment:** Dealers update identifier and capacity/product details; type and parent relationships are immutable.
- **Delete equipment:** Dealers remove equipment, with safeguards preventing deletion of a dispenser while child nozzles exist.

Admins see all equipment across the entire network read-only, filterable by station, for diagnostics and auditing.

## Benefits

- **Operational foundation:** Equipment records enable every station to define its fuel-delivery infrastructure precisely.
- **Dealer autonomy:** Dealers manage their own equipment; changes are scoped to their stations automatically.
- **Admin visibility:** Support and operations teams gain cross-network equipment oversight without risk of accidental modification.
- **Data integrity:** Constraints prevent orphaned nozzles and cascading deletions; a nozzle's parent dispenser cannot be removed while in use.

## Acceptance Criteria

- [x] Dealer can create tanks, dispensers, and nozzles scoped to their own stations
- [x] Dealer can edit equipment details (identifier and type-specific fields only)
- [x] Dealer can delete equipment with conflict detection (409 if dispenser has child nozzles)
- [x] Dealer attempts to access another dealer's equipment receive 403 (Forbidden)
- [x] Admin can view all equipment across stations and filter by station
- [x] Admin cannot create, edit, or delete equipment (read-only)
- [x] Unauthenticated requests receive 401 (Unauthorized)
- [x] Malformed requests receive 400 (Bad Request) with validation details
- [x] Frontend provides equipment management forms for dealers on each station's page
- [x] Frontend provides read-only equipment view for admins across all stations
- [x] All routes validate authentication and authorization
- [x] Tests cover 30+ scenarios including edge cases (validation, permissions, conflicts, scoping)
