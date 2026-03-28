# ADR 0004: Separation of Gateway Auth from Core Backend

## Status
Proposed

## Context
The Cardio Trace **platform** separates **identity and auth orchestration** from **domain data and rules**:

- **Cardio Trace Gateway** (FastAPI) is the **only** browser-facing API for authenticated app traffic: login URLs, OAuth callback handling, invite URL generation, and proxies to the inner REST API and Hasura (see [ADR 0003](0003-auth-token-handling.md)). The gateway integrates with **Auth0** for those flows.
- **Core Backend** (Django DRF): domain data (patients, doctors, records, alarms) and business logic.

The earlier label **Auth Service** (“thin wrapper around Auth0”) describes **concerns** that the gateway **implements at the edge**—not a second public surface for the SPA. A separate internal auth microservice remains optional if you later split gateway code for scaling; the architectural rule below stays the same.

Key considerations:

- Separation of concerns between authentication/identity and domain logic
- Scalability and maintainability
- Security and minimal duplication of sensitive data
- Clear boundaries for future extensions (multi-tenancy, RBAC)

## Decision
We keep **auth and identity orchestration** separate from the **Core Backend**:

- **Gateway (auth-facing) responsibilities**:
  - Redirecting users to Auth0 login and handling the OAuth2 callback
  - Invite flows via Auth0 Management API (or equivalent) and role assignment during invitation
  - Session/token handling at the edge per ADR 0003 before proxying
- **Core Backend responsibilities**:
  - Mapping `auth0_user_id` to domain entities (patient/doctor)
  - Enforcing domain-specific permissions and business logic
  - Storing and managing all medical and relational data

The Core Backend does **not** own Auth0 redirect/callback or invite issuance. It does **not** replace the gateway as the SPA’s auth entry point. Domain-only endpoints (e.g. `/me` mapping) live in the Core Backend; they are reached **through** the gateway on proxied REST traffic.

## Consequences
- Clear separation of responsibilities improves maintainability
- Reduces risk of accidental exposure of sensitive domain data
- Simplifies testing and development of gateway vs Core Backend
- Allows independent scaling or refactoring of gateway auth code without entangling domain models
- Core Backend (and Hasura, where applicable) continue to rely on **Auth0-issued JWTs** carried as **`Authorization: Bearer`** on requests the gateway forwards—typically validating issuer, audience, and signature as today, without parsing the gateway’s encrypted session cookie
