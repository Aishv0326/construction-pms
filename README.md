# Construction Project Management System (CPMS)

A full-featured, enterprise-grade **SaaS platform** for construction businesses — designed to replace manual Excel-based operations with a centralized, role-based system covering project lifecycle management, financial accounting, inventory tracking, employee management, and client portal access.

> **Status:** Actively in development — backend microservices architecture built, Identity Service complete, remaining services in progress.

---

## Why I'm Building This

Construction businesses in Canada still manage budgets, inventory, attendance, and invoicing through spreadsheets. I saw this firsthand — data duplication, no real-time cost visibility, no structured approval workflows, no client-facing progress tracking.

CPMS is my solution to that problem. V1 is a fully functional single-tenant deployment. The architecture is intentionally designed to support **multi-tenant SaaS in V2 with zero schema migration** — every table includes `CompanyId` from day one. The long-term vision is to commercialize this as a subscription-based platform serving multiple construction businesses.

---

## Architecture

The system follows a **Microservices architecture** with an **API Gateway pattern**. Each service is independently deployable, owns its own database, and communicates via REST APIs (synchronous) or Azure Service Bus (asynchronous for events and notifications).

```
                                ┌─────────────────┐
                                │    React.js      │
                                │   (Frontend)     │
                                └────────┬────────┘
                                         │
                                ┌────────▼────────┐
                                │   API Gateway    │
                                │   (Ocelot)       │
                                │   JWT Validation │
                                └────────┬────────┘
                                         │
                 ┌───────────┬───────────┼───────────┬───────────┐
                 │           │           │           │           │
          ┌──────▼──┐ ┌──────▼──┐ ┌──────▼──┐ ┌──────▼──┐ ┌──────▼──────┐
          │Identity │ │Project  │ │Finance  │ │Inventory│ │Notification │
          │Service  │ │Service  │ │Service  │ │Service  │ │Service      │
          └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └──────┬──────┘
               │           │           │           │             │
          ┌────▼────┐ ┌────▼────┐ ┌────▼────┐ ┌────▼────┐       │
          │identity │ │project  │ │finance  │ │inventory│  Azure Service
          │  _db    │ │  _db    │ │  _db    │ │  _db    │     Bus
          └─────────┘ └─────────┘ └─────────┘ └─────────┘
                               │
                          ┌────▼────┐
                          │  Redis  │
                          │ Cache + │
                          │Blacklist│
                          └─────────┘
```

**Key architectural principle:** No service directly queries another service's database. Cross-service data needs are satisfied through API calls only — ensuring loose coupling and independent deployability.

---

## Tech Stack

| Layer | Technology |
|:---|:---|
| **Backend** | .NET Core 8, ASP.NET Core Web API |
| **Frontend** | React.js |
| **Database** | SQL Server (database-per-service) |
| **Cache** | Redis — dashboard caching + JWT token blacklist |
| **ORM** | Entity Framework Core 8 |
| **Authentication** | ASP.NET Core Identity + JWT Bearer + Refresh Tokens |
| **API Gateway** | Ocelot — single entry point, JWT validation, routing |
| **File Storage** | Azure Blob Storage — documents, photos, invoice PDFs |
| **Secrets Management** | Azure Key Vault |
| **Messaging** | Azure Service Bus — async event-driven notifications |
| **Containerization** | Docker + docker-compose |
| **CI/CD** | GitHub Actions |
| **Hosting** | Azure App Service |
| **API Documentation** | Swagger / OpenAPI |

---

## Microservices Breakdown

| Service | Database | Responsibility |
|:---|:---|:---|
| **Identity Service** | `cpms_identity_db` | Authentication, JWT issuance, user management, role-based permission system |
| **Project Service** | `cpms_project_db` | Projects, sites, milestones, tasks, daily site logs, progress tracking |
| **Finance Service** | `cpms_finance_db` | Budgets, expenses, invoicing (GST/HST), payments, payroll, subcontractor bills |
| **Inventory Service** | `cpms_inventory_db` | Material catalog, stock tracking, purchase orders, inter-site transfers |
| **Notification Service** | — | Worker Service listening on Azure Service Bus — email and in-app alerts |
| **API Gateway** | — | Ocelot-based routing, JWT validation at gateway level, single entry point |

---

## Modules & Features

### Project & Site Management (Core Module)
This is the central module — every financial record, inventory movement, attendance entry, and document in the system is linked to a project. Progress tracking is construction-specific, not generic.

- Multi-site projects — one project can span multiple sites (e.g., Block A, North Wing, Phase 2)
- Support for Fixed Price, Percentage of Completion, and Hybrid contract types
- Project status lifecycle: Draft → Planning → Active → On Hold → Completed → Cancelled
- Assign Project Manager per project, Site Supervisor per site
- **Project types:** Residential (flats/apartments), Commercial buildings, Individual houses — each with its own progress tracking approach
- **Construction-specific progress tracking** — adapts based on project type:
  - **Residential (Flats/Apartments):** Floor-wise and unit-wise tracking — each flat on each floor is tracked individually for work completion
  - **Work-type breakdown per unit:** Every unit tracks individual work stages — foundation, structure, plumbing, electrical, plastering, painting, finishing — each with its own completion percentage
  - **Commercial / Individual builds:** Milestone-based tracking with planned vs. actual dates and delay detection
- Billable milestones — link billing amounts to milestones for milestone-based invoicing
- Delay alerts when actual end date exceeds planned end date
- Full activity log — every change recorded with user and timestamp

### Financial & Accounting
- Project budgets broken down by category — Labour, Materials, Equipment, Subcontractor, Overhead, Contingency
- Real-time budget utilization with alerts at 80% and 100% thresholds
- Professional invoice generation with configurable tax rates (GST/HST)
- Subcontractor bill approval workflow: Submitted → Approved → Paid
- Payroll calculation for both monthly salary and daily wage employees

### Inventory Management
- Centralized material catalog with supplier directory
- Per-site stock tracking with low stock alerts
- **Inter-site material transfer** — full approval workflow with dispatch and receipt confirmation
- Purchase order vs. delivery reconciliation with shortage/damage flagging

### Employee & HR
- Two employment types: Permanent (monthly) and Daily Wage (per day rate)
- Daily attendance per employee per project/site
- Leave management with approval workflow and payroll impact
- Safety certification tracking with automated expiry alerts

### Client Portal
- Clients receive login credentials to view their project progress, milestone status, and site photos
- Invoice and payment status visibility — no access to internal financials
- Email notifications on milestone completion and new invoices

### Role-Based Access Control
- Five defined roles: Admin, Project Manager, Site Supervisor, Client, Accountant
- Granular permission system — per module, per action (View, Create, Edit, Delete, Approve, Reject, Export)
- Dynamic sidebar menu generation from database-driven module hierarchy

---

## Key Design Decisions

These are the architectural choices I made and why — each one was a deliberate decision, not a default.

**Data-driven permissions over boolean columns**
Instead of `CanView`, `CanCreate`, `CanEdit` boolean columns on each role, I use one row per permission (RoleId + PermissionId). Adding a new action like "Export" is a database insert — no schema change, no migration, no redeploy.

**Module table with self-referencing hierarchy instead of an enum**
Modules need parent-child relationships for dynamic sidebar generation (e.g., Finance → Budget, Expenses, Invoicing). An enum can't represent hierarchy. The Module table with a nullable `ParentId` lets the frontend fetch the full menu tree from the API. Adding a new module = one database insert.

**PermissionAction stays as an enum**
Actions (View, Create, Edit, Delete, Approve) are truly fixed — they don't change. Enum gives compile-time safety. Modules change frequently, actions don't. Different rate of change = different solution.

**Guid for all entity IDs**
Integer IDs are sequentially predictable — an attacker can enumerate `/api/projects/1`, `/2`, `/3`. Guid is random and unguessable. Also avoids ID collision in a distributed microservices environment.

**Redis for JWT blacklist**
JWTs are stateless — a token remains valid until expiry even after the user logs out. On logout, the revoked token is stored in Redis with a TTL matching the token's remaining lifetime. Every authenticated request checks Redis first. In-memory lookup = microsecond overhead.

**ClockSkew = TimeSpan.Zero**
The default .NET JWT validation allows a 5-minute grace period after token expiry. I set this to zero — a token is rejected the exact second it expires. No grace period = tighter security.

**Database-per-service**
Each microservice owns its own SQL Server database. No shared tables, no cross-database joins. Services communicate through APIs only. This enforces loose coupling and means any service can be independently deployed, scaled, or replaced.

**CompanyId on every table from V1**
Even though V1 is single-tenant, every core table includes `CompanyId`. When V2 adds multi-tenant SaaS provisioning, it's an infrastructure change — not a schema migration.

---

## Repository Structure

The project is split across three independent repositories, each with its own CI/CD pipeline:

```
construction-pms-backend     → All .NET microservices (this repo)
construction-pms-frontend    → React.js SPA
construction-pms-infra       → Docker, CI/CD configs, documentation
```

**Backend solution structure:**

```
CPMS.sln
  ├── ApiGateway/            → Ocelot routing configuration (no controllers)
  ├── IdentityService/       → Auth, JWT, roles, permissions
  │     ├── Models/          → User, Role, Module, Permission, RolePermission, RefreshToken
  │     ├── Data/            → AppDbContext (IdentityDbContext<User, Role, Guid>)
  │     ├── Services/        → Token and auth service layer
  │     └── Controllers/     → Auth endpoints
  ├── ProjectService/        → Projects, sites, milestones, tasks
  ├── FinanceService/        → Budgets, expenses, invoices, payroll
  ├── InventoryService/      → Materials, stock, transfers
  ├── NotificationService/   → Worker Service + Azure Service Bus listener
  └── Shared/                → Class library — shared across all services
        ├── Models/          → ApiResponse<T>, PagedResponse<T>, JwtSettings, PermissionAction
        ├── Helpers/         → IJwtHelper, JwtHelper
        └── Extensions/     → ServiceCollectionExtensions
```

---

## Database Schema

Each service has its own isolated database:

| Database | Key Tables |
|:---|:---|
| `cpms_identity_db` | AspNetUsers, AspNetRoles, Modules, Permissions, RolePermissions, RefreshTokens |
| `cpms_project_db` | Projects, ProjectSites, Milestones, Tasks, SiteLogs |
| `cpms_finance_db` | Budgets, Expenses, Invoices, Payments, Payroll |
| `cpms_inventory_db` | Materials, Stock, PurchaseOrders, Transfers |

---

## Security

- JWT Bearer authentication with short-lived access tokens (60 min) and 7-day refresh tokens
- Redis-backed token blacklist for immediate logout enforcement
- Role-based access control — every API endpoint validates permissions
- Azure Key Vault for all production secrets
- Private file URLs via time-limited SAS tokens (Azure Blob Storage)
- HTTPS enforced across all environments
- Bcrypt password hashing with complexity requirements
- Audit logging on all create, update, and delete operations

---

## Caching Strategy

| What | TTL | Why |
|:---|:---|:---|
| Dashboard summary data | 5 minutes | Avoids heavy SQL aggregation on every page load |
| Project and client lists | Until invalidated | Cached per company, invalidated on create/update |
| JWT blacklist entries | Token's remaining lifetime | Enables instant logout without database dependency |
| Report results | 10 minutes | Same parameters = cached response |

---

## Local Development

**Prerequisites:** Docker Desktop, .NET 8 SDK, Node.js, Visual Studio 2022 / VS Code

```bash
# Start infrastructure
cd infra
docker-compose up -d
# Starts: SQL Server 2022 (port 1433) + Redis (port 6379)

# Run Identity Service
cd backend/CPMS/IdentityService
dotnet run

# API available at https://localhost:5001
# Swagger UI at https://localhost:5001/swagger
```

---

## V2 Roadmap — Multi-Tenant SaaS

The V2 phase transforms this into a fully commercialized SaaS product:

- **Tenant provisioning engine** — new business signs up, gets isolated database + portal within minutes
- **Subscription billing** via Stripe — Starter, Professional, Enterprise tiers
- **White-label branding** — each tenant sees their own logo, colors, and custom subdomain
- **Industry expansion** — Interior Design, IT Consulting, Event Management, Real Estate
- **Mobile app** — React Native for site supervisors with offline attendance sync
- **Equipment & machinery module** — tracking owned and rented equipment per project

---

## Author

**Aishvarya Shah**
Full-Stack Developer (.NET + React)

---

*This is a private repository. If you'd like a live demo or code walkthrough, feel free to reach out.*
