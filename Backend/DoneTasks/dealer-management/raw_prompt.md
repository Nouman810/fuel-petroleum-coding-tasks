# Task 03: Dealer Management

**Depends on:** Task 02 (auth & role-based routing)

The Admin needs a way to actually build out the dealer network — right now the only dealer in the system is the one seeded for login testing.

Build Admin-only management of dealers: create a new dealer (name, region, contact info), edit their details, and change their status between active and suspended. A suspended dealer's login should be rejected even with correct credentials — investigate how the login flow from Task 02 is structured so you plug this check in at the right point rather than bolting it on separately.

Give the Admin a list view of all dealers (with region and status visible at a glance) and a detail view for one dealer. Creating a dealer should also create the login account tied to it (a dealer's `User` record), since a dealer without a way to log in isn't useful yet — that user won't have any stations until Task 04 exists.

This is Admin-only for now; there's no Dealer-facing UI in this task since a dealer can't yet do anything with their own profile beyond logging in.

See full architecture plan: E:/Fuel_Petroleum/Planning_Tasks/Fuel_Petroleum-Plan-DealerFuelStationMgmt/spec.md
