# Aryanveda-DMS — Complete System Architecture & Workflow Diagram

---

## 1. System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           BROWSER (Client)                                  │
│                                                                             │
│   Next.js 16 App Router  ·  React 19  ·  Tailwind 4  ·  Inter Font        │
│   ┌──────────┐  ┌──────────────┐  ┌─────────────┐  ┌───────────────────┐  │
│   │  Zustand │  │ TanStack     │  │ React Hook  │  │  Axios Client     │  │
│   │ AuthStore│  │ Query Cache  │  │ Form + Zod  │  │  withCredentials  │  │
│   └──────────┘  └──────────────┘  └─────────────┘  └───────────────────┘  │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │  HTTP/HTTPS  (httpOnly cookies for JWT)
                              │  CORS: credentials: true
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     EXPRESS BACKEND  (Port 5000)                            │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    MIDDLEWARE PIPELINE                               │   │
│  │  cors() → express.json() → cookieParser() → authMiddleware          │   │
│  │         ↓ (on protected routes)                                     │   │
│  │  allowRoles([...]) / denyRoles([...])  →  hierarchyScopeMiddleware()│   │
│  │         ↓                                                           │   │
│  │                    ROUTE HANDLER                                    │   │
│  │         ↓                                                           │   │
│  │                     SERVICE LAYER                                   │   │
│  │         ↓                                                           │   │
│  │                   REPOSITORY LAYER                                  │   │
│  │         ↓                                                           │   │
│  │                errorHandler (global catch)                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │  Mongoose ODM  (connection pooling)
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MONGODB (Replica Set)                                     │
│                                                                             │
│   users  ·  sku_master  ·  inventory  ·  inventory_ledger                  │
│   orders  ·  display_audits                                                 │
│                                                                             │
│   Transactions: Multi-SKU orders / Inventory + Ledger writes                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Organization Hierarchy Tree

```
Super Stockist (ss-001)
│    Can see: ALL own downstream entities
│    Cannot see: other Super Stockists
│
└── Distributor (dist-001)
│    │    Can see: own Sales Agents + own Retailers
│    │    Cannot see: other Distributors
│    │
│    ├── Sales Agent (agent-001)
│    │    │    Can see: assigned Retailers only
│    │    │    Cannot see: other Distributors or Agents
│    │    │
│    │    └── Retailer (ret-001)
│    │             Can see: self only
│    │             No upstream visibility
│    │
│    └── Sales Agent (agent-002)
│         └── Retailer (ret-002)
│
└── Distributor (dist-002)
     └── ...
```

**parentId chain in MongoDB:**
```
Retailer.parentId   → Sales Agent entityId
Sales Agent.parentId → Distributor entityId
Distributor.parentId → Super Stockist entityId
Super Stockist.parentId = null  (root)
```

---

## 3. Frontend Layer Architecture

```
src/
├── app/                          ← Next.js App Router pages
│   ├── layout.tsx                ← Root layout (wraps Providers)
│   ├── page.tsx                  ← Root redirect → /dashboard
│   ├── (auth)/
│   │   └── login/page.tsx        ← Login form (react-hook-form + zod)
│   └── dashboard/
│       ├── page.tsx              ← Server component (metadata)
│       └── dashboard-client.tsx  ← Client component (role KPIs + AppLayout)
│
├── middleware.ts                 ← Next.js Edge middleware (route guard)
│
├── components/
│   ├── providers.tsx             ← QueryClientProvider + AuthProvider
│   ├── providers/
│   │   └── AuthProvider.tsx      ← Hydrates auth state via GET /api/auth/me
│   └── layout/
│       ├── AppLayout.tsx         ← Sidebar + TopNavbar shell
│       ├── Sidebar.tsx           ← Role-aware nav items
│       ├── TopNavbar.tsx         ← User chip + logout
│       └── nav-items.ts          ← Nav config keyed by UserRole
│
├── store/
│   └── auth.store.ts             ← Zustand: user, isAuthenticated, isLoading
│
├── services/
│   └── auth.service.ts           ← login(), logout(), me(), refresh()
│
├── lib/
│   └── api-client.ts             ← Axios: baseURL, withCredentials, interceptors
│
└── types/
    └── index.ts                  ← UserRole, IUser, ISku, IOrder, etc.
```

---

## 4. Backend Layer Architecture

```
src/
├── server.ts                     ← Bootstrap: connect DB → app.listen
├── app.ts                        ← Express: cors, middleware, routes, errorHandler
│
├── config/
│   ├── env.ts                    ← All env vars as typed config object
│   └── database.ts               ← connectDatabase() / disconnectDatabase()
│
├── types/
│   ├── index.ts                  ← UserRole, IUser, ISku, IOrder, enums...
│   └── express.d.ts              ← Augments Request with req.user: TokenPayload
│
├── utils/
│   ├── base.repository.ts        ← Generic CRUD: create/find/update/delete
│   ├── asyncHandler.ts           ← Wraps async handlers → next(err)
│   └── logger.ts                 ← Winston (JSON, structured)
│
├── middlewares/
│   ├── auth.middleware.ts         ← Verify access_token cookie → attach req.user
│   ├── roleGuard.middleware.ts    ← allowRoles([]) / denyRoles([]) factories
│   ├── hierarchyScope.middleware.ts ← BFS tree walk, blocks cross-hierarchy
│   └── error.middleware.ts        ← Global error handler (uses err.statusCode)
│
└── modules/
    ├── auth/
    │   ├── auth.service.ts       ← login, refresh, getMe, sign/verify tokens
    │   ├── auth.controller.ts    ← loginHandler, refreshHandler, meHandler...
    │   └── auth.routes.ts        ← POST /login, POST /refresh, POST /logout, GET /me
    │
    ├── user/
    │   ├── user.model.ts         ← Mongoose schema
    │   └── user.repository.ts    ← findByEmail, findDownstream, activate...
    │
    ├── sku/                      ← Phase 3
    ├── inventory/                ← Phase 4
    ├── order/                    ← Phase 5 + 6
    └── display-audit/            ← Phase 7
```

---

## 5. Request Lifecycle — Every API Call

```
Browser
  │
  │  HTTP request (cookie: access_token=<jwt>)
  ▼
Next.js Edge Middleware (middleware.ts)
  │  — checks access_token cookie exists
  │  — if missing → redirect to /login
  │  — if present → allow request
  ▼
Axios apiClient (lib/api-client.ts)
  │  — baseURL: http://localhost:5000/api
  │  — withCredentials: true  (sends cookies cross-origin)
  ▼
Express Backend
  │
  ├─ cors()                 ← sets Access-Control-Allow-Origin + Allow-Credentials
  ├─ express.json()         ← parse request body
  ├─ cookieParser()         ← parse cookies into req.cookies
  │
  ├─ (public routes skip auth below)
  │
  ├─ authMiddleware
  │    reads: req.cookies.access_token
  │    verifies: jwt.verify(token, config.jwtSecret)
  │    attaches: req.user = { userId, role, parentId }
  │    on fail: 401 { success: false, message: 'Token expired' | 'Invalid token' }
  │
  ├─ allowRoles([UserRole.SUPER_STOCKIST])   ← optional, per route
  │    checks: req.user.role in allowed list
  │    on fail: 403 { success: false, message: 'insufficient role' }
  │
  ├─ hierarchyScopeMiddleware()              ← optional, per route
  │    reads: req.params.entityId | req.body.entityId
  │    BFS: walks parentId tree from req.user.userId
  │    on fail: 403 { success: false, message: 'outside hierarchy scope' }
  │
  ├─ Route Handler (controller)
  │    — thin: reads req, calls service, returns res.json
  │
  ├─ Service Layer
  │    — all business logic lives here
  │    — calls repository methods only
  │
  ├─ Repository Layer (extends BaseRepository)
  │    — all DB queries live here
  │    — Mongoose models accessed via this.model
  │
  ├─ MongoDB  
  │    — query executed, documents returned
  │
  └─ errorHandler
       on throw: catches err.statusCode + err.message
       returns: { success: false, message }
```

---

## 6. Authentication Flow (Complete)

```
┌──────────────────────────────────────────────────────────────────┐
│  LOGIN FLOW                                                      │
│                                                                  │
│  User fills login form                                           │
│       ↓                                                          │
│  Zod validates email + password (client-side)                    │
│       ↓                                                          │
│  POST /api/auth/login  { email, password }                       │
│       ↓                                                          │
│  auth.service.login()                                            │
│       ├─ userRepo.findByEmail(email)   → MongoDB users query     │
│       ├─ bcrypt.compare(password, user.password)                 │
│       ├─ if inactive → 403                                       │
│       ├─ jwt.sign({ userId, role, parentId }, jwtSecret)         │
│       │     → access_token  (expires: 15m)                       │
│       └─ jwt.sign({ userId, role, parentId }, jwtRefreshSecret)  │
│             → refresh_token (expires: 7d)                        │
│       ↓                                                          │
│  auth.controller sets cookies:                                   │
│       Set-Cookie: access_token=<jwt>; HttpOnly; SameSite=Strict  │
│       Set-Cookie: refresh_token=<jwt>; HttpOnly; Path=/api/auth/refresh
│       ↓                                                          │
│  Response: 200 { success: true, data: { user: {...} } }          │
│       ↓                                                          │
│  Frontend: useAuthStore.setUser(user) → router.push('/dashboard')│
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  SESSION HYDRATION (every page load)                             │
│                                                                  │
│  AuthProvider mounts                                             │
│       ↓                                                          │
│  authService.me()  →  GET /api/auth/me                           │
│       ↓                                                          │
│  authMiddleware verifies access_token cookie                     │
│       ├─ valid → meHandler → getMe(req.user.userId)              │
│       │          userRepo.findByEntityId() → MongoDB             │
│       │          responds 200 { user }                           │
│       │          setUser(user) in Zustand                        │
│       └─ invalid/expired → 401                                   │
│                setUser(null)  (stays on /login)                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  TOKEN REFRESH FLOW                                              │
│                                                                  │
│  POST /api/auth/refresh                                          │
│       ↓                                                          │
│  reads: req.cookies.refresh_token                                │
│       ↓                                                          │
│  verifyRefreshToken(token)                                       │
│  userRepo.findByEntityId() → verify still active                 │
│       ↓                                                          │
│  sign new access_token + refresh_token                           │
│  set both cookies                                                │
│  → 200 { success: true }                                         │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  LOGOUT FLOW                                                     │
│                                                                  │
│  POST /api/auth/logout                                           │
│       ↓                                                          │
│  clearCookie(access_token)                                       │
│  clearCookie(refresh_token)                                      │
│       ↓                                                          │
│  Frontend: useAuthStore.logout() → router.push('/login')         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 7. Hierarchy Scope Enforcement

```
Every protected entity request goes through:

req.user.userId = "dist-001"  (Distributor)
req.params.entityId = ???

hierarchyScopeMiddleware() BFS walk:
  Start: dist-001
  Children of dist-001: [agent-001, agent-002]
  Children of agent-001: [ret-001]
  Children of agent-002: [ret-002]
  Accessible set: { dist-001, agent-001, agent-002, ret-001, ret-002 }

  targetEntityId = "ret-001"  → ✅ ALLOW
  targetEntityId = "dist-002" → ❌ BLOCK 403
  targetEntityId = "ss-001"   → ❌ BLOCK 403
  targetEntityId = "ret-009"  → ❌ BLOCK 403 (not in tree)

Special rules:
  SALES_AGENT + RETAILER → can only access themselves (no downstream check)
  Any role              → always allowed to access own entityId
```

---

## 8. Database Collections & Indexes

### 8.1 users
```
Field        Type      Index
──────────── ───────── ─────────────────────────────
entityId     String    UNIQUE (primary identifier)
name         String
email        String    UNIQUE
password     String    (bcrypt hash)
role         String    INDEX (enum: 4 roles)
parentId     String    INDEX (null for root)
isActive     Boolean   INDEX
phone        String?

Compound indexes:
  (parentId, role)    ← hierarchy + role queries
  (role, isActive)    ← active users by role
```

### 8.2 sku_master
```
Field            Type      Index
──────────────── ───────── ──────────────
skuId            String    UNIQUE
name             String
description      String
price            Number
displayRequired  Number
schemeEligible   Boolean
isActive         Boolean   INDEX
createdBy        String    (entityId of SS)
```

### 8.3 inventory
```
Field              Type      Index
────────────────── ───────── ────────────────────────────
entityId           String    COMPOUND UNIQUE (entityId+skuId)
skuId              String    COMPOUND UNIQUE (entityId+skuId)
quantity           Number    (min: 0 — never negative)
lowStockThreshold  Number

Critical: Atomic conditional update pattern
  db.inventory.findOneAndUpdate(
    { entityId, skuId, quantity: { $gte: deductAmount } },
    { $inc: { quantity: -deductAmount } }
  )
  → returns null if insufficient stock (first-write-wins)
```

### 8.4 inventory_ledger
```
Field           Type      Index
─────────────── ───────── ──────────────────────────────
entityId        String    INDEX
skuId           String    INDEX
quantityDelta   Number    (positive = receive, negative = ship)
reason          String    (enum: 6 ledger reasons)
referenceOrderId String   INDEX
timestamp       Date

Immutable — never updated or deleted after creation
```

### 8.5 orders
```
Field          Type      Index
────────────── ───────── ──────────────────────────────
orderId        String    UNIQUE
type           String    (primary | secondary)
status         String    (CREATED→APPROVED→DISPATCHED→DELIVERED)
fromEntityId   String    INDEX
toEntityId     String    INDEX
items[]        Array     [{ skuId, quantity, price }]
totalAmount    Number
idempotencyKey String    UNIQUE (prevents duplicate orders)
createdBy      String
```

### 8.6 display_audits
```
Field              Type      Index
────────────────── ───────── ─────────────────────
retailerId         String    INDEX
salesAgentId       String    INDEX
skuId              String
actualDisplay      Number
requiredDisplay    Number
compliancePercent  Number    = (actual/required) * 100
imageUrl           String?
auditDate          Date
```

---

## 9. Role-Based Workflows

---

### 9.1 SUPER STOCKIST Workflow

```
LOGIN → Dashboard
  │
  ├── Dashboard KPIs
  │    ├─ Total Distributors → GET /api/users?role=distributor&parentId=<myId>
  │    ├─ Primary Orders Pending → GET /api/orders?type=primary&status=CREATED
  │    ├─ Total Inventory Value → GET /api/inventory/<myId>  (aggregate)
  │    └─ Low Stock Alerts → GET /api/inventory/<myId>?lowStock=true
  │
  ├── SKU Master  (EXCLUSIVE — only SS can create/edit)
  │    ├─ List all SKUs → GET /api/sku
  │    ├─ Create SKU   → POST /api/sku
  │    │    Controller: allowRoles([SUPER_STOCKIST])
  │    │    Body: { skuId, name, price, displayRequired, schemeEligible }
  │    │    DB: sku_master.insertOne()
  │    └─ Toggle active → PATCH /api/sku/:skuId
  │
  ├── Distributors List
  │    └─ GET /api/users?role=distributor
  │         hierarchyScopeMiddleware → only own Distributors
  │
  ├── Primary Order Approvals
  │    ├─ List pending → GET /api/orders/primary?status=CREATED
  │    └─ Approve order → POST /api/orders/:orderId/approve
  │         Service:
  │           1. Validate order status = CREATED
  │           2. Validate SS inventory has stock
  │           3. MongoDB Transaction:
  │              a. inventory.deduct(ssId, skuId, qty)    ← atomic
  │              b. inventory.add(distributorId, skuId, qty)
  │              c. ledger.write(PRIMARY_ORDER_OUT, ssId)
  │              d. ledger.write(PRIMARY_ORDER_IN, distributorId)
  │              e. order.updateStatus(APPROVED)
  └─
```

---

### 9.2 DISTRIBUTOR Workflow

```
LOGIN → Dashboard
  │
  ├── Dashboard KPIs
  │    ├─ Active Retailers → GET /api/users?role=retailer&parentId=<myAgents>
  │    ├─ Secondary Orders (Month) → GET /api/orders?type=secondary
  │    ├─ Inventory Value → GET /api/inventory/<myId>
  │    └─ Pending Orders → GET /api/orders?status=CREATED
  │
  ├── Place Primary Order (order stock from Super Stockist)
  │    └─ POST /api/orders/primary
  │         denyRoles([RETAILER, SALES_AGENT])
  │         Body: { items: [{skuId, qty}], idempotencyKey }
  │         Service:
  │           1. Validate idempotency key unique
  │           2. Check SS has inventory
  │           3. Create order with status=CREATED
  │           4. Wait for SS to APPROVE (triggers inventory transfer)
  │
  ├── Inventory Dashboard
  │    └── GET /api/inventory/<myId>
  │         Returns: all SKUs with qty + low stock flag
  │
  ├── Sales Agents Management
  │    └── GET /api/users?role=sales_agent&parentId=<myId>
  │         hierarchyScope enforced → only own agents
  │
  ├── Fulfil Secondary Order (ship to retailer)
  │    └── POST /api/orders/:orderId/dispatch
  │         Service:
  │           MongoDB Transaction:
  │             a. inventory.deduct(distributorId, skuId, qty)
  │             b. inventory.add(retailerId, skuId, qty)
  │             c. ledger.write(SECONDARY_ORDER_OUT, distibutorId)
  │             d. ledger.write(SECONDARY_ORDER_IN, retailerId)
  │             e. order.updateStatus(DISPATCHED)
  └─
```

---

### 9.3 SALES AGENT Workflow

```
LOGIN → Dashboard
  │
  ├── Dashboard KPIs
  │    ├─ Assigned Retailers → GET /api/users?role=retailer (scoped to agent)
  │    ├─ Orders This Week → GET /api/orders?createdBy=<myId>
  │    ├─ Display Audits (Month) → GET /api/display/audit?salesAgentId=<myId>
  │    └─ Avg Compliance % → computed from display audit records
  │
  ├── Retailer List
  │    └── GET /api/users?role=retailer&parentId=<myId>
  │         hierarchyScope → only own retailers
  │
  ├── Book Secondary Order (on behalf of retailer)
  │    └── POST /api/orders/secondary
  │         allowRoles([SALES_AGENT, RETAILER])
  │         Body: { retailerId, items: [{skuId, qty}], idempotencyKey }
  │         Service (CRITICAL PATH):
  │           1. hierarchyScope: retailerId must be in agent's tree
  │           2. Validate idempotency key (prevent duplicates)
  │           3. Validate distributor has stock (all SKUs atomically)
  │           4. MongoDB Transaction:
  │              a. FOR EACH SKU:
  │                 inventory.deduct(distributorId, skuId, qty)
  │                   ← atomic: { $gte: qty } guard, returns null if insufficient
  │              b. invoice / order document created
  │              c. inventory.add(retailerId, skuId, qty)  per SKU
  │              d. ledger entries written (SECONDARY_ORDER_OUT × n SKUs)
  │              e. ledger entries written (SECONDARY_ORDER_IN × n SKUs)
  │           5. On any failure → full transaction ROLLBACK → 409/422 error
  │
  └── Display Audit
       └── POST /api/display/audit
            allowRoles([SALES_AGENT])
            Body: { retailerId, skuId, actualDisplay, imageUrl? }
            Service:
              compliancePercent = (actualDisplay / sku.displayRequired) * 100
              displayAudit.create({ ...body, compliancePercent, salesAgentId })
              → Distributor can view these for their retailers
```

---

### 9.4 RETAILER Workflow

```
LOGIN → Dashboard
  │
  ├── Dashboard KPIs (minimal)
  │    ├─ My Orders (Month) → GET /api/orders?fromEntityId=<myId>
  │    ├─ Pending Deliveries → GET /api/orders?status=DISPATCHED
  │    ├─ Inventory Level → GET /api/inventory/<myId>
  │    └─ Last Order Date → last record from orders collection
  │
  ├── SKU Catalog (read-only)
  │    └── GET /api/sku?isActive=true
  │         denyRoles — cannot create/modify
  │
  ├── Order History
  │    └── GET /api/orders?fromEntityId=<myId>
  │         hierarchyScope auto-scopes to self
  │
  └── My Inventory
       └── GET /api/inventory/<myId>
            hierarchyScope: retailerId must match req.user.userId
```

---

## 10. Order Engine — State Machine

```
PRIMARY ORDER (Distributor → Super Stockist)

  CREATED ──── SS approves ──→ APPROVED ──── SS ships ──→ DISPATCHED
     │                                                         │
     │  SS rejects                              Dist confirms  │
     └──────────────────────────────────────────────────────→ DELIVERED
                                                              (or CANCELLED)

  Inventory moves on: APPROVED (deduct SS) + DISPATCHED (add Distributor)


SECONDARY ORDER (Retailer/Agent → Distributor)

  CREATED ──── Dist fulfils ──→ DISPATCHED ──── Retailer confirms ──→ DELIVERED
     │                                │
     │  Cancelled                     └── Inventory transferred on DISPATCHED
     └──→ CANCELLED

  Inventory moves on: DISPATCHED atomically (deduct Dist + add Retailer)


IDEMPOTENCY:
  Every order POST requires header: Idempotency-Key: <uuid>
  If same key arrives again → return original response (DB lookup)
  Prevents duplicate orders on network retry
```

---

## 11. API Endpoint Map (Full Planned Surface)

```
PUBLIC (no auth required)
  POST   /api/auth/login          ← login, sets httpOnly cookies
  POST   /api/auth/refresh        ← refresh tokens using refresh_token cookie
  POST   /api/auth/logout         ← clears cookies
  GET    /api/health              ← health check

PROTECTED (auth required — all below need valid access_token cookie)

AUTH
  GET    /api/auth/me             ← current user from token

USERS (Phase 3+)
  GET    /api/users               ← list users (scoped by hierarchy)
  POST   /api/users               ← create user [SS only]
  GET    /api/users/:entityId     ← get user detail (hierarchy scoped)
  PATCH  /api/users/:entityId     ← update user [SS only]
  DELETE /api/users/:entityId     ← deactivate [SS only]

SKU MASTER (Phase 3)
  GET    /api/sku                 ← list active SKUs (all roles read)
  POST   /api/sku                 ← create SKU [SS only]
  GET    /api/sku/:skuId          ← SKU detail
  PATCH  /api/sku/:skuId          ← update SKU [SS only]

INVENTORY (Phase 4)
  GET    /api/inventory/:entityId ← stock for entity (hierarchy scoped)
  PATCH  /api/inventory/adjust    ← manual adjustment [SS only]

ORDERS (Phase 5 + 6)
  POST   /api/orders/primary      ← place primary order [Distributor]
  POST   /api/orders/secondary    ← place secondary order [Agent/Retailer]
  GET    /api/orders              ← list orders (scoped)
  GET    /api/orders/:orderId     ← order detail
  POST   /api/orders/:orderId/approve    ← SS approve primary
  POST   /api/orders/:orderId/dispatch   ← Dist dispatch order
  POST   /api/orders/:orderId/deliver    ← confirm delivery
  POST   /api/orders/:orderId/cancel     ← cancel order

DISPLAY AUDIT (Phase 7)
  POST   /api/display/audit       ← create audit [Agent only]
  GET    /api/display/audit       ← list audits (scoped)
  GET    /api/display/audit/:id   ← audit detail
```

---

## 12. Frontend Route Map

```
/                    → redirects to /dashboard (or /login if no cookie)
/login               → login form (public, no auth needed)

/dashboard           → role-aware dashboard (protected)

/distributors        → SS: list own distributors
/agents              → Dist: list own sales agents
/retailers           → Dist + Agent: list retailers

/skus                → SS: full CRUD    |   Others: read-only catalog
/skus/new            → SS only: create SKU form
/skus/:id            → SKU detail

/inventory           → view inventory for own entity
/inventory/:entityId → SS/Dist view downstream entity stock

/orders              → order list (filtered by role)
/orders/new/primary  → Dist: place primary order
/orders/new/secondary → Agent/Retailer: place secondary order
/orders/:id          → order detail + status actions

/audits              → Agent: list + create display audits
/audits/new          → Agent: create audit form
/audits/:id          → audit detail

/profile             → own account info
```

---

## 13. Next.js Middleware Route Guard

```
Every browser request →  middleware.ts  (Edge Runtime)

  Checks:
    Is path /.well-known/* ?   → PASS (Chrome DevTools probes)
    Is path /_next/* ?         → PASS (static assets)
    Is path /login ?           → PASS (public)
    Is path /api/auth/* ?      → PASS (auth endpoints)

    Has access_token cookie?
      YES + path is /login     → REDIRECT /dashboard  (already logged in)
      YES + other path         → PASS (let backend verify token validity)
      NO  + any protected path → REDIRECT /login?from=<original-path>
```

---

## 14. Data Flow — Secondary Order (Critical Path, Most Complex)

```
Sales Agent submits order for Retailer

  Agent Browser
     │
     │  POST /api/orders/secondary
     │  { retailerId: "ret-001", items: [{skuId:"SKU01",qty:10},{skuId:"SKU02",qty:5}] }
     │  Header: Idempotency-Key: <uuid>
     ▼
  Next.js Middleware → access_token cookie valid → pass
     ▼
  Express
     │
     ├─ authMiddleware: req.user = { userId: "agent-001", role: SALES_AGENT }
     ├─ allowRoles([SALES_AGENT, RETAILER])
     ├─ hierarchyScopeMiddleware: "ret-001" in agent-001's tree? ✅
     │
     ▼
  orderController.createSecondary(req, res)
     │
     ▼
  orderService.createSecondary({ agentId, retailerId, items, idempotencyKey })
     │
     ├─ 1. Check idempotency: orders.findOne({ idempotencyKey })
     │       if found → return cached response (HTTP 200, same order)
     │
     ├─ 2. Find distributor: userRepo.findByEntityId(retailer.parentId.parentId)
     │       (Retailer → Agent → Distributor)
     │
     ├─ 3. START MongoDB Transaction (session)
     │
     ├─ 4. For each SKU in items[]:
     │       inventory.findOneAndUpdate(
     │         { entityId: distributorId, skuId, quantity: { $gte: reqQty } },
     │         { $inc: { quantity: -reqQty } },
     │         { session, returnDocument: 'after' }
     │       )
     │       → null result = INSUFFICIENT STOCK → throw, abort transaction
     │
     ├─ 5. Create order document (session)
     │
     ├─ 6. For each SKU: inventory.increment(retailerId, skuId, qty) (session)
     │
     ├─ 7. Write ledger entries × (2 × n SKUs):
     │       SECONDARY_ORDER_OUT per SKU (distributor)
     │       SECONDARY_ORDER_IN  per SKU (retailer)
     │
     ├─ 8. COMMIT Transaction
     │
     └─ 9. Return 201 { success: true, data: { order } }

  On ANY failure in steps 4–7:
     → ROLLBACK (all inventory changes undone, order NOT created)
     → 409 CONFLICT with message explaining which SKU was insufficient
```

---

## 15. Compliance % Formula (Display Audit)

```
compliancePercent = (actualDisplay / requiredDisplay) × 100

Example:
  SKU requires 10 units displayed
  Agent counts 7 units at retailer
  compliancePercent = (7/10) × 100 = 70%

Stored in display_audits.compliancePercent
Distributor can query: GET /api/display/audit?retailerId=ret-001
SS can query all their network's compliance
```

---

## 16. Current Build Status (Phase Tracker)

```
✅ Phase 0 — Foundation
     Backend: Express + TypeScript + MongoDB + Winston Logger
     Frontend: Next.js App Router + Tailwind + Zustand + TanStack Query

✅ Phase 1 — Data Models & Indexes
     6 Mongoose models: users, sku_master, inventory,
                        inventory_ledger, orders, display_audits
     6 Repositories (all extend BaseRepository)
     Physical indexes created & verified
     74 model/repository/transaction tests passing

✅ Phase 2 — Authentication & Hierarchy Engine
     JWT login/refresh/logout/me endpoints
     httpOnly cookie strategy (access: 15m, refresh: 7d)
     authMiddleware (verifies token → req.user)
     allowRoles() / denyRoles() middleware factories
     hierarchyScopeMiddleware() (BFS tree walk)
     Global errorHandler
     CORS configured (credentials: true)
     Next.js edge middleware route guard
     Login page UI (react-hook-form + zod + show/hide password)
     AuthProvider session hydration
     Sidebar with role-aware navigation
     Role-based dashboard KPI cards
     21 auth integration tests passing
     Total: 95 tests, 10 suites — all passing

⬜ Phase 3  — SKU Master Module
⬜ Phase 4  — Inventory Engine (atomic deduction)
⬜ Phase 5  — Primary Order Flow (SS ↔ Distributor)
⬜ Phase 6  — Secondary Order Engine (CRITICAL — 50+ parallel order tests)
⬜ Phase 7  — Display Audit Module
⬜ Phase 8  — Full Role Dashboards with live data
⬜ Phase 9  — Pagination, Rate Limiting, Input Validation Sweep
⬜ Phase 10 — Load Testing & Security Penetration Tests
⬜ Phase 11 — Pilot Rollout to ~800 Distributors
```

---

## 17. Security Model Summary

```
Layer 1 — Transport
  HTTPS in production
  CORS: only allow configured origin (CORS_ORIGIN env var)

Layer 2 — Session
  JWT in httpOnly, SameSite=Strict cookies
  No token in localStorage or JS-readable storage
  access_token expires: 15 minutes
  refresh_token expires: 7 days (httpOnly, path-restricted)

Layer 3 — Route Authentication
  authMiddleware on every /api/* route (except /auth/login, /auth/refresh, /auth/logout)
  Next.js Edge middleware blocks unauthenticated browser navigation

Layer 4 — Role Authorization
  allowRoles([...]) — whitelist specific roles per endpoint
  denyRoles([...])  — blacklist specific roles per endpoint
  No role checks ever live in controllers or UI only

Layer 5 — Hierarchy Scope
  hierarchyScopeMiddleware() on all entity-scoped routes
  BFS walk of parentId tree from caller's entityId
  Any request targeting an entity outside caller's tree → 403

Layer 6 — Data Integrity
  Idempotency key on all order mutations
  MongoDB transactions on multi-document writes
  Atomic conditional inventory deduction (never negative)
  Immutable ledger (no update/delete on inventory_ledger)
```

---

*Document based on: PRD.md · Project.md · Plan.md · Design.md · full codebase scan*
*Last updated: Phase 2 complete — 95 tests passing*
