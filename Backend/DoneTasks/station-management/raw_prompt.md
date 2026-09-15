# Task 04: Station Management

**Depends on:** Task 03 (dealer management)

Every dealer runs one or more physical fuel stations, and right now there's no concept of a station in the system at all. This is the piece that almost everything downstream (equipment, orders, complaints, sales) hangs off of.

Build station management with two sides:
- **Admin:** create a station (name, address, region) and assign it to a dealer. View all stations across the network, filterable by dealer or region.
- **Dealer:** view and edit the profile of their own station(s) — address and contact details, not reassignment. A dealer should never be able to see or touch a station that isn't theirs; make sure this goes through the dealer-scoping check built in Task 02, not a separate ad-hoc check.

A dealer can end up with zero, one, or multiple stations — don't assume exactly one. Design the Dealer dashboard's station view to handle a list, even if most dealers only have one today.

See full architecture plan: E:/Fuel_Petroleum/Planning_Tasks/Fuel_Petroleum-Plan-DealerFuelStationMgmt/spec.md
