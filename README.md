# Cloud Resource Reservation & Billing System — CCK PFE

> Full-stack web platform built during a 4-month final year internship at 
> El Khawarizmi Computing Center (CCK), enabling professors to reserve 
> Huawei ECS cloud resources, universities to manage allocations, and CCK 
> to oversee billing across academic institutions.

> Source code is proprietary to CCK. This repository contains the academic 
> report, architecture documentation, screenshots, and demo video.

---

## Overview

| Field        | Value                                              |
|--------------|----------------------------------------------------|
| Context      | Bachelor's PFE — ISAMM / CCK                      |
| Period       | February – June 2025                               |
| Team         | Mohamed Mensi & Zaineb Arfaoui                     |
| Supervisor   | Dr. Amine Kechiche (ISAMM) · Mr. Badis Saadi (CCK)|
| Stack        | Angular 18 · Spring Boot · Flask · MySQL           |
| Infra        | Huawei ManageOne · ECS · IAM API                  |
| Methodology  | Scrum — 4 sprints · 2 releases                    |

---

## Deliverables

- 📄 [Academic Report (PDF)](report/PFE_Mohamed_Mensi_Zaineb_Arfaoui_2025.pdf)
- 🎥 [Cloud Resource Reservation & ECS Lifecycle Management Platform — Full Workflow Demonstration](demo/README.md)
- 🖼️ Screenshots — see /screenshots
- 📐 Architecture — see /docs/architecture.md
- 🔌 API Integration Detail — see /docs/api-integration.md

---

## What Was Built

Three-role platform connecting professors, university admins, 
and CCK administrators for end-to-end cloud resource management:

**Professor**
- Reserve ECS instances via form or AI-assisted natural language input
- Schedule reservations with unavailable slot detection
- Start/stop/access reserved ECS instances
- View live metrics (CPU, RAM, network) per instance
- Consult reservation history and status

**University Admin**
- Manage professor accounts (add, edit, remove)
- Approve or reject reservation requests
- Monitor quota consumption with usage alerts
- Consult monthly billing history and invoice details

**CCK Admin**
- Oversee all university reservations and resource allocation
- Generate automated monthly invoices (fixed + pay-as-you-go)
- Export billing data to CSV/Excel
- Cross-university usage statistics and trend dashboards

---

## Infrastructure Integration

The Flask orchestration layer integrates directly with 
Huawei ManageOne's cloud APIs:

- **IAM Authentication** — token-based auth scoped to CCK 
  project space, obtained per session and passed through to 
  all resource management calls
- **ECS Lifecycle** — create, start, stop, and delete ECS 
  instances via ManageOne compute API using flavor/image parameters
- **Real-time Metrics** — polls ManageOne monitoring endpoints 
  for live CPU, RAM, and network data per instance
- **Scheduling Engine** — detects existing approved reservations 
  and surfaces unavailable time slots before confirming new requests; 
  scheduled jobs trigger ECS creation at start time and teardown at expiry

See [/docs/api-integration.md](docs/api-integration.md) for full detail.

---

## Architecture

```
┌─────────────────────────────────┐
│         Angular 18 Frontend      │
│  Professor · University · CCK    │
└──────────────┬──────────────────┘
               │ HTTP/REST
┌──────────────▼──────────────────┐
│      Flask Orchestration API     │  ← decouples frontend from ManageOne
└──────────────┬──────────────────┘
               │ HTTP
┌──────────────▼──────────────────┐
│      Spring Boot Backend         │  ← RBAC · billing · scheduling
│   Controller · Service · JPA    │
└────────┬─────────────┬──────────┘
         │ JDBC        │ HTTP
┌────────▼──────┐ ┌────▼────────────────────┐
│  MySQL DB     │ │  Huawei ManageOne APIs   │
│               │ │  IAM · ECS · Monitoring  │
└───────────────┘ └─────────────────────────┘
```

---

## Scrum Delivery

| Release | Sprints | Scope |
|---------|---------|-------|
| Release 1 | Sprint 1 + 2 | Authentication · Profile Management · Resource Reservation |
| Release 2 | Sprint 3 + 4 | Usage Tracking · Statistics · Automated Billing |

---

## Note on Source Code

Source code is the intellectual property of CCK 
(El Khawarizmi Computing Center). Full technical 
implementation is documented in the academic report.
