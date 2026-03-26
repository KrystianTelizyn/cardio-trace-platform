# ADR 0006: Role & Permission Model

## Status
Proposed

## Context
The Cardio Trace Platform has two primary user roles: **doctor** and **patient**. Proper enforcement of permissions is critical for protecting sensitive medical data and ensuring that users can only access resources they are authorized for.  

Requirements:
- Support role-based access control (RBAC)
- Integrate seamlessly with Auth0
- Avoid duplicating roles/permissions in the Core Backend for simplicity
- Minimal complexity for a portfolio/demo project

## Decision
We will use **roles embedded in JWTs issued by Auth0**:

- Roles are included in a custom claim: `https://cadrio.trace.com/roles`
- Auth Service assigns roles during invitation via Auth0 Management API
- Core Backend interprets roles from the token and enforces permissions per request
- No separate RBAC store is maintained in the backend
- Any role-based logic is evaluated on-the-fly from the JWT

## Consequences
- Simplifies backend architecture (no separate role tables or sync needed)
- Ensures that the source of truth for roles is Auth0
- Adding or changing roles requires updating Auth0, not backend database
- Backend enforces domain-specific permissions securely based on the role claims in JWT
- Consistent and predictable behavior for multi-tenant, role-based access control