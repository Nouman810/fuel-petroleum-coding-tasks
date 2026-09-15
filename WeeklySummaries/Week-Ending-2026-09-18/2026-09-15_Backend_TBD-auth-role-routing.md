# Auth & Role-Based Routing

**Date:** 2026-09-15 | **Developer:** noumanmuzaffar007@gmail.com | **Platform:** Backend

**What:** Admins and Dealers shared one login page but needed to land in different parts of the app, and every later module needs to know who's calling and what they can touch.

**Fix:** Built the shared login flow (JWT access token + rotated httpOnly refresh cookie), `requireRole`/`dealerScope` middleware for later modules to build on, rate-limited login, and the frontend routing split into Admin/Dealer shells with session restore on reload. PR #5, merged into story/3-dealer-fuel-station-management-system 2026-09-15.
