# Bug Analysis - Revenue Dashboard Issues

## Bug #1: Cache Privacy Issue (Client B)

**The Problem:**
Client B reported seeing revenue numbers from another company when refreshing the page. This is a serious privacy breach.

**What I Found:**
The cache key in `backend/app/services/cache.py` was only using the property ID, not the tenant ID. The problem is that both Client A and Client B can have properties with the same ID. Looking at the seed data, both tenants have a property called `prop-001` - Client A has "Beach House Alpha" and Client B has "Mountain Lodge Beta".

When Client A queries their `prop-001`, the system creates a cache key `revenue:prop-001` and stores Client A's data. Then when Client B queries their `prop-001`, it uses the exact same cache key `revenue:prop-001`, so Client B gets Client A's cached data instead of their own.

**The Fix:**
Changed the cache key to include the tenant ID: `revenue:{tenant_id}:{property_id}`. Now each tenant's data is cached separately and there's no cross-contamination.

**File:** `backend/app/services/cache.py` line 13

---

## Bug #2: Precision Loss (Finance Team)

**The Problem:**
The finance team noticed revenue totals were "slightly off" by a few cents here and there. They couldn't figure out when or why it happened.

**What I Found:**
There were two places where precision was being lost:

1. In `backend/app/api/v1/dashboard.py`, the code was converting the revenue total from a string to a float. The database stores these values as `NUMERIC(10, 3)` which is exact, but floats in Python use IEEE 754 representation which can't represent all decimal numbers exactly. So `333.333` might become `333.33300000000002` or something close.

2. In `frontend/src/components/RevenueSummary.tsx`, the code was using `Math.round()` on these already-imprecise floats, which compounds the error.

For example, if you have three reservations that should add up to exactly `1000.000` (like `333.333 + 333.333 + 333.334`), the float conversion and rounding can make it show as `999.99` or `1000.01` instead.

**The Fix:**
Keep the revenue as a string throughout the backend (it's already returned as a string from the database query). Don't convert it to float in the API response. In the frontend, handle it as a string and only convert to number for display purposes when needed.

**Files:** 
- `backend/app/api/v1/dashboard.py` line 18
- `frontend/src/components/RevenueSummary.tsx` line 64

---

## Bug #3: Timezone Handling (Client A)

**The Problem:**
Client A said their revenue numbers for March didn't match their internal records. They were worried about accuracy for their board meeting.

**What I Found:**
The `calculate_monthly_revenue` function in `backend/app/services/reservations.py` was using naive datetime objects (without timezone information) when calculating monthly revenue ranges. The problem is that properties have different timezones - Client A's property is in Paris timezone, and reservations are stored with timezone information.

Here's the issue: A reservation might have a check-in date of `2024-02-29 23:30:00 UTC`, which is February 29th in UTC. But in Paris timezone (UTC+1), that same moment is `2024-03-01 00:30:00`, which is March 1st. Client A's internal records correctly count this as March revenue, but the buggy code was counting it as February revenue because it was comparing dates without considering the property's timezone.

**The Fix:**
Made the datetime objects timezone-aware using `zoneinfo.ZoneInfo`. Now when calculating monthly ranges, the dates are properly timezone-aware and will correctly handle properties in different timezones.

**File:** `backend/app/services/reservations.py` lines 4, 11-16

---

## Summary

All three bugs are now fixed:
1. Cache keys now include tenant ID to prevent cross-tenant data leakage
2. Revenue totals maintain precision by keeping values as strings instead of converting to floats
3. Monthly revenue calculations now properly handle timezones for accurate monthly totals

The fixes are minimal and focused - I didn't rebuild anything, just fixed the specific issues that were causing the problems.
