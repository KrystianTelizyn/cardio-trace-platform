# ADR 0006: Role & Permission Model

## Status
Pending

## Context
The Cardio Trace Platform has two primary user roles: **doctor** and **patient**. Proper enforcement of permissions is critical for protecting sensitive medical data and ensuring that users can only access resources they are authorized for.  

Requirements:
- Support role-based access control (RBAC)
- Integrate seamlessly with Auth0
- Avoid duplicating roles/permissions in the Core Backend for simplicity
- Minimal complexity for a portfolio/demo project

## Decision
We will use **roles embedded in JWTs issued by Auth0** as the source of truth, with edge enforcement at the gateway:

- Roles are included in a custom claim: `https://cardio-trace.com/roles`
- Gateway assigns roles during invitation flow via Auth0 Management API integration
- Gateway reads role claims from the decrypted session token and enforces route-level authorization before proxying
- No separate RBAC store is maintained in the backend
- Core Backend may apply additional domain authorization rules, but does not depend on direct end-user bearer token propagation

## Consequences
- Simplifies backend architecture (no separate role tables or sync needed)
- Ensures that the source of truth for roles is Auth0
- Adding or changing roles requires updating Auth0, not backend database
- Gateway centralizes role-checking for incoming browser traffic; downstream services use internal trust context plus domain rules
- Consistent and predictable behavior for multi-tenant, role-based access control