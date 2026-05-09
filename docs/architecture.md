# Architecture Overview

> Full architecture documentation is in the academic report:
> [PFE_Mohamed_Mensi_Zaineb_Arfaoui_2025.pdf](../report/PFE_Mohamed_Mensi_Zaineb_Arfaoui_2025.pdf)
> (Chapter 2 — Sprint 0: Analysis and Specification of Requirements)
>
> This document summarizes the key architectural decisions and system structure.

---

## System Overview

The platform connects three actor types — professors, university admins, 
and CCK admins — to a shared cloud infrastructure managed by CCK 
(El Khawarizmi Computing Center) using Huawei ManageOne.

The architecture follows a **3-tier model** with a dedicated 
orchestration layer isolating the application from direct 
cloud infrastructure exposure.

---

## Logical Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Presentation Layer                     │
│                                                          │
│   ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│   │  Professor  │  │  University  │  │  CCK Admin    │  │
│   │  Interface  │  │  Admin UI    │  │  Interface    │  │
│   └─────────────┘  └──────────────┘  └───────────────┘  │
│              Angular 18 (MVVM · TypeScript)              │
└────────────────────────┬─────────────────────────────────┘
                         │ HTTP / REST / JSON
┌────────────────────────▼─────────────────────────────────┐
│                  Orchestration Layer                      │
│                                                          │
│                  Flask (Python)                          │
│                                                          │
│   - Decouples frontend from ManageOne APIs               │
│   - IAM token acquisition and lifecycle                  │
│   - ECS lifecycle calls (create/start/stop/delete)       │
│   - Real-time metrics polling                            │
│   - Scheduling job triggers                              │
└────────────────────────┬─────────────────────────────────┘
                         │ HTTP
┌────────────────────────▼─────────────────────────────────┐
│                   Application Layer                       │
│                                                          │
│              Spring Boot (Java · MVC)                    │
│                                                          │
│   ┌──────────────┐  ┌─────────────┐  ┌───────────────┐  │
│   │    Auth &    │  │ Reservation │  │   Billing &   │  │
│   │   Profile    │  │ & Scheduling│  │   Invoicing   │  │
│   │  Management  │  │   Engine    │  │    Module     │  │
│   └──────────────┘  └─────────────┘  └───────────────┘  │
│                                                          │
│   Controller → Service → Repository (Spring Data JPA)   │
└──────────┬─────────────────────────┬─────────────────────┘
           │ JDBC                    │ HTTP
┌──────────▼──────────┐  ┌──────────▼──────────────────────┐
│     Data Layer      │  │     Cloud Infrastructure         │
│                     │  │                                  │
│   MySQL Database    │  │   Huawei ManageOne APIs          │
│                     │  │   IAM · ECS · Monitoring         │
│   - Users           │  │                                  │
│   - Reservations    │  │   (External · CCK-managed)       │
│   - Subscriptions   │  │                                  │
│   - Invoices        │  │                                  │
│   - Billing Entries │  │                                  │
└─────────────────────┘  └──────────────────────────────────┘
```

---

## Physical Architecture (Deployment)

```
┌─────────────────────────────────┐
│     Client Devices (Browser)    │
│                                 │
│  Professor · University · CCK   │
└──────────────┬──────────────────┘
               │ HTTP
┌──────────────▼──────────────────┐
│    Web Server (Tomcat)          │
│                                 │
│    Angular App (served)         │
└──────────────┬──────────────────┘
               │ HTTP
┌──────────────▼──────────────────┐
│    Application Server           │
│                                 │
│    Spring Boot Backend          │
│    ├── Authentication Service   │
│    ├── Registration Service     │
│    ├── Reservation Management   │
│    ├── Resource Management      │  ← Facade pattern
│    ├── Billing Service          │
│    ├── Invoice Management       │
│    └── Approval Workflow        │
└────────┬──────────────┬─────────┘
         │ JDBC         │ HTTP
┌────────▼──────┐  ┌────▼────────────────────────┐
│  MySQL DB     │  │  CCK API (Flask)             │
│  Server       │  │                             │
│               │  │  ← External cloud component │
│  500 GB init  │  │  ← CCK Infrastructure       │
│  scalable     │  │     Management System        │
└───────────────┘  └─────────────────────────────┘
```

Development environment: shared VM — 4 vCPUs · 16 GB RAM · 
100 GB SSD · Windows Server 2019/2022.

---

## Design Patterns

### Backend — MVC (Spring Boot)

```
Request
   │
   ▼
Controller        ← receives HTTP request, delegates
   │
   ▼
Service           ← business logic, transaction management
   │
   ▼
Repository        ← Spring Data JPA, database access
   │
   ▼
MySQL Database
```

### Frontend — MVVM (Angular 18)

```
User Interaction
   │
   ▼
View              ← Angular templates, HTML/CSS
   ↕ two-way binding
ViewModel         ← Angular components + services
   │
   ▼
Model             ← TypeScript interfaces, API response DTOs
```

---

## Key Design Decisions

### Why a Flask orchestration layer?

Inserting Flask between Spring Boot and ManageOne serves 
two purposes. First, raw ManageOne API endpoints are never 
directly reachable from the frontend or the main backend — 
all cloud calls are proxied through Flask, reducing the 
attack surface. Second, it decouples the Spring Boot 
application from ManageOne's API contract, so changes 
to the cloud provider API only require updates to the 
Flask layer, not the entire backend.

### How RBAC is enforced

Three roles exist in the system: Professor, University Admin, 
and CCK Admin. Permissions are enforced server-side in the 
Spring Boot service layer on every request — not derived 
from frontend state. University admins authenticate via 
federated identity through the CCK IAM system. Professors 
authenticate via a password-based flow managed by the 
platform's own authentication service.

### How tenant isolation works

Each university operates within a quota allocated by CCK. 
Quota boundaries are enforced at two independent layers: 
the Spring Boot backend rejects requests that exceed the 
university's allocated quota before they reach the cloud layer, 
and ManageOne enforces project-level resource limits at the 
infrastructure level. Neither layer alone is trusted.

### Billing model

The platform implements a hybrid billing model inspired 
by Azure and IBM Cloud:

- **Fixed cost** — monthly subscription charge per university 
  based on reserved quota
- **Pay-as-you-go** — additional charges for on-demand 
  resource requests beyond the fixed reservation

Invoices are generated automatically on the first of each 
month by a scheduled billing job, stored in MySQL, and 
made available for PDF download and CSV/Excel export.

### Scheduling and unavailable slot detection

When a professor submits a reservation request for a 
specific time window, the system queries all existing 
approved reservations for the requested resource type 
and period. Conflicting slots are surfaced in the UI 
before the professor confirms. On approval by both 
university and CCK, a scheduled job registers the 
start and end datetime. At start time Flask triggers 
ECS creation via ManageOne. At end time Flask triggers 
deletion and releases quota back to the university pool.

---

## Data Model (Key Entities)

```
University
  └── has many → Professors
  └── has one  → Subscription (Plan + quota)
  └── has many → Invoices

Professor
  └── belongs to → University
  └── has many   → Reservations

Reservation
  └── belongs to → Professor
  └── belongs to → University
  └── references → CloudResource
  └── status: PENDING_UNIVERSITY | APPROVED_UNIVERSITY 
               | PENDING_CCK | APPROVED_CCK | REJECTED

Invoice
  └── belongs to → University
  └── contains   → BillingEntries (per reservation/resource)
  └── status: paid | unpaid | overdue

CloudResource
  └── type: ECS
  └── specs: vCPU · RAM · storage · image · pricePerHour
```

---

## Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | Angular 18 + TypeScript | SPA, component-based UI |
| Styling | Bootstrap + CSS | Responsive layout |
| Orchestration | Flask (Python) | ManageOne API proxy |
| Backend | Spring Boot (Java) | Business logic, RBAC, billing |
| ORM | Spring Data JPA | Database abstraction |
| Database | MySQL 8.0 | Relational data storage |
| Cloud Infra | Huawei ManageOne | ECS provisioning, IAM, monitoring |
| Design | Figma | UI prototyping |
| Diagrams | Draw.io | UML diagrams |
| Docs | LaTeX / Overleaf | Academic report |
| API Testing | Postman | Endpoint validation |
| Methodology | Scrum | Sprint-based delivery |
