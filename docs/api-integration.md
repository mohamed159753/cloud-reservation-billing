# Huawei ManageOne API Integration

> This document describes the infrastructure integration layer 
> built during the CCK PFE internship. Source code is proprietary 
> to CCK — this is architectural documentation only.

## Overview

The Flask orchestration layer sits between the Spring Boot backend 
and Huawei ManageOne's cloud management APIs. Its role is to 
translate platform-level resource requests into ManageOne API calls, 
shielding the rest of the application from direct cloud API exposure.

## Authentication — IAM Token Flow

```
Spring Boot Backend
      │
      │  Reserve ECS request
      ▼
Flask Orchestration API
      │
      │  POST /v3/auth/tokens
      │  { username, password, project_scope }
      ▼
Huawei ManageOne IAM
      │
      │  X-Subject-Token (scoped)
      ▼
Flask (token stored per session, 
       passed in X-Auth-Token header
       on all subsequent API calls)
```

## ECS Lifecycle Management

| Action | ManageOne API Call |
|--------|-------------------|
| Create ECS | POST /v1/{project_id}/cloudservers |
| Start ECS | POST /v1/{project_id}/cloudservers/action (os-start) |
| Stop ECS | POST /v1/{project_id}/cloudservers/action (os-stop) |
| Delete ECS | DELETE /v1/{project_id}/cloudservers/{server_id} |

Parameters passed at creation: flavor ID (vCPU/RAM spec), 
image ID (OS), network config, reservation metadata.

## Real-time Metrics Polling

The professor metrics dashboard pulls live data from ManageOne's 
monitoring endpoints:

- CPU usage percentage
- Memory used / available
- Network in/out (KB/s)

Data is fetched on dashboard load and on manual refresh. 
No caching — always reflects current ECS state.

## Scheduling Engine

When a professor submits a reservation request:

1. System queries existing approved reservations for the 
   requested resource type and time window
2. Unavailable slots are surfaced in the UI before confirmation
3. On approval, a scheduled job is registered with the start 
   and end datetime
4. At start time: Flask triggers ECS creation via ManageOne API
5. At end time: Flask triggers ECS deletion, quota is released

## Quota Verification

Before any reservation is confirmed at CCK level:

1. Flask calls ManageOne to verify available quota 
   (vCPU, RAM, storage) against university allocation
2. If quota is insufficient, request is flagged for 
   additional resource approval workflow
3. University admin approves → CCK admin approves → 
   ECS creation is triggered
