# ADR-TENANT-AUTHORITY-AND-RBAC-001

Status: ACCEPTED

Decision date: 2026-08-28

Scope: Transportation MVP Tenant authority, membership, RBAC assignment, and execution context

## Context

Transportation has a verified first-class Tenant foundation in `V43__tenant_foundation.sql`. Platform Tenancy owns Tenant lifecycle; Identity owns global credentials, explicit Tenant membership, and authenticated resolution. Existing access and refresh tokens do not contain a Tenant claim. Operational business rows are not fully Tenant-scoped, so isolation remains partial.

No recoverable legacy production database was found. The clean bootstrap Tenant is UUID `4f8b6a3b-2c1e-4d89-9a72-f9e4c5b3671a`, code `CLTS-LK`, Ceylon Logistics & Transport Solutions (Pvt) Ltd, `LKR`, `Asia/Colombo`, `ACTIVE`.

## Problem

Central governance previously required immutable `tenant_id` in JWT and described the foundation as absent. It also failed to distinguish global RBAC templates from Tenant-scoped assignment. That conflicted with verified runtime behavior and would make the operational retrofit depend on stale authority and legacy-backfill assumptions.

## Decision

### JWT and Tenant authority

The MVP uses `SERVER_SIDE_MEMBERSHIP_RESOLUTION`. Authentication establishes user identity. On login, refresh, and every bearer request, the server resolves the active user's current active `TenantMembership`, then its active Tenant.

Mandatory `tenant_id` in JWT is not required. Existing JWT and refresh-token contracts remain unchanged. A future Tenant claim may be an optimization or selection hint only when validated against current server membership; it may never be sole authority.

### Membership

The MVP permits one operational Tenant per user. `tenant_membership` enforces one membership record per user with `UNIQUE(user_id)`. Missing/inactive membership or Tenant denies access. Multi-membership and Tenant switching are deferred.

### RBAC templates and assignments

`app_permission`, `app_role`, and `app_role_permission` are global capability/template catalogues and do not grant Tenant ownership.

Authorization assignment is membership-scoped. Target:

`TenantMembership -> Membership Role Assignment -> Global Role Template -> Global Permissions`

Existing `app_user_role` is `LEGACY_UNSCOPED_ROLE_ASSIGNMENT` and transitional. It may preserve current RBAC compatibility temporarily but is not final Tenant-safe authority. A later forward migration must introduce a repository-convention membership-role table, define the compatibility window, migrate assignments, and deprecate/remove global user-role authority.

### Complete authorization rule

Access requires authenticated user, active Tenant membership, active Tenant, required RBAC permission, and matching Tenant-owned resource scope. None substitutes for another; privileged roles do not bypass isolation.

### Runtime context and client input

`CurrentTenant` / `TenantExecutionContext` is authoritative for business runtime. Business modules do not depend on Spring Security, JWT claims, or HTTP headers. Payload/query Tenant IDs, browser storage, and arbitrary `X-Tenant-ID` are untrusted. The single-membership MVP needs no switcher.

### Background workers

HTTP `TenantExecutionContext` is request-scoped and cleared after processing. Async, scheduled, and background work receives Tenant identity explicitly and establishes/clears a bounded context. Thread-local request state is never blindly inherited.

### Clean initialization and operational retrofit

Legacy preservation is not required and legacy backfill is not applicable. V43 is clean foundation initialization, not historical backfill. Operational aggregates must gain direct Tenant ownership via forward migrations; new writes derive ownership from `CurrentTenant`, not client input.

`TENANT-OPERATIONAL-DATA-RETROFIT-001` is next. US-29 stays blocked until Freight and Reporting source scoping and operational isolation acceptance pass.

## Security Rationale

Server resolution keeps mutable authorization state authoritative at the server. Membership reassignment/deactivation or Tenant deactivation takes effect on the next authenticated request without waiting for access-token expiry. A token proves signed identity and existing capability claims but never replaces current membership, Tenant status, or resource-scope checks.

## Alternatives Considered

### Mandatory immutable `tenant_id` JWT claim — rejected for the current MVP

It duplicates mutable authorization state into a token, adds revocation/refresh concerns, conflicts with verified per-request resolution, and risks treating a stale claim as sole authority. A validated hint remains possible later.

### Global `app_user_role` as final assignment — rejected

A global assignment cannot represent different capabilities for one user across different Tenants. Membership-scoped assignment supports future multi-membership without changing global templates.

### Client-selected Tenant without membership validation — rejected

Client-controlled Tenant identifiers are not authorization evidence. Future selection may choose only a current authorized membership.

## Consequences

- JWT and refresh-token public contracts remain compatible.
- Every authenticated request performs current membership and Tenant validation unless a future cache preserves immediate revocation semantics.
- Global role and permission catalogues remain reusable.
- `app_user_role` is controlled transitional debt.
- Business modules use one Tenant-context contract and remain decoupled from transport security.
- The foundation is implemented, but multi-tenancy is not complete until persistence, reads, events, jobs, caches, offline paths, and reports pass isolation acceptance.

## Migration Implications and Future Compatibility

The retrofit must use forward-only migrations, preserve V1–V43, add immutable `tenant_id` to Tenant-owned rows, scope constraints and repositories, and introduce membership-scoped role assignment without silently changing authentication behavior. Membership association permits a future user to be Dispatcher in Tenant A and Viewer in Tenant B while global role definitions remain unchanged.
