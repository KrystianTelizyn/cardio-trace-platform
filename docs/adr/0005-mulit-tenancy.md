# ADR 0005: Multi-Tenancy Strategy (Auth0 Organizations + org_id in JWT)

## Status
Proposed

## Context
The Cardio Trace Platform must support multiple clinics (organizations) with isolated user bases. Each user belongs to exactly one organization.  

Requirements:
- Isolate data between organizations
- Enforce access control per organization
- Minimal complexity for portfolio/demo project
- Integration with Auth0 for authentication and role management

## Decision
We will implement **multi-tenancy using Auth0 Organizations** and embed `org_id` in JWTs:

- Each clinic maps to one **Auth0 Organization**
- Users are members of exactly one organization
- JWT issued by Auth0 includes:
  - `sub` (user ID)
  - `org_id` (organization ID)
  - Custom claim `https://cardio-trace.com/roles` (roles in that org)
- Backend uses `org_id` from the token to enforce organization-level access control
- Auth Service does not store tenant info; all tenant data is in JWT

## Consequences
- Clear isolation of user data per organization
- Minimal custom multi-tenancy logic in backend
- Backend enforces access based on `org_id` and role claims
- Adding new organizations is straightforward via Auth0
- Simplifies portfolio/demo implementation while supporting real multi-tenant design