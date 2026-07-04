# 2. Adopt Supabase and Deno for a multi-tenant SaaS

Date: 2026-07-04

## Status

Accepted

## Context

Roostara is a new "adulting" SaaS product that will serve many independent customers
(tenants) from shared infrastructure. As a greenfield project with a small team, we need a
platform that gives us — without building it ourselves — authentication, a relational data
store, file storage, realtime updates, and a place to run server-side logic, while keeping
each tenant's data strictly isolated.

The central risk in a shared-database multi-tenant SaaS is cross-tenant data leakage. We
want that isolation guaranteed at a layer that cannot be forgotten in application code, and
we want a small, auditable trust boundary rather than a sprawl of bespoke services.

This decision records the foundational technology stack that the project constitution
(v1.1.0) already assumes.

## Decision

We will build Roostara as a **multi-tenant SaaS on Supabase**, with **Deno Edge Functions**
for server-side logic.

- **Supabase** is the platform of record: managed Postgres (source of truth), Supabase Auth
  for identity, plus Storage and Realtime as needed. We prefer these native primitives over
  external services (Supabase-Native First).
- **Tenant isolation is enforced by Postgres Row-Level Security (RLS)** on every
  tenant/user-scoped table. Policies key off Supabase Auth claims (`auth.uid()` and
  tenant/role claims). No request path serves end users via the service-role key, which is
  never exposed to clients.
- **Deno** runs all server-side logic as Edge Functions in TypeScript (`strict` mode), with
  explicitly pinned dependencies (import maps / pinned URLs) and least-privilege
  permissions.
- **Schema changes flow through versioned Supabase CLI migrations**; no out-of-band changes
  to hosted schemas.

## Consequences

**Easier**

- Tenant isolation becomes a uniform, database-enforced invariant instead of a convention
  each developer must remember; it is directly testable per table.
- Identity, data, and authorization live inside one trust boundary that RLS can reason
  about, shrinking the attack surface.
- Less undifferentiated infrastructure to build and operate (auth, storage, realtime come
  from the platform).

**More difficult / accepted trade-offs**

- We take on a strong dependency on Supabase; replacing a core primitive later is costly and
  would itself warrant a superseding ADR.
- Correct RLS is now a first-class engineering discipline: every user-scoped table needs
  policies and isolation tests, and reviewers must scrutinize them.
- The Deno edge model constrains available libraries and runtime APIs compared to a
  general-purpose Node/container backend; dependencies must be edge-compatible and pinned.
- Local development and CI must run a local Supabase stack so migrations and RLS can be
  tested before deploy.

**Follow-on decisions to capture as future ADRs**

- The concrete tenancy data model (e.g. `tenant_id` column strategy vs. Postgres schemas).
- Auth/session model details and how tenant/role claims are populated into JWTs.
