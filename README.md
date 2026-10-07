# Finance and Daily Collection Management

## Project Case Study

A full-stack platform for lenders who give short-term loans repaid in daily instalments collected in person. Office staff manage customers, loans and accounts in a React web application; field agents follow a daily route and record payments in a Flutter mobile app that keeps working without internet. Every disbursement and collection is posted to the books automatically, so the accounts stay balanced without manual bookkeeping. One installation serves several lending businesses, each with its own isolated data.

## Business Problem

Daily collection operations need consistent records of loans, payments, outstanding balances and agent activity, and the accounts have to match what was collected in the field. Field agents also need to record collections when internet access is unavailable, and managers need up-to-date figures on collections, overdue loans and portfolio risk.

## Core Features

**Customers and loans**
- Customer management with KYC documents
- Loan products, and loans created directly or through an approval and staged-disbursement workflow
- Daily repayment schedules generated for each loan
- Loan lifecycle actions: approve, reject, disburse, close, mark defaulted, classify as non-performing (NPA), write off, foreclose and reschedule

**Collections**
- Daily collection recording and outstanding balance tracking
- Bulk collection entry for many borrowers at once
- Payments applied to the oldest unpaid instalments first; loans close automatically when fully repaid
- Areas, area groups, agent assignments and daily route plans for field agents
- Offline mobile collections with later synchronization

**Accounting**
- Automatic double-entry journal entries for disbursements and collections
- Chart of accounts, manual journal entries, and income and expense transactions
- Bank reconciliation and fiscal periods
- Financial reports: trial balance, profit and loss, balance sheet, loan outstanding and collection ageing
- Dashboards and analytics: portfolio KPIs, portfolio-at-risk and collection efficiency

**Platform**
- Multiple lending businesses (tenants) on one installation, with isolated data
- Super-admin portal for tenants, subscription plans and billing
- Role-based access and permission checks
- Audit log, notifications and global search
- CSV, Excel and PDF exports

## Application Screenshots

*Screenshots use synthetic demo data: fictional customers, phone numbers and amounts.*

### Web Application

| Dashboard | Customer management |
|---|---|
| ![Dashboard](01-dashboard.png) | ![Customer management](02-customers.png) |
| **Loan management** | **Loan details** |
| ![Loan management](06-loans.png) | ![Loan details](03-loan-details.png) |
| **Daily collection entry** | **Bulk collection** |
| ![Collection entry](04-collection-entry.png) | ![Bulk collection](05-bulk-collection.png) |

### Mobile Application (field agents)

| Sign in | Home | Today's route |
|---|---|---|
| <img src="00-login.png" alt="Agent sign-in" width="250"> | <img src="01-home.png" alt="Agent home" width="250"> | <img src="02-route.png" alt="Collection route overview" width="250"> |
| **Route stops** | **Today's collections** | **Record a collection** |
| <img src="03-route-stops.png" alt="Route stops and collection actions" width="250"> | <img src="04-collections-today.png" alt="Today's collections" width="250"> | <img src="05-collect.png" alt="Record collection" width="250"> |

## Technology Stack

| Area | Technologies |
|---|---|
| Backend | Java 17, Spring Boot 3.3, Spring Security |
| Authentication | JWT access and refresh tokens, BCrypt |
| Database | PostgreSQL (partitioned journal table, materialized views for reports) |
| Persistence | Spring Data JPA, Hibernate |
| Database migrations | Flyway |
| Mapping and rate limiting | MapStruct, Lombok, Bucket4j |
| Web frontend | React 18, TypeScript, Vite, Tailwind CSS |
| State and API calls | React Context, Axios |
| Forms and charts | React Hook Form, Recharts |
| Mobile | Flutter, Dart, Riverpod, go_router, Dio, SQLite |
| Backend testing | JUnit 5, Mockito, AssertJ |
| Build and deployment | Maven, shell and Windows batch scripts (web app packaged into the backend JAR) |

## Architecture

### System overview

```mermaid
flowchart TD
    subgraph Clients["Clients"]
        Web["React web application<br/>office staff"]
        subgraph MobileApp["Flutter mobile application · field agents"]
            Mobile["Screens and state<br/>Riverpod"]
            Queue[("SQLite<br/>offline collection queue")]
        end
    end

    subgraph Backend["Spring Boot backend · modular monolith"]
        Security["Security filters<br/>JWT authentication · rate limiting"]
        Controllers["REST controllers<br/>input validation · permission checks"]
        subgraph Modules["Business modules"]
            Lending["Customers · Loans<br/>Collections · Routes"]
            Accounting["Journals · Ledger<br/>Reports"]
            Platform["Users and roles · Tenants<br/>Subscriptions · Audit"]
        end
        Jobs["Scheduled jobs<br/>overdue marking · report refresh"]
    end

    subgraph Data["PostgreSQL"]
        Tables[("Tenant-scoped tables")]
        Journal[("Journal entries<br/>partitioned by month")]
        Views[("Materialized report views")]
    end

    Web -->|"HTTPS · JSON · JWT"| Security
    Mobile -->|"HTTPS · JSON · JWT"| Security
    Mobile <-->|"save when offline"| Queue
    Queue -->|"sync when online"| Security
    Security --> Controllers --> Modules
    Lending -->|"posts double-entry journals"| Accounting
    Modules --> Tables
    Accounting --> Journal
    Jobs --> Tables
    Jobs --> Views
    Accounting -.->|"reads"| Views
```

- **Clients:** office staff use the React web application; field agents use the Flutter app, which saves collections in a local SQLite queue when there is no connection and synchronizes them later.
- **Backend:** one Spring Boot application organised as a modular monolith, with a module per business area. Each module follows the same layers: controller, service (business rules and transactions), repository and entity, with MapStruct for DTO mapping.
- **Security:** every request passes JWT authentication and rate limiting; controllers validate input and check permissions before any business logic runs.
- **Data:** PostgreSQL with Flyway-versioned migrations. Every business table carries a tenant id; journal entries are partitioned by month; financial reports read materialized views so they stay fast as data grows.
- **Background jobs:** overdue instalments are marked nightly and report views are refreshed every two hours.
- **Deployment:** the web application is built into the backend JAR, so one process serves both the API and the web interface.

### Recording a collection, step by step

```mermaid
sequenceDiagram
    autonumber
    actor Agent as Field agent
    participant App as Mobile or web app
    participant API as Spring Boot API
    participant Service as Collection service
    participant DB as PostgreSQL

    Agent->>App: Enter today's payment for a loan
    alt No internet connection (mobile app)
        App->>App: Save to the SQLite queue and sync later
    end
    App->>API: POST /api/collections with JWT
    API->>API: Verify token and tenant, validate input, check permission
    API->>Service: Record collection
    Note over Service,DB: One database transaction
    Service->>DB: Check the loan is active and disbursed, amount within outstanding balance
    Service->>DB: Save the collection and reduce the outstanding balance
    Service->>DB: Post journal entry: debit Cash, credit Loan Receivable
    Service->>DB: Mark the oldest unpaid instalments as paid
    Service-->>API: Commit, or roll everything back if any step fails
    API-->>App: 201 Created with the saved collection
```

A payment either updates the loan, the books and the instalment schedule together, or changes nothing. When a balance reaches zero, the loan closes automatically in the same transaction.

## Business Rules

- A customer can have only one active loan at a time; the database enforces this with a partial unique index.
- A new loan can be created after the existing loan is completed.
- Loan records track the principal, the upfront commission, the net disbursed amount, the daily collection amount and the repayment period.
- A collection is accepted only for an active, disbursed loan, and only for a positive amount that does not exceed the remaining outstanding balance.
- Payments are applied to the oldest unpaid instalments first; a loan closes automatically when its balance reaches zero.
- Each collection records its date, amount, notes and the user who recorded it.
- Every disbursement and collection posts a balanced journal entry; edits and deletions post reversing entries instead of changing past records.
- Every record belongs to one lending business (tenant), and data access is scoped to it.
- Access to operations is controlled through roles and permissions.

## Engineering Highlights

- JWT authentication with silent token refresh; concurrent requests wait for a single refresh
- Server-side role and permission checks on business endpoints
- Multi-tenant data isolation through tenant-scoped queries and a Hibernate filter
- Double-entry accounting with immutable journal entries and reversal-based corrections
- Monetary amounts held as exact decimals (BigDecimal) throughout
- Database-enforced business rules (one active loan per customer)
- Monthly-partitioned journal table and scheduled materialized views for reports
- Request rate limiting
- Versioned database migrations
- DTO mapping with MapStruct
- Service-layer transaction management
- Offline collection queue and synchronization
- Secrets kept out of source control: database password and JWT secret come from environment variables or a local, untracked configuration file
- Automated collection-service tests

## Testing and Validation

The backend includes 29 unit tests for the collection service using JUnit 5, Mockito and AssertJ, with mocked repositories. They cover validation rules, balance updates, automatic loan closure, allocation of payments to instalments, journal posting failures, edits and deletions with reallocation, and bulk collection results.

### Current Coverage Limits

- Backend automated tests focus on the collection service.
- Broader integration and end-to-end testing remain areas for improvement.

## Known Limitations and Next Steps

- **Concurrent payments on the same loan:** two payments recorded at the same moment are not yet serialized. Next step: row-level locking or optimistic versioning on the loan balance.
- **Duplicate submissions:** a retried request or a re-sent offline collection can be recorded twice. Next step: idempotency keys generated by the web and mobile clients.
- **Accounting for loan adjustments:** write-offs, foreclosures, waivers and reschedules update loan balances but do not yet post journal entries. Next step: journal postings for these actions.
- **Test coverage:** extend automated tests to loan, accounting and security flows, and add integration tests against a real database.

## My Contribution

Handled requirements, application design, backend and frontend development, mobile functionality, testing, deployment, and server maintenance.

## About This Repository

This repository presents the project's business context, architecture, and engineering approach as a portfolio case study.
