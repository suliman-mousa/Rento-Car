# Rento Car Web — Engineering Improvement Report

**Scope:** Customer Web (Next.js BFF frontend)

This report documents a security, performance, stability, and quality hardening pass on the customer-facing web platform: **what was identified**, **why it mattered**, and **the resolution delivered**.
---

## Executive Summary

| Dimension | Focus of this pass | Outcome |
|---|---|---|
| Security | Proxy access control, session handoff integrity, request abuse protection, HTTP security headers | Meaningfully reduced attack surface on the BFF layer |
| Performance | Server-side prefetching, response caching, race-condition elimination | Faster first paint, more resilient UI under rapid interaction |
| Stability | Error boundaries, cancellation-safe data loading, consistent session state | Fewer stuck loading states and rendering failures |
| Quality | Automated test suite, CI enforcement, SEO/structured data consistency | Regression protection and stronger public discoverability |

An internal before/after scoring exercise (code-review based, not live telemetry) showed measurable improvement across all four dimensions, with the largest gains in security and quality.

---

## 1. Security Hardening

### 1.1 BFF Proxy Access Control

**Problem identified:** The proxy layer connecting the browser to the upstream API needed stricter boundaries on which paths and methods could be reached through it.

**Resolution:** Implemented an explicit allowlist of permitted upstream paths and HTTP methods; any request outside this list is rejected before the server-only API key is ever attached.

**Result:** The proxy now only ever forwards requests to explicitly sanctioned upstream endpoints — closing off the largest potential attack surface on the frontend.

### 1.2 Session Handoff Integrity

**Problem identified:** Certain authentication handoff points (password reset, account sync) needed tighter guarantees that session cookies are only ever set from verified backend responses — never from client-supplied data.

**Resolution:** Re-architected these flows so that HttpOnly session cookies are populated exclusively from a fresh, server-verified upstream response at each step, with any client-submitted profile or role data explicitly disregarded during cookie writes.

**Result:** Session state is now guaranteed to originate only from the authenticated backend, eliminating an entire class of session-integrity risk.

### 1.3 Request Abuse Protection

**Resolution:** Added same-origin validation for state-changing requests and in-memory rate limiting on authentication and OTP-related endpoints.

**Result:** Meaningfully raises the cost of cross-origin abuse and naive flooding attempts against sensitive auth flows.

### 1.4 HTTP Security Headers

**Resolution:** Added a baseline set of browser-enforced security headers (frame protection, MIME-sniffing prevention, referrer policy, permissions policy, HSTS).

**Result:** Stronger browser-side defense-in-depth layered on top of application-level controls.

---

## 2. Performance Optimization

### 2.1 Server-Side Prefetching for Public Data

**Problem identified:** Public, high-traffic pages (car listings, airport transport, contact info) were loading their initial data entirely client-side, causing avoidable request waterfalls and slower first content.

**Resolution:** Introduced server-side prefetching with response caching for public GET data, seeding the page with data at render time and skipping redundant client-side fetches on initial load.

**Result:** Meaningfully faster first paint on the highest-traffic pages, with data available immediately at server-render time when upstream is reachable.

### 2.2 Race-Condition-Free Data Fetching

**Problem identified:** Fast, repeated user interaction (filtering, pagination, navigating between listings) could cause an older, slower response to overwrite a newer one in the UI.

**Resolution:** Introduced request cancellation across list and detail data-fetching hooks, ensuring only the most recent request's response is ever applied to the UI.

**Result:** Eliminated an entire category of "stale data flashes" under rapid user interaction.

### 2.3 Indexable, Shareable Detail Pages

**Resolution:** Car detail pages now render their core content server-side rather than relying solely on a client-side modal.

**Result:** Detail pages are directly shareable and crawlable, improving both user experience and SEO.

---

## 3. Stability Improvements

### 3.1 Application-Level Error Boundaries

**Resolution:** Added route-level and global error boundaries at the application-router level.

**Result:** Render failures now show a recoverable error screen instead of falling through to a generic, unbranded failure state.

### 3.2 Cancellation-Safe Loading States

**Problem identified:** Concurrent API calls sharing a single loading indicator could produce inconsistent loading UI when one request was cancelled while another was still in flight.

**Resolution:** Loading state is now correctly scoped so cancelled requests never incorrectly clear the loading indicator for still-active ones.

**Result:** Stable, predictable loading UX even under rapid navigation and cancelled requests.

### 3.3 Consistent Session State

**Resolution:** Session-related cookies are now updated exclusively through server-verified flows (see §1.2), removing prior scenarios where client and server state could drift out of sync.

**Result:** Reduced risk of a broken or inconsistent authenticated session state.

---

## 4. Code Quality

### 4.1 Automated Test Coverage

**Resolution:** Introduced an automated test suite covering the highest-risk request-guarding and authentication-parsing logic — areas with previously zero automated coverage.

**Result:** Regressions in critical security and session-handling logic are now caught automatically before merge.

### 4.2 Continuous Integration Enforcement

**Resolution:** CI now runs type-checking, the automated test suite, and a full production build on every change.

**Result:** Type errors, test regressions, and build breakages are caught before reaching production.

### 4.3 SEO & Structured Data Consistency

**Resolution:** Unified page metadata generation across the application, added structured data (JSON-LD) to key public pages, and ensured private/authenticated routes are correctly excluded from search indexing.

**Result:** Stronger, more consistent public discoverability with no risk of private pages being inadvertently indexed.

### 4.4 Documentation

**Resolution:** Maintained living architecture and feature-development documentation alongside the codebase, including a checklist for safely adding new upstream-connected features.

**Result:** Consistent implementation patterns for future feature work, reducing the chance of reintroducing previously-solved problems.

---

## 5. Engineering Principles Reinforced

This hardening pass reinforced several architectural guarantees that were already foundational to the platform:

- The browser never holds a bearer token for API calls — session state lives exclusively in HttpOnly cookies, with the BFF layer mediating all upstream communication.
- Server-only secrets never reach the client bundle.
- All feature modules route API access through a single, centrally-guarded proxy layer rather than calling the upstream API directly.

---

## How to Verify

```bash
npm run typecheck
npm test
npm run build
```

**Manual verification performed:** exercised core listing, detail, and authentication flows (login, signup, forgot-password) end-to-end, and confirmed that requests to non-sanctioned proxy paths are correctly rejected.

---

## Document Ownership

| Related documentation | Role |
|---|---|
| `web-architecture.md` | Security model & BFF data flow |
| **This file** | Improvement history and outcomes |

---

*This report reflects a completed engineering hardening pass on the Rento Car customer web platform, covering security, performance, stability, and code quality.*
