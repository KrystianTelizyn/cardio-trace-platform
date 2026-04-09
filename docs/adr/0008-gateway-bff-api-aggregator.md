# ADR 0008: Gateway as BFF and API Aggregator

## Status
Pending

## Context
The Cardio Trace platform is composed of multiple backend capabilities:

- **Core Backend** (Django DRF) exposing REST APIs
- **GraphQL layer** (Hasura)
- **Auth0** (external IdP), integrated at the **Cardio Trace Gateway**—not as a separate browser-facing “Auth Service” (see [ADR 0002](0002-idp-auth0.md), [ADR 0004](0004-backend-auth-separation.md))
- **Additional services** (e.g. workers, AI), reached from the product as needed—typically not directly from the browser

The frontend is a **React SPA** that needs:

- A **single** API surface for normal application traffic
- **Authentication** handled at a dedicated edge (see [ADR 0003](0003-auth-token-handling.md))
- **Simpler** access to both REST and GraphQL without juggling many origins and auth rules

Without a gateway, the SPA would tend to:

- Call multiple services directly (broader CORS exposure, duplicated auth handling)
- Coordinate **REST** and **GraphQL** and protocol differences itself
- Push orchestration concerns into the client

## Decision
We introduce **`cardio-trace-gateway`** as:

- **Backend-for-Frontend (BFF)** for the SPA: the only first-class integration point for authenticated app traffic, auth flows, and invite orchestration ([ADR 0007](0007-invitation-onboarding.md)).
- **API gateway / aggregator**: routes and proxies to internal HTTP APIs while enforcing auth **before** forwarding.

The gateway **does**:

- **Auth0-facing** flows: login redirect, callback, refresh/logout as designed (details in ADR 0003)
- **Session handling at the edge**: browser cookie carrying an **encrypted session** (access, refresh, id tokens) per ADR 0003—use **`HttpOnly`**, **`Secure`**, and **`SameSite`** appropriately in implementation; not a second copy of domain logic
- **Expose a unified surface** (exact paths are an implementation choice), for example:
  - `/graphql` → proxy to Hasura
  - `/api/*` or `/rest/*` → proxy to Core Backend (DRF)
- **Decrypt/validate** session and tokens before proxying; enforce role-based authorization at the edge and forward only minimal trusted request context to upstream services (the SPA does not send Bearer; see ADR 0003)

The gateway **does not**:

- Own **domain business rules** (those stay in Core Backend, Hasura metadata/RBAC, and other services)
- **Replace** DRF, Hasura, or workers—it **orchestrates and proxies**
- Rely on a **server-side session store** for auth as the default model: scaling is **horizontal** and instances stay interchangeable (session *material* is carried in the encrypted cookie; optional refresh/token logic remains gateway-owned, not “fat” domain state)

## Consequences
- The SPA integrates primarily with **one origin** for API and auth helpers, reducing client complexity
- **Authentication** and edge validation are **centralized** (see ADR 0003 for CSRF and cookie considerations)
- End-user access tokens are kept at the edge and are not propagated to internal APIs
- Core Backend and Hasura stay **focused** on domain data, permissions, and schema
- **REST and GraphQL** both sit behind one gateway-shaped interface
- The gateway is **operationally critical**: availability and safe rollout matter; multiple instances are expected
- Further ADRs (tenancy, roles) apply **behind** the gateway; the BFF does not redefine those rules—it forwards authenticated calls

## References
- [ADR 0002: Auth0 as IdP](0002-idp-auth0.md)
- [ADR 0003: Auth token handling](0003-auth-token-handling.md)
- [ADR 0004: Gateway auth vs Core Backend](0004-backend-auth-separation.md)
- [ADR 0007: Invitation onboarding](0007-invitation-onboarding.md)
- [ADR 0009: Hasura placement with the gateway](0009-graphql-hasura-gateway-placement.md)
