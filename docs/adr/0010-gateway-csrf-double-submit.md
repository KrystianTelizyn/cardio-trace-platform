# ADR 0010: Gateway CSRF Protection with Double-Submit Token

## Status
Proposed

## Context
The Cardio Trace Gateway uses a cookie-backed browser session at the edge for authenticated traffic (see ADR 0003 and ADR 0008). Cookie-backed auth requires explicit CSRF mitigation for browser-initiated unsafe requests.

ADR 0003 documents CSRF as a required concern and lists candidate mitigations, but the gateway implementation now applies one concrete pattern consistently across protected routes.

We need a clear, stable decision so frontend, gateway, and tests all follow the same contract.

## Decision
We standardize on a **double-submit CSRF token** pattern for browser requests handled by the gateway:

- On successful login callback, the gateway sets a gateway-owned CSRF cookie (`gateway_csrf`) with a short TTL.
- The SPA reads that cookie and sends the same value in the `X-CSRF-Token` header for protected requests.
- The gateway validates that cookie token and header token are both present and equal.
- CSRF checks are enforced on:
  - REST proxy: `POST`, `PUT`, `PATCH`, `DELETE`
  - GraphQL proxy: `POST`
- Requests that fail validation return `403`.

Cookie attributes for `gateway_csrf`:

- `HttpOnly=false` (so the SPA can read and submit the token)
- `SameSite=lax`
- `Secure` derived from configured frontend URL scheme
- `Path=/`

This CSRF token is **separate** from the encrypted Auth0 session cookie used for authentication.

## Consequences
- CSRF behavior is now explicit and testable as a cross-service contract between SPA and gateway.
- Frontend clients must include `X-CSRF-Token` for protected calls, or requests will be rejected.
- The model keeps JWTs unavailable to browser JavaScript while still protecting cookie-backed authenticated requests from cross-site submission.
- Future changes to CSRF method coverage, cookie attributes, or token transport should update this ADR.

## References
- [ADR 0003: Authentication Token Handling Strategy](0003-auth-token-handling.md)
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
