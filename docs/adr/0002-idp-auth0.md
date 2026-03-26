# ADR 0002: Use of External Identity Provider (Auth0)

## Status
Accepted

## Context
The Cadrio Trace Platform requires user authentication and management systemfor two main roles: doctors and patients. Security, reliability, and maintainability are critical, but the project is primarily a backend-focused portfolio piece.  

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

The internal Auth Service will act as a **thin wrapper** around Auth0 for:
- redirecting users to Auth0 login 
- handling OAuth2 callback 
- sending invites via Auth0 Management API 
- assigning roles during invitation

The Core Backend will rely on JWTs from Auth0 to identify users and enforce domain-specific permissions.

## Consequences
- Reduces implementation complexity and avoids handling sensitive user credentials directly
- Security is largely delegated to Auth0, including password policies and token signing
- System can scale to multiple organizations (clinics) easily
- Future flexibility is maintained: if needed, we could switch to another identity provider with minimal changes in Auth Service
- Auth Service remains thin and stateless, focusing only on orchestrating Auth0 flows
- Development focus can remain on backend domain logic and portfolio features rather than authentication internals