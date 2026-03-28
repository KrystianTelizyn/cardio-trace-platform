# ADR 0007: Invitation-Based User Onboarding

## Status
Proposed

## Context
The Cardio Trace Platform does **not allow public signup**. Users must be invited by an administrator (clinic admin, yet to be decided) so that only authorized doctors and patients can access the system.

Requirements:

- Controlled onboarding for doctors and patients
- Assign appropriate roles during onboarding
- Integrate with Auth0 at the **Cardio Trace Gateway** (see [ADR 0003](0003-auth-token-handling.md), [ADR 0004](0004-backend-auth-separation.md))

## Decision
We will implement **invitation-based onboarding** using **Auth0 Organizations** and the **gateway**:

- An admin triggers an invitation via a **gateway endpoint** (e.g. `/invite` or the path chosen in the gateway API design)—not by calling the Core Backend or Auth0 directly from the browser except through the gateway.
- The gateway calls the **Auth0 Management API** to:
  - Send the invitation email
  - Assign the correct role (`doctor` or `patient`)
  - Associate the user with the correct organization
- For portfolio/demo purposes, **invite links can use a local redirect URL**:
  - The gateway may rewrite or configure the redirect URI in the Auth0 invite flow so it targets a **local** app URL
  - Alternatively, the frontend can surface the invite link for local testing
  - This supports exercising the invitation flow without a public production domain
- The user follows the link and completes account setup via **Auth0 hosted login**, or is redirected to the Auth0 login page as configured

## Consequences
- Ensures that only invited users can access the system
- Supports multi-tenant onboarding via Auth0 Organizations
- Allows portfolio/demo testing locally without requiring a public domain
- Aligns with the gateway as the sole public orchestration point for invites (consistent with token and proxy handling in ADR 0003)
