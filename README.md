# ERP Backend Coding Structure Standard

## Architecture: Microservice-Based + DDD-Inspired

---

## 1. Objective

This document defines the mandatory coding structure and architectural guidelines for all ERP backend microservices.

This structure ensures:

- Clear domain boundaries
- High maintainability
- Scalability
- Reduced technical debt
- Predictable module growth
- Consistency across microservices

Every backend microservice must strictly follow this structure.

---

## 2. Root Folder Structure

Each microservice (example: MRP Service) must follow:

```
mrp-service/
│
├── server.ts
├── package.json
├── tsconfig.json
├── .env
│
├── logs/
├── docs/
├── scripts/
├── migrations/
│
└── src/
```

| Folder | Purpose |
|---|---|
| `server.ts` | Application entry point |
| `logs/` | Runtime logs |
| `docs/` | Swagger / API documentation |
| `scripts/` | Seeders, utilities, migration helpers |
| `migrations/` | Sequelize migration files |
| `src/` | All application source code |

> ⚠️ No business logic should exist outside `src`.

---

## 3. src Folder Structure

```
src/
│
├── app.ts
│
├── api/
│   └── routes/
│       └── index.ts
│
├── bootstrap/
│   ├── database.ts
│   ├── redis.ts
│   ├── queue.ts
│   └── index.ts
│
├── common/
│   ├── errors/
│   ├── middleware/
│   ├── utils/
│   └── constants/
│
├── config/
│   └── index.ts
│
├── infrastructure/
│   ├── database/
│   │   ├── models/
│   │   └── index.ts
│   ├── redis/
│   ├── queue/
│   ├── email/
│   └── pdf/
│
└── modules/
    ├── product/
    ├── purchase-order/
    └── inwards/
```

---

## 4. Entry Layer

### 4.1 server.ts

Responsibilities:
- Load environment variables
- Execute bootstrap
- Start HTTP server

```ts
import { createApp } from "./src/app";
import { bootstrap } from "./src/bootstrap";

async function start() {
  await bootstrap();
  const app = createApp();
  app.listen(process.env.PORT, () => {
    console.log(`Service running on port ${process.env.PORT}`);
  });
}

start();
```

> No business logic allowed here.

### 4.2 app.ts

Responsibilities:
- Create Express app
- Mount middleware
- Mount routes
- Register global error handler

---

## 5. Layer Responsibilities

### 5.1 API Layer — `api/routes/index.ts`

Centralized mounting point for all module routes. No direct route mounting in `app.ts` except this index.

### 5.2 Bootstrap Layer — `bootstrap/`

Responsible for initializing infrastructure before server start. Server must not start until bootstrap completes.

```
bootstrap/
├── database.ts
├── redis.ts
├── queue.ts
└── index.ts
```

### 5.3 Common Layer — `common/`

Reusable logic shared across the entire microservice.

**`errors/`** — All custom error classes.
- All errors must extend a `BaseError`.
- No raw `throw new Error()` in services.

**`middleware/`** — Reusable middleware such as:
- `auth.middleware.ts`
- `validation.middleware.ts`
- `global-error-handler.ts`
- No business logic inside middleware.

**`utils/`** — Shared utilities:
- `asyncHandler`
- `successResponse`
- `paginationHelper`
- `dateHelper`
- `validatorHelper`

All controllers must use a consistent response format.

**`constants/`** — All reusable application constants.

```ts
export const STATUS = {
  ACTIVE: "ACTIVE",
  INACTIVE: "INACTIVE",
  CANCELLED: "CANCELLED",
} as const;
```

No hardcoded status strings, magic numbers, or inline business constants anywhere outside this folder.

### 5.4 Configuration Layer — `config/`

All configuration must be centralized here.

- No direct `process.env` usage outside `config/`.
- No hardcoded credentials or ports.
- No inline configuration values.

### 5.5 Infrastructure Layer — `infrastructure/`

Handles external systems only. No business logic allowed.

**Database:**
```
infrastructure/database/
├── models/
└── index.ts
```
- Sequelize models are defined here.
- Associations are defined centrally.
- Modules must NOT define their own models.

---

## 6. Domain Layer — `modules/`

Each domain module must follow this structure:

```
modules/
└── inwards/
    ├── inwards.routes.ts
    ├── inwards.controller.ts
    ├── inwards.service.ts
    ├── inwards.validator.ts
    └── inwards.types.ts
```

| File | Responsibility |
|---|---|
| `routes` | Define endpoints only |
| `controller` | Handle request/response |
| `service` | Business logic |
| `validator` | Input validation |
| `types` | Module-specific TypeScript types |

If a module contains subdomains, use nested submodule folders:

```
purchase-order/
├── approval/
└── cancellation/
```

Each submodule must follow the same structure internally.

---

## 7. Coding Discipline & Enforcement Rules

### 7.1 Naming Conventions

**Folders** — Lowercase, kebab-case only.

```
✅ purchase-order/
✅ product-management/

❌ PurchaseOrder/
❌ productManagement/
❌ purchase order/
```

**Files** — Kebab-case with required suffixes:
- `.routes.ts`
- `.controller.ts`
- `.service.ts`
- `.validator.ts`
- `.types.ts`

### 7.2 Slug Usage Standard

Where business identity matters (roles, types, statuses, categories), use **slug-based identifiers** instead of numeric IDs.

```ts
// ✅ Correct
roleSlug = "SUPER_ADMIN"

// ❌ Incorrect
roleId = 1
```

Slugs must be centralized in `common/constants/`.

### 7.3 Soft Delete Standard

Soft delete must use Sequelize's built-in `paranoid: true`. No manual `isDeleted` flags or custom delete logic.

```ts
const User = sequelize.define("User", {}, {
  paranoid: true,
});
```

### 7.4 Microservice Communication Rules

**Prohibited:**
- Cross-database communication between microservices
- Direct table querying across services
- Shared schema dependency

**Allowed:**
- API-to-API communication only
- Event-based communication (if implemented)

Each microservice must fully own its database.

### 7.5 Migration Discipline

Every Sequelize model must have a corresponding migration file.

- No `sequelize.sync({ alter: true })` in production
- No manual DB modifications
- All schema changes via migrations
- Migration naming format: `YYYYMMDDHHMMSS-create-table-name.ts`

### 7.6 Database Indexing Standard

Each table must define proper indexes.

Mandatory indexes:
- Primary key
- Foreign keys
- Slug columns
- Status columns
- Frequently filtered columns

```ts
indexes: [
  { fields: ["slug"], unique: true },
  { fields: ["status"] }
]
```

---

## 8. Mandatory Rules

1. No business logic inside routes.
2. No DB queries inside controllers.
3. No hardcoded values (IDs, strings, ports, secrets).
4. All reusable constants centralized in `common/constants/`.
5. Slugs used where business identity requires it.
6. Soft delete via Sequelize `paranoid: true` only.
7. No cross-service DB access — API or event-based only.
8. Every Sequelize model must have a migration file.
9. Proper database indexing required on all tables.
10. All folder and file naming conventions must be followed.
11. No direct `process.env` usage outside `config/`.
12. Errors must use custom error classes extending `BaseError`.
13. All responses must follow the common response format.
14. Each module must define its own types file.

---

## 9. Definition of Module Completion

A module is considered **complete** only if all of the following are satisfied:

- [ ] Routes implemented
- [ ] Controller implemented
- [ ] Service implemented
- [ ] Validation implemented
- [ ] Types defined
- [ ] No hardcoded values
- [ ] Slugs properly implemented where required
- [ ] Migration created for each model
- [ ] Soft delete configured correctly (`paranoid: true`)
- [ ] Required indexes defined
- [ ] Errors handled with custom error classes
- [ ] Fully tested manually
- [ ] API documented in `docs/`

If any item above is missing, the module **cannot** be marked complete.

---

> This coding standard is mandatory for all ERP backend services.