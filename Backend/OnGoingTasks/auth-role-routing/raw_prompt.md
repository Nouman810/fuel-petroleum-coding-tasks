# Task 02: Auth & Role-Based Routing

**Story:** Fuel_Petroleum#3 (Dealer & Fuel Station Management System)
**Story branch:** story/3-dealer-fuel-station-management-system
**Depends on:** Task 01 (project scaffolding, PR #4)

Admins and Dealers share one login page, but they need to land in completely different parts of the app afterward, and every API call needs to know who's making it and what they're allowed to touch.

Build the login flow: a user submits email/password, the backend verifies credentials and issues a token that carries the user's role. Investigate what token strategy fits the scaffolded backend best (access + refresh token pair is the intended approach — short-lived access token, longer-lived refresh token) and how to store/rotate the refresh token safely.

On the frontend, after login, read the role from the response and route to an Admin shell or a Dealer shell — two separate top-level areas, even though today they're mostly empty.

On the backend, build the pieces every later module will rely on:
- A role-guard so a route can declare "Admin only" or "Dealer only"
- A dealer-scoping check so that when a Dealer hits any endpoint tied to a station/order/complaint/sales record, the backend verifies that resource actually belongs to them, not just that they're logged in as a Dealer
- Password hashing, and basic rate-limiting on the login endpoint

There's no dealer or station data yet, so seed one Admin user and one Dealer user (with no stations assigned yet) directly via a script or migration seed, just enough to prove both login paths work.

See full architecture plan: E:/Fuel_Petroleum/Planning_Tasks/Fuel_Petroleum-Plan-DealerFuelStationMgmt/spec.md
