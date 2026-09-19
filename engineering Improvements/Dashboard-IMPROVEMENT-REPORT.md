
# Rento Car Dashboard — Engineering Improvement Report

**Scope:** Frontend SPA — React 19 / Vite 7

This report documents a security, performance, stability, and quality hardening pass on the admin dashboard: **what was identified**, **why it mattered**, and **the resolution delivered**.

---

## Summary

| Area | Focus of this pass | Outcome |
|---|---|---|
| Security | Session hygiene, upload validation, safe links, CSP | Reduced client-side session and XSS attack surface |
| Performance | Route splitting, vendor chunking, dependency cleanup | Smaller initial load, faster first paint |
| Stability | Session UX, error boundaries, request race conditions | Eliminated unexpected logouts and blank-screen failures |
| Quality | Dead code removal, automated smoke tests | Cleaner codebase with regression protection on auth |

---

## 1. Security Hardening

### 1.1 Access Token Storage Consolidation

**Problem identified:** The access token was duplicated across two storage locations (a dedicated token store and a user-profile blob), creating inconsistent cleanup on logout and unnecessary exposure surface.

**Resolution:** Consolidated to a single source of truth for the access token, with the profile store holding only non-sensitive user data.

**Result:** Session clearing is now reliable and consistent; no duplicated credential storage.

### 1.2 Robust Token Expiry Handling

**Problem identified:** Session expiry logic incorrectly treated all non-JWT tokens as expired, which broke login flows for a subset of valid authentication tokens issued by the backend.

**Resolution:** Implemented format-aware expiry validation — JWT-shaped tokens are checked against their expiry claim, while other valid token formats are correctly recognized and preserved.

**Result:** Login and session persistence now work reliably across all supported token formats.

### 1.3 Client-Side Upload Validation

**Problem identified:** File upload size and type limits shown in the UI were not actually enforced, allowing invalid files to be selected.

**Resolution:** Enforced MIME-type allowlisting and size limits at the component level for both general file uploads and vehicle image uploads, with proper cleanup of temporary object URLs.

**Result:** Cleaner upload UX with fewer invalid submissions reaching the backend.

### 1.4 Safe External Link Rendering

**Problem identified:** Externally-supplied URLs (e.g. contact information) were rendered as clickable links without scheme validation.

**Resolution:** Added a URL-sanitization utility ensuring only `http:`/`https:` links are ever rendered as clickable.

**Result:** Unsafe link schemes are neutralized before reaching the UI.

### 1.5 Content Security Policy

**Resolution:** Added a baseline Content-Security-Policy header at the reverse-proxy layer, restricting script and connection sources to trusted origins.

**Result:** Browser-enforced defense-in-depth against injected script execution, on top of application-level protections.

---

## 2. Performance Optimization

### 2.1 Route-Level Code Splitting

**Resolution:** All feature pages now load via lazy-loaded chunks instead of being bundled into the initial JavaScript payload.

**Result:** Each route (offices, bookings, map-heavy pages, etc.) loads on demand, reducing the initial bundle size significantly.

### 2.2 Vendor Chunk Strategy

**Resolution:** Heavy third-party libraries (charting, maps, animation, real-time) were split into dedicated cacheable chunks rather than sharing the main dependency graph.

**Result:** Better browser caching and faster subsequent loads; heavy libraries only download when their feature is actually used.

### 2.3 Dependency Cleanup

**Resolution:** Removed installed-but-unused dependencies from the project.

**Result:** Leaner build output and reduced supply-chain surface.

### 2.4 Bounded Data Sets

**Resolution:** Applied sensible upper bounds to in-memory notification history and bulk-export page sizes.

**Result:** Predictable memory usage and API load under normal and edge-case usage.

### 2.5 Image and Object-URL Lifecycle Management

**Resolution:** Object URLs used for image previews are now properly revoked, and lazy loading was applied to office logos.

**Result:** Reduced memory churn during extended admin sessions.

---

## 3. Stability Improvements

### 3.1 Reliable Session Persistence

**Problem identified:** A browser-event heuristic intended to clear sessions on tab close was unreliable and occasionally logged users out on a normal page refresh.

**Resolution:** Removed the unreliable heuristic in favor of explicit, event-driven session termination (logout action or authentication failure).

**Result:** Normal page reloads reliably preserve the active session.

### 3.2 Application-Level Error Boundary

**Resolution:** Added a root-level error boundary around the application shell.

**Result:** A single component failure now shows a recoverable error screen instead of a blank page — significantly improving resilience for end users.

### 3.3 Request Race-Condition Prevention

**Problem identified:** Rapid filtering or pagination on list views could cause an older, slower API response to overwrite a newer one.

**Resolution:** Introduced request cancellation for affected list views, so only the most recent request's response is applied.

**Result:** List views always reflect the user's latest action, even under rapid interaction.

### 3.4 Consistent Loading & Submission States

**Resolution:** Fixed loading-state propagation in the shared API hook and corrected submission-state handling on the messaging feature.

**Result:** Consistent, predictable loading/disabled states across the dashboard.

### 3.5 Reliable Post-Login Navigation

**Resolution:** Hardened the post-login redirect and token-extraction logic to correctly handle the actual shape of the backend's authentication response.

**Result:** Users reliably land on the dashboard home screen immediately after a successful login.

---

## 4. Code Quality

### 4.1 Dead Code Removal

**Resolution:** Removed leftover scaffold files and unused API endpoint stubs from the codebase.

**Result:** A clearer, unambiguous application entry point and API surface for future contributors.

### 4.2 Automated Regression Testing

**Resolution:** Introduced an automated test suite (Vitest) covering authentication helpers and URL-safety utilities — areas with previously zero test coverage.

**Result:** Regressions in session handling and link safety are now caught automatically before release.

### 4.3 Living Documentation

**Resolution:** This report itself, maintained alongside the codebase, capturing engineering intent and decisions for future maintainers.

**Result:** A shared, durable record of what was improved and why.

---

## How to Verify

```bash
npm install
npm test          # authentication + URL-safety regression tests
npm run build     # confirms lazy-loaded route and vendor chunks
```

**Manual verification performed:**
- Login with a valid account lands on the dashboard home and survives a browser refresh
- External contact links only render for safe URL schemes
- Oversized or invalid file uploads are rejected client-side with clear feedback

---

## Key Areas Touched

| Area | Scope |
|---|---|
| Authentication & session | Token storage, expiry validation, login flow |
| Application shell | Error handling, routing, root layout |
| Data fetching | Shared API hook, request cancellation |
| Uploads & links | File validation, URL sanitization |
| Build & delivery | Bundling strategy, reverse-proxy security headers |
| Testing | Authentication and URL-safety regression suite |

---

*This report reflects a completed engineering hardening pass on the Rento Car admin dashboard, covering security, performance, stability, and code quality.*
