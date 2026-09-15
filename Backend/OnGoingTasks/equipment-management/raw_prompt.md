# Task 05: Equipment Management

**Depends on:** Task 04 (station management)

A fuel station isn't just an address — it has physical equipment that later tasks depend on: tanks that hold fuel, dispensers that pump it, and nozzles on each dispenser that actually dispense to a customer. Shift-closing sales (Task 09) get recorded per nozzle, so this equipment hierarchy needs to exist and be correct before that task can work.

Build Dealer-facing CRUD for their own station's equipment:
- Tanks: identifier, capacity, which product they hold
- Dispensers: identifier, which station they belong to
- Nozzles: identifier, which dispenser they're attached to, and a running current meter reading field (starts at whatever the dealer records as the baseline when the nozzle is added)

Investigate how the station-scoping from Task 04 works so equipment inherits the same "a dealer only sees their own station's stuff" guarantee — don't re-derive it. Admin should be able to view equipment read-only across all stations (useful for support/troubleshooting) but not edit it.

See full architecture plan: E:/Fuel_Petroleum/Planning_Tasks/Fuel_Petroleum-Plan-DealerFuelStationMgmt/spec.md
