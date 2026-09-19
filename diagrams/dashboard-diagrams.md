# Admin Dashboard — Architecture Diagrams

Enterprise-grade architecture diagrams for the Rento Car Admin Dashboard codebase.
Render with any Mermaid-compatible viewer (GitHub, GitLab, VS Code, Obsidian).

**Related:** [dashboard-architecture.md](../dashboard-architecture.md)

---

## Diagram Index

| # | Diagram | Standard |
|---|---|---|
| 1 | [System Context](#1-system-context-c4-level-1) | C4 Level 1 |
| 2 | [Layered Architecture](#2-layered-application-architecture) | Layered Design |
| 3 | [Session State Machine](#3-session-state-machine) | UML State |
| 4 | [401 Session Teardown](#4-401-session-teardown-sequence) | UML Sequence |
| 5 | [Defense in Depth](#5-defense-in-depth-model) | Security Model |
| 6 | [Data Flow](#6-data-flow-rest-layer) | REST Pipeline |
| 7 | [Real-Time Flow](#7-real-time-notification-flow) | Event-Driven |
| 8 | [Deployment Pipeline](#8-deployment-pipeline) | Docker Multi-Stage |

---

## 1. System Context (C4 Level 1)

How the admin SPA interacts with backend services and production infrastructure.

```mermaid
flowchart TB
    subgraph Users["Actors"]
        Admin["Admin User"]
    end

    subgraph App["Rento Car Admin SPA"]
        Dashboard["React Dashboard<br/>/dashboard/"]
    end

    subgraph Backend["Backend Services"]
        API["Laravel REST API"]
        Broadcast["Broadcasting Auth"]
        Pusher["Pusher WebSocket"]
    end

    subgraph Infra["Production Infrastructure"]
        Nginx["nginx reverse proxy"]
        Docker["Docker multi-stage image"]
    end

    Admin --> Dashboard
    Dashboard --> API
    Dashboard --> Broadcast
    Dashboard --> Pusher
    Docker --> Nginx
    Nginx --> Dashboard
```

**Key files:** `Dockerfile`, `nginx.conf`, `src/services/axiosInstance.js`, `src/services/echo.js`

---

## 2. Layered Application Architecture

Separation of presentation, application, domain, and infrastructure concerns.

```mermaid
flowchart TB
    subgraph Presentation["Presentation Layer"]
        Pages["Feature Pages"]
        Layout["Sidebar Layout"]
        Components["Shared Components"]
    end

    subgraph Application["Application Layer"]
        Hooks["useApi · useNotificationsListener"]
        Router["React Router + Guards"]
        i18n["i18next RTL/LTR"]
    end

    subgraph Domain["Domain / State Layer"]
        Auth["auth.js"]
        Cleanup["sessionCleanup.js"]
        Stores["Zustand Stores"]
    end

    subgraph Infrastructure["Infrastructure Layer"]
        Axios["axiosInstance"]
        Endpoints["apiEndpoints.js"]
        Echo["Laravel Echo singleton"]
    end

    subgraph External["External Systems"]
        REST["REST API"]
        WS["Private Channels"]
    end

    Pages --> Hooks
    Pages --> Router
    Layout --> Pages
    Hooks --> Axios
    Hooks --> Endpoints
    Router --> Auth
    Auth --> Cleanup
    Pages --> Stores
    Axios --> REST
    Echo --> WS
    Auth --> Echo
```

**Key files:** `src/features/`, `src/hooks/`, `src/utils/`, `src/services/`

---

## 3. Session State Machine

Session lifecycle from anonymous access through coordinated teardown.

```mermaid
stateDiagram-v2
    [*] --> Anonymous

    Anonymous --> Authenticated: login success
    Authenticated --> Authenticated: authorized API calls

    Authenticated --> Expired: JWT exp reached
    Authenticated --> Revoked: API returns 401/403
    Authenticated --> Anonymous: tab/window close

    Expired --> Teardown: clearSession()
    Revoked --> Teardown: handleUnauthorizedSession()

    state Teardown {
        [*] --> ClearStorage
        ClearStorage --> ClearStore
        ClearStore --> KillSockets
        KillSockets --> HardRedirect
        HardRedirect --> [*]
    }

    Teardown --> Anonymous: redirectToLogin()
```

**Key files:** `src/utils/auth.js`, `src/utils/sessionCleanup.js`, `src/components/ProtectedRoute.jsx`

---

## 4. 401 Session Teardown Sequence

UML sequence when authorization fails. An orchestrator coordinates auth, WebSocket, and navigation cleanup as a single atomic operation — rather than leaving each concern to clean up independently and risk an inconsistent half-logged-out state.

```mermaid
sequenceDiagram
    autonumber
    participant UI as React UI
    participant Guard as Route Guard
    participant Auth as auth.js
    participant Axios as axiosInstance
    participant API as Backend API
    participant Cleanup as sessionCleanup
    participant Echo as echo.js

    UI->>Guard: navigate protected route
    Guard->>Auth: isAuthenticated()
    Auth->>Auth: decode JWT + check exp
    Auth-->>Guard: true
    Guard-->>UI: render page

    UI->>Axios: API request
    Axios->>Auth: attach Bearer token
    Axios->>API: HTTP request
    API-->>Axios: 401 Unauthorized

    Axios->>Cleanup: handleUnauthorizedSession()
    Cleanup->>Auth: clearSession()
    Cleanup->>Echo: disconnectEcho()
    Note over Echo: leaveAllChannels() then disconnect()
    Cleanup->>Auth: redirectToLogin()
```

**Key files:** `src/services/axiosInstance.js`, `src/utils/sessionCleanup.js`, `src/services/echo.js`

---

## 5. Defense in Depth Model

Five security layers, from proactive client-side checks through to edge hardening. No single layer is trusted as the sole line of defense.

```mermaid
flowchart LR
    subgraph L1["Layer 1 — Proactive"]
        A1["Route Guards"]
        A2["JWT exp validation"]
    end

    subgraph L2["Layer 2 — Reactive"]
        B1["401/403 interceptor"]
        B2["Auth endpoint exclusion"]
    end

    subgraph L3["Layer 3 — Orchestration"]
        C1["handleUnauthorizedSession()"]
    end

    subgraph L4["Layer 4 — State Purge"]
        D1["localStorage"]
        D2["Zustand store"]
        D3["WebSocket teardown"]
        D4["Hard redirect"]
    end

    subgraph L5["Layer 5 — Edge"]
        E1["nginx security headers"]
        E2["TLS + forceTLS"]
    end

    L1 --> L2 --> L3 --> L4
    L5 -.-> L1
```

**Key files:** `nginx.conf`, `src/utils/auth.js`, `src/services/axiosInstance.js`, `src/services/echo.js`

---

## 6. Data Flow (REST Layer)

Standard request path from feature pages to the backend API.

```mermaid
flowchart LR
    Page["Feature Page"] --> Hook["useApi()"]
    Hook --> Axios["axiosInstance"]
    Axios --> EP["apiEndpoints.js"]
    EP --> API["Backend API"]

    API --> Axios
    Axios --> Hook
    Hook --> Page

    Page --> Store["Zustand / local state"]
    Page --> UI["Render UI"]
```

**Key files:** `src/hooks/useApi.js`, `src/services/apiEndpoints.js`, `src/services/axiosInstance.js`

---

## 7. Real-Time Notification Flow

Private channel subscription with Bearer auth and coordinated teardown on logout or 401.

```mermaid
flowchart TB
    Dashboard["Dashboard / Pages"] --> Listener["useNotificationsListener"]
    Listener --> Factory["createEcho()"]
    Factory --> Auth["getAccessToken()"]
    Auth --> Private["private channel user.id"]
    Private --> Event[".request.made event"]
    Event --> Store["useNotificationStore"]
    Store --> Bell["NotificationBell / Panel"]

    Logout["401 / logout"] --> Teardown["disconnectEcho()"]
    Teardown --> Private
```

**Key files:** `src/hooks/useNotificationsListener.js`, `src/services/echo.js`, `src/store/useNotificationStore.js`

---

## 8. Deployment Pipeline

Production build and serve flow via a Docker multi-stage image.

```mermaid
flowchart LR
    Source["Source Code"] --> Build["Node 20 build stage<br/>npm ci + vite build"]
    Build --> Dist["dist/ assets"]
    Dist --> Nginx["nginx:alpine"]
    Nginx --> Serve["/dashboard/ SPA"]
    Env["VITE_* build args"] --> Build
```

**Key files:** `Dockerfile`, `docker-compose.yml`, `nginx.conf`, `.env.example`

---

## Conventions

| Standard | Usage in this project |
|---|---|
| **C4 Model** | System context and container boundaries |
| **UML Sequence** | Auth failure and session teardown |
| **UML State** | Session lifecycle states |
| **Layered Architecture** | `features/` · `hooks/` · `utils/` · `services/` |
| **Defense in Depth** | Guards → interceptors → orchestrator → edge |

---

## Viewing Diagrams Locally

**VS Code:** install the [Markdown Preview Mermaid Support](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid) extension, then open this file and preview.

**GitHub / GitLab:** Mermaid blocks render natively in the repository UI — no setup required.
