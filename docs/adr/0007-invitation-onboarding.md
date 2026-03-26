# ADR 0007: Invitation-Based User Onboarding

## Status
Proposed

## Context
The Cardio Trace Platform does **not allow public signup**. Users must be invited by an administrator (clinic admin, yet to be decided) to ensure that only authorized doctors and patients can access the system.  

Requirements:
- Controlled onboarding for doctors and patients
- Assign appropriate roles during onboarding
- Integrate with Auth0 and Auth Service

## Decision
We will implement **invitation-based onboarding** using Auth0 Organizations and the Auth Service:

- Admin triggers an invitation via **Auth Service endpoint `/invite`**
- Auth Service calls Auth0 Management API to:
  - Send invitation email
  - Assign the correct role (`doctor` or `patient`)
  - Associate the user with the correct organization
- For portfolio/demo purposes, **invite links will use a "local" URL**:
  - Auth Service replaces the redirect URI in the Auth0 invite link with self local URL
  - Alternatively, the frontend can display the invite link for local testing
  - This allows testing the invitation flow without hosting a public domain
- User either clicks the link and completes account setup via Auth0 hosted login or is automatically redirected to Auth0 login page


## Consequences
- Ensures that only invited users can access the system
- Supports multi-tenant onboarding via Auth0 Organizations
- Allows portfolio/demo testing locally without needing a public domain or hosting