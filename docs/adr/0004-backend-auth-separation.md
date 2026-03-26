# ADR 0004: Separation of Auth Service from Core Backend

## Status
Proposed

## Context
The Cardio Trace Platdorm has two main components for authentication and user management:
- **Auth Service**: a thin wrapper around Auth0
- **Core Backend**: Django DRF handling all domain data (patients, doctors, records, alarms)

Key considerations:
- Separation of concerns between authentication/identity and domain logic
- Scalability and maintainability
- Security and minimal duplication of sensitive data
- Clear boundaries for future extensions (multi-tenancy, RBAC)

## Decision
We will keep the **Auth Service separate from the Core Backend**:

- **Auth Service responsibilities**:
  - Redirecting users to Auth0 login
  - Handling OAuth2 callback 
  - Sending invitations via Auth0 Management API 
  - Assigning roles during invitation
- **Core Backend responsibilities**:
  - Mapping `auth0_user_id` to domain entities (patient/doctor)
  - Enforcing domain-specific permissions and business logic
  - Storing and managing all medical and relational data

The Auth Service does **not** store any domain data or handle `/me` endpoint logic. All domain mapping is done in the Core Backend.

## Consequences
- Clear separation of responsibilities improves maintainability
- Reduces risk of accidental exposure of sensitive domain data
- Simplifies testing and development of both services
- Allows independent scaling or replacement of Auth Service in the future
- Backend fully trusts JWTs issued by Auth0 via the Auth Service