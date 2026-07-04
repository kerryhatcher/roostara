<!--
SYNC IMPACT REPORT
==================
Version change: 1.0.0 → 1.1.0
Bump rationale: MINOR — added a new governance obligation (Architecture Decision Records
                recorded via the `adrs` CLI). No principles removed or redefined.

History:
  - 1.0.0 (2026-07-04) Initial ratification: five principles + platform/workflow sections
                       for a multi-tenant SaaS on Supabase + Deno.
  - 1.1.0 (2026-07-04) Added "Architecture Decision Records" subsection to Governance.

Principles (unchanged in 1.1.0):
  - I. Tenant Isolation via Row-Level Security (NON-NEGOTIABLE)
  - II. Supabase-Native First
  - III. Deno Edge Functions & Typed Contracts
  - IV. Test-First (NON-NEGOTIABLE)
  - V. Observability & Secure Secrets

Sections:
  - Technology & Platform Constraints
  - Development Workflow & Quality Gates
  - Governance (+ Architecture Decision Records subsection, new in 1.1.0)

Templates requiring updates:
  - .specify/templates/plan-template.md   ✅ aligned (Constitution Check gate is generic)
  - .specify/templates/spec-template.md   ✅ aligned (no principle-specific mandatory sections)
  - .specify/templates/tasks-template.md  ✅ aligned (foundational/security/observability
                                              task categories already cover principle-driven work)
  - .specify/templates/checklist-template.md  ✅ aligned (no constitution references)
  - Command files (.specify/templates/commands/*.md)  ✅ N/A (directory does not exist)
  - README.md / docs/quickstart.md        ✅ N/A (no runtime guidance docs yet)

Deferred TODOs:
  - none (multi-tenant interpretation confirmed by maintainer 2026-07-04).
-->

# Roostara Constitution

Roostara is a multi-tenant SaaS "adulting" app built on Supabase and served by Deno
Edge Functions. These principles are the shared tenets every contributor and agent MUST
uphold; they supersede convenience, habit, and individual preference.

## Core Principles

### I. Tenant Isolation via Row-Level Security (NON-NEGOTIABLE)

Every table holding tenant- or user-scoped data MUST have Postgres Row-Level Security
(RLS) enabled with explicit policies; no table ships with RLS disabled "temporarily."
Access is decided in the database, not in application code — no query path may bypass
RLS by using the service-role key to serve end-user requests. The service-role key MUST
never be exposed to clients or embedded in browser/mobile bundles; it is used only in
trusted server contexts (migrations, admin jobs, back-office functions). Every new table
and policy MUST have a test proving that tenant A cannot read or write tenant B's rows.

**Rationale**: In a shared-database multi-tenant SaaS, isolation is the product's core
safety guarantee. Enforcing it at the database layer makes it uniform and unbypassable,
rather than depending on every developer remembering to add a `WHERE tenant_id = …`.

### II. Supabase-Native First

Prefer Supabase platform primitives — Postgres, Auth, Storage, Realtime, and Edge
Functions — over bespoke infrastructure. The Postgres database is the source of truth.
All schema changes MUST go through versioned migrations managed by the Supabase CLI
(no manual, out-of-band changes to hosted schemas). Auth MUST use Supabase Auth
(`auth.uid()` / JWT claims) as the identity source that RLS policies key off of.
Introducing an external service to do something a Supabase primitive already does
requires a justification recorded in the plan's Complexity Tracking.

**Rationale**: Leaning on the platform keeps the surface small, keeps identity and data
in one trust boundary that RLS can reason about, and avoids drift between environments.

### III. Deno Edge Functions & Typed Contracts

Server-side logic runs as Deno Edge Functions written in TypeScript with `strict` mode
enabled. Dependencies MUST be declared explicitly via import maps or pinned URLs — no
implicit or unpinned imports. Functions run with least privilege: request only the Deno
permissions actually needed. Every function exposes a typed request/response contract,
validates all inbound input at the boundary, and returns structured errors (never leaks
stack traces or secrets to clients).

**Rationale**: Deno's explicit-permissions, URL-based module model is a security feature;
using it deliberately (pinned deps, minimal permissions, validated inputs) keeps the edge
tier auditable and safe.

### IV. Test-First (NON-NEGOTIABLE)

Tests are written and MUST fail before implementation begins (Red-Green-Refactor).
Required coverage: (a) RLS policy tests for tenant isolation on every user-scoped table
(see Principle I), (b) contract tests for every Edge Function's request/response shape and
error paths, and (c) migration tests that apply against a local Supabase stack before any
deploy. A change that alters data access or a function contract MUST update or add tests
in the same change.

**Rationale**: Isolation and contract correctness are exactly the failures that are
invisible in a demo and catastrophic in production; only executable tests prove them.

### V. Observability & Secure Secrets

Edge Functions and database routines MUST emit structured logs (JSON) carrying enough
context to trace a request — without logging secrets, tokens, or another tenant's data.
Authentication and authorization-relevant events MUST be auditable. Secrets and
configuration live in Supabase secrets / environment variables, never in source, client
bundles, or committed files; `.env` and key material stay git-ignored. Every principle
here is verifiable at review time.

**Rationale**: A multi-tenant system must be debuggable across tenants without becoming a
data-leak or secret-leak vector; structured, secret-free logs make that possible.

## Technology & Platform Constraints

- **Platform**: Supabase (managed Postgres 15+, Auth, Storage, Realtime, Edge Functions).
- **Runtime**: Deno (TypeScript, `strict: true`) for all Edge Functions and server logic.
- **Database access**: Row-Level Security enabled on all tenant/user data; policies key
  off Supabase Auth claims (`auth.uid()`, tenant/role claims).
- **Schema management**: Supabase CLI migrations only; migrations are reviewed and tested
  against a local stack before deploy.
- **Tooling**: `deno fmt` and `deno lint` are the formatting/linting authorities; `deno
  test` is the test runner.
- **Secrets**: Managed via Supabase secrets / env; service-role key restricted to trusted
  server contexts and never client-exposed.

## Development Workflow & Quality Gates

- **Branches & commits**: Work on branches; use Conventional Commits; keep changesets
  small and commit after each task/story.
- **Review**: Every PR MUST verify compliance with the Core Principles. Reviews pay
  special attention to new tables/policies (Principle I) and function contracts
  (Principle III).
- **Local validation gate**: Before merge, changes MUST pass `deno fmt --check`,
  `deno lint`, and `deno test`, and migrations MUST apply cleanly to a local Supabase
  stack.
- **Complexity gate**: Any deviation from Supabase-native primitives or from these
  principles MUST be recorded in the plan's Complexity Tracking with the simpler
  alternative and why it was rejected.

## Governance

This constitution supersedes all other practices and conventions. When guidance conflicts,
the constitution wins.

- **Amendments** MUST be proposed via PR, describe the change and its rationale, and
  update dependent templates (`plan`, `spec`, `tasks`, `checklist`) in the same change.
- **Versioning** follows semantic versioning of governance:
  - **MAJOR** — backward-incompatible removal or redefinition of a principle.
  - **MINOR** — a new principle/section or materially expanded guidance.
  - **PATCH** — clarifications and wording that do not change obligations.
- **Compliance review**: PR reviewers MUST confirm the change upholds every principle;
  unjustified complexity MUST be rejected or moved into Complexity Tracking. Runtime and
  agent guidance (e.g. CLAUDE.md, future `docs/quickstart.md`) MUST stay consistent with
  this document.

### Architecture Decision Records (ADRs)

Significant architecture and technology decisions MUST be recorded as Architecture
Decision Records in this repository using the **`adrs` CLI tool**.

- **Tooling**: Manage ADRs with `adrs` — initialize once with `adrs init` (creates the
  ADR directory, default `doc/adr/`) and add each decision with
  `adrs new "<decision title>"`. ADRs are numbered, immutable markdown files committed to
  the repo; a decision that changes an earlier one MUST supersede it (e.g.
  `adrs new -s <N> "<title>"`) rather than editing the accepted record.
- **When required**: Record an ADR for any durable, hard-to-reverse choice — data model
  or tenancy/RLS strategy, adopting or replacing a Supabase primitive or external service,
  Edge Function boundaries and contracts, auth model, or any deviation justified in a
  plan's Complexity Tracking. When in doubt, write the ADR.
- **Relationship to this constitution**: ADRs record *decisions made under* these
  principles; they do not amend the constitution. Changing a principle still requires a
  versioned amendment to this document (see Amendments above).

**Version**: 1.1.0 | **Ratified**: 2026-07-04 | **Last Amended**: 2026-07-04
