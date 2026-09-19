
# Mobile App — Key Architecture Diagrams

Mermaid diagrams for the most important parts of the Rento Flutter app (Customer + Office).
View in GitHub, VS Code (Markdown Preview Mermaid), or any Mermaid-compatible viewer.

**Related:** [mobile-architecture.md](../mobile-architecture.md)

---

## Index

| # | Diagram | What it shows |
|---|---|---|
| 1 | [System Context](#1-system-context) | App ↔ API ↔ Firebase ↔ Maps |
| 2 | [App Bootstrap](#2-app-bootstrap) | Startup order |
| 3 | [Role-Based Routing](#3-role-based-routing) | Customer / Office / Guest |
| 4 | [Customer Home Tabs](#4-customer-home-tabs) | Main customer shell |
| 5 | [Office Home Tabs](#5-office-home-tabs) | Main office shell + BLoCs |
| 6 | [Office Clean Architecture](#6-office-clean-architecture) | presentation → domain → data |
| 7 | [Auth & Secure Token Flow](#7-auth--secure-token-flow) | Login / storage / logout |
| 8 | [HTTP Request Pipeline](#8-http-request-pipeline) | Dio + errors + redacted logs |
| 9 | [Customer Booking Flow](#9-customer-booking-flow) | Search → checkout → history |
| 10 | [Office Order Handling](#10-office-order-handling) | Accept / reject / delivery |
| 11 | [Push Notifications](#11-push-notifications) | FCM + local + dedupe |
| 12 | [Image Caching](#12-image-caching) | CachedCarImage + decode bounds |
| 13 | [Security Boundaries](#13-security-boundaries) | What is protected, and how |
| 14 | [Test Coverage Map](#14-test-coverage-map) | What automated tests cover |

---

## 1. System Context

```mermaid
flowchart LR
    subgraph Mobile["Rento Flutter App"]
        CUS[Customer UI]
        OFF[Office UI]
        CORE[Core: Dio · Security · Config]
    end

    API[(Backend API + CDN)]
    FCM[Firebase Cloud Messaging]
    MAPS[Google Maps / Places / Directions]

    CUS --> CORE
    OFF --> CORE
    CORE -->|HTTPS REST + x-api-key| API
    CORE -->|FCM token / push| FCM
    CUS --> MAPS
    OFF --> MAPS
```

---

## 2. App Bootstrap

```mermaid
sequenceDiagram
    participant Main
    participant Firebase
    participant DI as GetIt Office DI
    participant STS as SecureTokenStorage
    participant NS as NotificationService
    participant App as GetMaterialApp

    Main->>Main: WidgetsFlutterBinding + imageCache bounds
    Main->>Firebase: initializeApp()
    Main->>Main: register FCM background handler
    Main->>DI: gi.init()
    Main->>STS: init() + migrate legacy token
    Main->>NS: initialize()
    Main->>App: runApp MyApp
    App->>App: locale ar_SA / en_US
    App->>App: AuthCubit + SplashScreen
```

Deliberate sequencing: security (secure storage init + legacy token migration) and notification registration both complete **before** the app renders its first authenticated screen, avoiding race conditions where a screen could momentarily render with stale or missing auth state.

---

## 3. Role-Based Routing

```mermaid
flowchart TD
    A[SplashScreen] --> B{Has session?}
    B -->|No| C[Welcome / Login]
    B -->|Yes| D{Role?}
    D -->|Customer| E[Customer HomeScreen<br/>5 tabs]
    D -->|Office| F[Office HomeScreenOfficer<br/>5 tabs]
    C --> G{Guest or Login?}
    G -->|Guest| E
    G -->|Login| H[OTP / Account]
    H --> E
    H --> F
```

---

## 4. Customer Home Tabs

```mermaid
flowchart LR
    subgraph CustomerHome["Customer HomeScreen"]
        T1[Home]
        T2[Bookings]
        T3[Airport]
        T4[Points]
        T5[Profile]
    end

    T1 --> W1[Search panel + Ads + Cars grid]
    T1 --> W2[home/widgets/*]
    T2 --> W3[BookingHistoryCubit]
    T3 --> W4[Airport transport]
    T5 --> W5[Language · Contact · Logout]
```

---

## 5. Office Home Tabs

```mermaid
flowchart LR
    subgraph OfficeHome["Office HomeScreenOfficer"]
        O1[Orders]
        O2[Cars]
        O3[Offers]
        O4[Finance]
        O5[Profile]
    end

    O1 --> B1[BookingBloc]
    O2 --> B2[OfficeCarsBloc]
    O3 --> B3[OfficeOffersBloc]
    O4 --> B4[SettlementBloc]
    O5 --> B5[OfficeProfileBloc]

    B1 & B2 & B3 & B4 & B5 --> DI[GetIt]
    DI --> DIO[DioHelper]
    DIO --> API[(Backend API)]
```

---

## 6. Office Clean Architecture

```mermaid
flowchart TB
    subgraph Presentation
        UI[Screens / Widgets]
        BL[BLoC]
    end

    subgraph Domain
        UC[Use Cases]
        RI[Repository Interfaces]
        ENT[Entities]
    end

    subgraph Data
        IMPL[Repository Implementations]
        DS[Remote Data Sources]
        MOD[Models]
    end

    UI --> BL
    BL --> UC
    UC --> RI
    IMPL -.implements.-> RI
    IMPL --> DS
    DS --> MOD
    MOD --> ENT
```

Example feature folders: `office_cars`, `office_offers`, `office_orders`, `office_profile`, `finance`.

---

## 7. Auth & Secure Token Flow

```mermaid
sequenceDiagram
    participant UI as Login / AuthCubit
    participant Repo as AuthRepository
    participant STS as AuthTokenStore<br/>SecureTokenStorage
    participant GS as GetStorage<br/>user_data
    participant API as Backend

    UI->>API: POST /login
    API-->>UI: user + token
    UI->>Repo: saveUserToStorage(user)
    Repo->>STS: write(token)
    Repo->>GS: write profile WITHOUT token

    Note over STS,GS: Token only ever lives in the secure store

    UI->>Repo: clearStorage on logout
    Repo->>STS: delete()
    Repo->>GS: remove user + FCM keys
```

```mermaid
stateDiagram-v2
    [*] --> LoggedOut
    LoggedOut --> Guest: loginAsGuest
    LoggedOut --> LoggingIn: credentials
    LoggingIn --> LoggedIn: success
    LoggingIn --> LoggedOut: failure
    Guest --> LoggedOut: clear / logout
    LoggedIn --> LoggedOut: logout
```

**Key design decision:** the profile snapshot cached in `GetStorage` (unencrypted, used for fast UI reads) is structurally prevented from ever containing the token field — token storage and profile storage are two separate write paths, not one object split after the fact.

---

## 8. HTTP Request Pipeline

```mermaid
flowchart LR
    SRC[Cubit / BLoC / Repository] --> DIO[Dio]
    DIO --> E[ApiErrorInterceptor]
    E --> L[ApiLogInterceptor<br/>debug + redacted]
    L --> P[PrettyDioLogger<br/>debug only]
    P --> API[Backend API]

    API --> MAP[DioExceptionMapper]
    MAP -->|429| RL[RateLimitException]
    MAP -->|other| SE[ServerException]
    RL & SE --> SRC
```

Sensitive fields (`token`, `phone`, `email`, OTP, document paths, …) are masked by a dedicated log-redaction layer **before** anything reaches debug logs — so verbose logging during development never becomes an accidental data-leak surface.

---

## 9. Customer Booking Flow

```mermaid
flowchart TD
    A[Governorate + dates] --> B[SearchCubit]
    B --> C[Home / View-all cars]
    C --> D[Car Details]
    D --> E[CheckoutCubit]
    E --> F{Logged in?}
    F -->|Yes| G[POST /bookings + Bearer]
    F -->|No| H[Guest POST /bookings]
    G --> I[Confirmation]
    H --> I
    I --> J{OTP needed?}
    J -->|Yes| K[OtpCubit]
    J -->|No| L[Booking details / history]
    K --> L
```

---

## 10. Office Order Handling

```mermaid
flowchart TD
    A[Orders tab] --> B[Fetch bookings]
    B --> C{Status}
    C -->|Pending| D[Accept / Reject]
    C -->|Accepted| E[Details + delivery map]
    C -->|Other| F[Read-only details]

    D --> G[AcceptOrReject use case]
    G --> H[Refresh list]

    E --> I[Documents · CachedNetworkImage]
    E --> J[DeliveryMapScreen · Directions]
```

---

## 11. Push Notifications

```mermaid
sequenceDiagram
    participant FCM as Firebase Messaging
    participant NS as NotificationService
    participant Dedupe as NotificationDedupeStore
    participant Local as Local Notifications
    participant API as Backend

    NS->>FCM: permission + getToken
    NS->>API: register FCM token
    FCM->>NS: onMessage foreground
    NS->>Dedupe: shouldDisplay?
    Dedupe-->>NS: yes / no
    NS->>Local: show if allowed
    FCM->>NS: onMessageOpenedApp / getInitialMessage
    Note over NS: Debug logs use keys only,<br/>never full PII payloads
```

A dedicated dedupe store prevents the same push event from surfacing twice across foreground/background/terminated app states — a subtlety that's easy to miss until it produces duplicate notifications in production.

---

## 12. Image Caching

```mermaid
flowchart TD
    W[Remote image needed] --> T{Widget}
    T -->|Cars / lists| CCI[CachedCarImage]
    T -->|Ads / docs / logos| CNI[CachedNetworkImage]
    T -->|Fullscreen| FSV[CachedNetworkImageProvider]

    CCI --> DIM[ImageCacheDims<br/>mem + disk decode size]
    CNI --> DIM
    DIM --> CACHE[cached_network_image]
    CACHE --> MEM[Flutter imageCache<br/>bounded size]
    CACHE --> DISK[Disk cache]
    CACHE --> CDN[CDN / API images]
```

Explicit decode-size bounds prevent full-resolution images from being decoded into memory just to render a small thumbnail — a common, easy-to-miss source of memory pressure and jank on lower-end devices.

---

## 13. Security Boundaries

```mermaid
flowchart LR
    subgraph Trusted
        STS[SecureTokenStorage]
        DIO[Dio + Bearer]
        RED[Log Redaction Layer]
    end

    subgraph Guarded
        URL[SafeUrlLauncher]
        WV[SafeWebViewHelper]
    end

    subgraph LocalNonSecret
        GS[GetStorage profile<br/>no token]
        LOC[Locale prefs]
    end

    subgraph External
        WEB[Allowlisted HTTPS / tel / mailto]
        API[REST API]
        MAPS[Google Maps keys]
    end

    DIO --> STS
    DIO --> API
    DIO --> RED
    URL --> WEB
    WV --> WEB
    DIO -.-> MAPS
```

`SafeUrlLauncher` and `SafeWebViewHelper` exist specifically to prevent the app from opening arbitrary or attacker-controlled URLs/schemes from dynamic content (e.g. a malicious link embedded in a message or document) — a boundary that's easy to overlook until it's exploited.

---

## 14. Test Coverage Map

```mermaid
flowchart LR
    subgraph Covered["Automated — covered"]
        T1[AppVersionComparator]
        T2[SafeUrlLauncher.isAllowed]
        T3[RateLimitHelper]
        T4[DioExceptionMapper]
        T5[Log Redaction Layer]
        T6[UserModel.toStorageJson]
        T7[AuthRepository]
        T8[Widget smoke tests]
    end

    subgraph Gap["Not yet automated"]
        G1[CheckoutCubit]
        G2[BookingHistoryCubit]
        G3[Office BLoCs]
        G4[Full UI journeys]
    end
```

Documenting coverage gaps explicitly — rather than only showing what's covered — reflects the same trade-offs-first engineering approach used throughout this documentation set: a clear, honest picture of where the system stands, not just where it succeeds.
