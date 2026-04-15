# ADR 0013: User Provisioning on First Login

## Status
Proposed

## Context
[ADR 0007](0007-invitation-onboarding.md) defines the invitation flow: an admin triggers an invite via the gateway, the gateway calls Auth0 Management API to send the invite with a role and organization, and the user completes signup via Auth0 hosted login.

[ADR 0004](0004-backend-auth-separation.md) states that the Core Backend is responsible for "mapping `auth0_user_id` to domain entities (patient/doctor)." However, no ADR specifies **when** or **how** the local User record and domain profile (PatientProfile or DoctorProfile) are created in the backend database.

Without this step, a newly invited user completes Auth0 signup successfully but has no corresponding record in the Core Backend — Hasura queries return nothing, and domain operations fail.

## Decision
The **gateway** provisions the backend user during the **first-login-from-invite callback**.

### Flow

1. User accepts the invite and completes Auth0 signup
2. Auth0 redirects to the gateway callback endpoint
3. Gateway detects this is a **first-time login from invite** (as opposed to a regular returning login)
4. Gateway calls **`POST /users`** on the Core Backend with identity data extracted from the Auth0 token and userinfo:
   - `auth0_user_id` — `sub` claim
   - `auth0_org_id` — `org_id` claim
   - `role` — from `https://cardio-trace.com/roles` claim
   - `email` — from Auth0 userinfo
   - `name` — from Auth0 userinfo (optional)
5. Core Backend resolves `auth0_org_id` → local Tenant, creates User, and creates a PatientProfile or DoctorProfile seeded with `name`
6. Gateway completes session setup (sets encrypted session cookie) and redirects to the SPA

### Idempotency

`POST /users` is idempotent for the same user membership tuple (`auth0_user_id`, `auth0_org_id`, `role`): if that membership already exists, the endpoint returns the existing record (200) instead of creating a duplicate (201). This handles callback retries and edge cases safely.

### Prerequisites

- The **Tenant** record (with `auth0_organization_id`) must already exist in the backend database before any user in that organization is invited. Tenants are seeded as part of platform setup, not created by the invite flow.

### Profile completion

The provisioning step creates a **sparse profile** — only `name` is populated from Auth0. Domain-specific fields (`specialization`, `license_number`, `dob`, `medical_id`, `gender`) start with model defaults (empty strings or `null`, depending on field type) and are filled in later through profile update flows.

## Consequences
- The gateway remains the orchestration point for Auth0 flows, consistent with [ADR 0002](0002-idp-auth0.md) and [ADR 0004](0004-backend-auth-separation.md)
- The Core Backend does not need to call Auth0 or detect first-login independently — the gateway drives provisioning
- Every invited user has a backend record before they first see the SPA, so Hasura queries and domain operations work immediately
- The provisioning call is synchronous — the gateway waits for the backend response before setting the session cookie, so the user is never in a half-provisioned state
- Tenant seeding is an operational prerequisite; inviting into a non-existent organization fails explicitly (404)
- Future changes to provisioning data (e.g. adding fields from Auth0 metadata) only affect the gateway → backend call contract, not the Auth0 integration

## References
- [ADR 0004: Separation of Gateway Auth from Core Backend](0004-backend-auth-separation.md)
- [ADR 0007: Invitation-Based User Onboarding](0007-invitation-onboarding.md)
- [ADR 0012: Gateway-to-Backend Internal Trust Contract](0012-gateway-backend-trust-contract.md)
