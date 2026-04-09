# ADR 0002: Use of External Identity Provider (Auth0)

## Status
Pending

## Context
The **Cardio Trace** platform requires user authentication and a management story for two main roles: doctors and patients. Security, reliability, and maintainability are critical, but the project is primarily a backend-focused portfolio piece.

Implementing a custom authentication system would require:

- password storage and hashing
- secure signup/login flows
- invitation management
- role assignment logic
- ongoing maintenance and security updates

These requirements introduce significant complexity and potential security risks.

## Decision
We will use **Auth0** as the external identity provider. Auth0 provides:

- Hosted login pages and OAuth2/OpenID Connect flows
- Multi-tenant support via Organizations
- Role and permission management
- Invitation-based onboarding
- Token issuance with JWTs containing custom claims

The **Cardio Trace Gateway** (FastAPI) acts as the **thin orchestration layer** at the browser boundary for Auth0: redirecting users to Auth0 login, handling the OAuth2 callback, sending invites via the Auth0 Management API, and assigning roles during invitation. Session and token handling at the edge are specified in [ADR 0003](0003-auth-token-handling.md); separation from domain logic is in [ADR 0004](0004-backend-auth-separation.md).

The **Core Backend** receives gateway-mediated request context for authenticated browser flows and enforces domain-specific permissions. It does not replace the gateway for login, callback, or invite orchestration.

## Consequences
- Reduces implementation complexity and avoids handling sensitive user credentials directly
- Security is largely delegated to Auth0, including password policies and token signing
- The system can scale to multiple organizations (clinics) easily
- Future flexibility is maintained: switching identity providers would mainly affect **gateway** Auth0 integration, not the Core Backend’s domain model
- The gateway stays focused on orchestrating Auth0 flows and proxying authenticated traffic; the Core Backend stays focused on domain data and rules
- Development effort can remain on domain logic and portfolio features rather than building authentication from scratch
