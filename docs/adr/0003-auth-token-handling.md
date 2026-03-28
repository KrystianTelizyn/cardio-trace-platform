# ADR 0003: Authentication Token Handling Strategy

## Status

Pending

## Context

The **Cardio Trace** frontend is a React SPA. It is **only exposed to the Cardio Trace Gateway** (`cardio-trace-gateway`), implemented with **FastAPI**. The gateway is the single integration point for:

- Obtaining **Auth0 (or equivalent) login URLs** for the browser login flow
- **Invite URL generation** for onboarding patients and doctors
- **REST** traffic proxied to the inner backend (e.g. Django DRF)
- **GraphQL** traffic proxied to **Hasura**

The browser must not talk directly to the inner API or Hasura for normal application traffic; those hops go through the gateway after authentication is established.

After login, tokens must be handled in a way that supports validation on the gateway before proxying, while keeping secrets out of plain client storage where possible.

## Decision

### Gateway as the only frontend-facing API

- The SPA calls **only** the Cardio Trace Gateway for auth helpers, invites, REST, and GraphQL (the latter forwarded to Hasura).

### Session cookie (encrypted)

- The gateway **sets and refreshes** a browser cookie that carries an **encrypted session string** bundling **access token**, **refresh token**, and **id token** (opaque to the client).
- The cookie is **`HttpOnly`** (and **`Secure`** / **`SameSite`** as appropriate) so **the SPA cannot read JWTs from JavaScript** and **does not send `Authorization: Bearer`** on its own. **That is the role of the gateway**: decrypt the session on the server, validate tokens, and attach Bearer credentials when calling inner services.
- On each relevant request, the gateway **decrypts the session**, **validates** the token material (e.g. signature, expiry, audience), and **only then** proxies to the inner REST API or Hasura.

### Outbound authorization to backends

- Proxied requests to the inner API and to Hasura include **`Authorization: Bearer <access_token>`** where the access token is taken from the **decrypted session** and **added by the gateway** to the upstream request. Downstream services remain Bearer-authenticated and **never** receive the encrypted cookie from the browser path.

### CSRF and FastAPI

- **FastAPI does not automatically enable CSRF protection.** There is no built-in CSRF middleware comparable to some full-stack frameworks.
- Because the browser holds a **session cookie**, **CSRF must be considered** for browser-initiated, cookie-accompanying requests to the gateway (especially if the SPA origin and gateway could be abused in a cross-site form of attack).
- Mitigations are **application-owned**: e.g. **SameSite** cookie policy, **Origin/Referer** checks on mutating routes, **CSRF tokens** or **double-submit cookie** for state-changing operations, and keeping sensitive actions **same-site** behind the gateway. Document the chosen pattern in gateway implementation docs.

## Consequences

- Inner backend and Hasura **do not** need to parse the encrypted cookie; they receive **`Authorization: Bearer`** on proxied requests **injected by the gateway** from the decrypted session.
- Gateway owns **encryption**, **session lifecycle**, **token refresh** (if applicable), and **pre-proxy validation**—centralizing auth logic and reducing duplicate checks in every inner service.
- Frontend must **not** bypass the gateway for API/GraphQL that requires auth.
- **Security reviews** should explicitly cover **CSRF** for cookie-backed sessions and **XSS** (HttpOnly session cookies avoid exposing JWTs to JS; still assess XSS for other authenticated-session abuse).
- SPA behavior depends on gateway contracts for login URLs, invites, and proxy paths.

## References

- Cardio Trace Gateway (FastAPI) — implementation repository
- Hasura — GraphQL engine behind the gateway

