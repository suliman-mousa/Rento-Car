# Customer Web — Architecture Diagrams

Visual overview of the most important parts of the Customer Web application.
Diagrams use [Mermaid](https://mermaid.js.org/) (renders natively on GitHub and most Markdown previews).

**Related:** [web-architecture.md](../web-architecture.md)

---

## Table of Contents

1. [High-Level System](#1-high-level-system)
2. [Project Layout (Feature Map)](#2-project-layout-feature-map)
3. [BFF Proxy & Allowlist](#3-bff-proxy--allowlist)
4. [Session Cookies & Client State](#4-session-cookies--client-state)
5. [Login Flow](#5-login-flow)
6. [Register Flow](#6-register-flow)
7. [Forgot-Password Flow](#7-forgot-password-flow)
8. [Public Data: Cars Page (RSC + Client)](#8-public-data-cars-page-rsc--client)
9. [Car Details & Booking](#9-car-details--booking)
10. [Security Layers](#10-security-layers)

---

## 1. High-Level System

The browser never talks to the upstream API with secrets. Next.js acts as a **BFF** (Backend for Frontend).

```mermaid
flowchart LR
  subgraph Browser
    UI[React UI<br/>features/*]
    Z[Zustand<br/>user profile only]
    AX[axiosInstance<br/>baseURL /api/backend]
  end

  subgraph Next["Next.js App Router"]
    Pages[RSC pages<br/>app/*/page.tsx]
    AuthAPI["/api/auth/*"]
    Proxy["/api/backend/[...path]"]
    Public["lib/server/publicLists.ts<br/>direct upstream fetch"]
  end

  subgraph Upstream["Upstream REST API"]
    API["API_BASE_URL<br/>+ x-api-key"]
  end

  UI --> AX
  UI --> AuthAPI
  Pages --> Public
  AX --> Proxy
  AuthAPI --> API
  Proxy --> API
  Public --> API
  AuthAPI -.->|HttpOnly cookies| Browser
  Proxy -.->|reads accessToken cookie| Browser
  Z -.->|UI only, no token| UI
```

---

## 2. Project Layout (Feature Map)

```mermaid
flowchart TB
  subgraph app["app/"]
    PAGES["page.tsx — SEO metadata, RSC shell"]
    AUTH["api/auth/* — cookies + upstream auth"]
    BACKEND["api/backend/[...path] — allowlisted proxy"]
  end

  subgraph features["features/"]
    LOGIN[login]
    REG[register]
    FP[forgot-password]
    CARS[cars]
    BOOK[booking]
    MYB[my-bookings]
    TAXI[airport-taxi]
    ACC[account]
    MORE["contact · notifications · legal · …"]
  end

  subgraph lib["lib/"]
    API["api/ — API_ENDPOINTS"]
    ROUTER["router/ — ROUTES"]
    SERVER["server/ — upstream, allowlist, guards, publicLists"]
    AUTHLIB["auth/ — session cookies, parsers"]
    SEO["seo/ — buildPageMetadata"]
  end

  PAGES --> features
  features --> hooks["hooks/useApi"]
  hooks --> BACKEND
  LOGIN --> AUTH
  REG --> AUTH
  FP --> AUTH
  CARS --> SERVER
  BACKEND --> SERVER
  AUTH --> AUTHLIB
```

---

## 3. BFF Proxy & Allowlist

Every browser CRUD/list call goes through the proxy. Unknown paths never receive the server-only API key.

```mermaid
sequenceDiagram
  participant C as Client (useApi / axios)
  participant P as /api/backend/[...path]
  participant A as allowlist
  participant U as Upstream API

  C->>P: GET/POST /api/backend/cars/filter-web
  P->>A: assertProxyAllowed(method, path)
  alt not allowed
    A-->>P: 404 / 405
    P-->>C: rejected
  else allowed
    P->>P: Origin check (mutations)<br/>rate limit (OTP/login/…)<br/>require API key
    opt authRequired
      P->>P: require accessToken cookie
    end
    P->>U: forward + x-api-key<br/>+ Bearer from cookie
    U-->>P: response
    P-->>C: pass-through body/status
  end
```

**Key files**

- `app/api/backend/[...path]/route.ts`
- `lib/server/backendProxyAllowlist.ts`
- `lib/server/requestGuard.ts`
- `lib/api/index.ts` (`API_ENDPOINTS`)

---

## 4. Session Cookies & Client State

```mermaid
flowchart TB
  subgraph ServerOnly["Server-only (HttpOnly)"]
    AT[accessToken cookie<br/>Bearer for upstream]
    RU[rento_user cookie<br/>profile JSON, no token]
  end

  subgraph ClientSafe["Browser-visible"]
    ZU[useUserStore — UserProfile]
    SH[SessionHydrator<br/>GET /api/auth/session]
  end

  Login["/api/auth/login · signup<br/>password-reset-session"] --> AT
  Login --> RU
  Sync["/api/auth/sync-user<br/>GET upstream /user"] --> RU
  SH --> ZU
  RU -.->|session route reads| SH
  AT -.->|proxy attaches Authorization| Proxy["/api/backend"]
```

**Rule:** the client never stores the access token for API calls. Zustand is UI state only — the HttpOnly cookie is always the source of truth.

---

## 5. Login Flow

```mermaid
sequenceDiagram
  participant U as User
  participant L as LoginClient / useLogin
  participant Clear as POST /api/auth/clear
  participant Login as POST /api/auth/login
  participant API as Upstream /login

  U->>L: phone + password
  L->>Clear: clear stale cookies
  L->>Login: { phone, password }
  Note over Login: guardAuthMutation<br/>origin + rate limit
  Login->>API: x-api-key + body
  API-->>Login: user + token
  Login->>Login: setHttpOnlySessionFromPayload
  Login-->>L: { success, user } without token
  L->>L: setUser(Zustand) → home
```

---

## 6. Register Flow

```mermaid
sequenceDiagram
  participant R as Register wizard
  participant Proxy as /api/backend
  participant Signup as POST /api/auth/signup
  participant API as Upstream

  R->>Proxy: POST send-otp
  Proxy->>API: send-otp
  R->>Proxy: POST verify-otp
  Proxy->>API: verify-otp
  R->>Signup: profile + phone + password
  Note over Signup: parseSignupBody<br/>strips role
  Signup->>API: POST /signup
  API-->>Signup: user + token
  Signup->>Signup: set HttpOnly cookies
  Signup-->>R: { success, user } without token
```

---

## 7. Forgot-Password Flow

OTP verification for the reset **session** happens on the server — no client-submitted token is ever trusted into a cookie.

```mermaid
sequenceDiagram
  participant F as ForgotPassword UI
  participant Proxy as /api/backend
  participant PRS as POST /api/auth/password-reset-session
  participant API as Upstream

  F->>Proxy: POST send-reset-otp
  Proxy->>API: send-reset-otp
  F->>PRS: { phone, otp } only
  Note over PRS: server calls verify-otp<br/>then sets cookies
  PRS->>API: POST verify-otp
  API-->>PRS: token + user
  PRS->>PRS: setHttpOnlySessionFromPayload
  PRS-->>F: success
  F->>Proxy: POST reset-password<br/>Bearer via cookie
  Proxy->>API: reset-password
  F->>F: clear session → /login
```

---

## 8. Public Data: Cars Page (RSC + Client)

```mermaid
flowchart TB
  subgraph RSC["Server — app/cars/page.tsx"]
    PL[publicLists.ts]
    UP[upstreamFetch<br/>revalidate cache]
    INIT[CarsPageInitialData]
    PL --> UP
    UP --> INIT
  end

  subgraph Client["Client — CarsPageClient / useCars"]
    SEED[Seed state from initialData]
    SKIP[Skip mount refetch if seeded]
    SEARCH[User search / filters]
    ABORT[AbortController]
    USEAPI[useApi → /api/backend]
    SEED --> SKIP
    SEARCH --> ABORT --> USEAPI
  end

  INIT --> SEED
  USEAPI --> Proxy["Allowlisted proxy"]
  DETAIL["Click car → /cars/id"] --> CarPage["RSC car details page"]
```

This dual-path design (server-prefetched initial state + client-side refetch only on real user interaction) avoids a redundant fetch on first paint while keeping search and filtering fully dynamic. `AbortController` cancels stale in-flight searches so a fast typist never sees an old response overwrite a newer one.

**Related:** airports on `/airport-taxi` use the same prefetch pattern.

---

## 9. Car Details & Booking

```mermaid
sequenceDiagram
  participant List as /cars list
  participant Detail as /cars/[id] RSC + client
  participant Verify as bookings/verify
  participant Checkout as /bookings/checkout
  participant Create as POST /bookings multipart
  participant API as Upstream

  List->>Detail: navigate ROUTES.CAR_DETAILS(id)
  Detail->>API: GET /cars/id (server prefetch)
  Detail->>Detail: Product JSON-LD + UI
  Note over Detail: user picks dates / delivery
  Detail->>Verify: useApi POST bookings/verify
  Verify->>API: verify
  API-->>Verify: ok
  Detail->>Checkout: save session → navigate
  Checkout->>Create: create booking FormData
  Create->>API: POST /bookings
  Checkout->>Checkout: success → my bookings
```

Booking availability is verified in a separate step **before** checkout, rather than only at final submission — surfacing conflicts (e.g. a vehicle just booked by someone else) earlier in the flow instead of after the customer has filled out the full checkout form.

---

## 10. Security Layers

```mermaid
flowchart TB
  subgraph Edge["Request guards"]
    O[Same-origin Origin/Referer]
    R[In-memory rate limit]
    H[Security headers<br/>XFO, nosniff, HSTS, …]
  end

  subgraph AuthSurface["Cookie writes"]
    L["/api/auth/login"]
    S["/api/auth/signup"]
    P["/api/auth/password-reset-session"]
    Y["/api/auth/sync-user ← GET /user only"]
  end

  subgraph DataSurface["Data plane"]
    AL[Proxy allowlist]
    KEY[API key — server-only]
    COOKIE[Bearer from HttpOnly cookie]
  end

  Browser --> O
  O --> R
  R --> AuthSurface
  Browser --> AL
  AL --> KEY --> COOKIE
  H -.-> Browser
```

Only four routes are permitted to write session cookies (`login`, `signup`, `password-reset-session`, `sync-user`) — every other authenticated call flows through the generic proxy and only ever *reads* the existing cookie. This narrows the "who can create a session" surface to a small, auditable set of code paths.

---

## Quick Reference — Important Paths

| Concern | Path |
|---|---|
| Routes | `lib/router/index.ts` |
| API segments | `lib/api/index.ts` |
| Proxy | `app/api/backend/[...path]/route.ts` |
| Allowlist | `lib/server/backendProxyAllowlist.ts` |
| Guards | `lib/server/requestGuard.ts` |
| RSC public fetch | `lib/server/publicLists.ts`, `upstreamFetch.ts` |
| Session cookies | `lib/auth/session.server.ts` |
| Client API | `hooks/useApi.ts`, `services/axiosInstance.ts` |
| Cars UI | `features/cars/` |
| Booking | `features/booking/` |
| Auth UI | `features/login`, `register`, `forgot-password` |
