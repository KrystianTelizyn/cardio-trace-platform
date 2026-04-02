# ADR 0011: Gateway Redirect Safety and Proxy Header Policy

## Status
Proposed

## Context
The gateway is the browser-facing edge for login/callback redirects and proxying to inner REST and GraphQL services (ADR 0008, ADR 0009). This makes two security-sensitive behaviors architectural concerns:

- Redirect target handling for login, invite, and logout flows (`return_to`)
- HTTP header forwarding between browser and upstream services

Without explicit rules:

- Redirect parameters can introduce open-redirect risk
- Unbounded header forwarding can leak or propagate unsafe/inappropriate headers

The gateway implementation now applies concrete rules for both concerns; these rules should be documented as a durable architecture contract.

## Decision
We define the following gateway security policies.

### Redirect safety (`return_to`)

- Accept relative paths that start with `/`.
- Reject protocol-relative (`//...`) and malformed relative paths by falling back to `/`.
- For absolute URLs, allow only same-origin redirects matching configured frontend origin.
- If absolute URL origin does not match configured frontend origin, fall back to `/`.
- Callback success and error redirects are always built from configured frontend base URL and fixed callback paths.

**Logout (Auth0 RP-initiated logout):**

- The SPA may send an optional `return_to` in the JSON body of **`POST /logout`** (same validation rules as above). Omitted or invalid values fall back to the default path (`/`).
- After clearing the gateway session cookie, the gateway returns an **Auth0 logout URL** whose `returnTo` query parameter is an **absolute URL** built from the configured frontend base URL and the normalized path. The SPA completes logout by **navigating** the browser to that URL (full page load).
- **Auth0** configuration must list allowed post-logout destinations: the Auth0 application’s **Allowed Logout URLs** must include every frontend URL that may appear as `returnTo` (typically the frontend origin and specific paths you pass after normalization).

### Proxy request/response header policy

- Gateway always injects `Authorization: Bearer <access_token>` to upstream REST and GraphQL calls.
- Browser request headers are forwarded using an allowlist (`accept`, `accept-encoding`, `accept-language`, `content-type`, `user-agent`, `x-request-id`).
- Hop-by-hop response headers (for example `connection`, `transfer-encoding`, `upgrade`) are not forwarded back to clients.

## Consequences
- Redirect behavior is predictable and resistant to open-redirect abuse when untrusted `return_to` is provided (including on logout).
- Operations must keep Auth0 **Allowed Logout URLs** aligned with frontend routes used as post-logout landing pages.
- Proxy boundary is intentionally narrow and auditable, reducing accidental header propagation across trust boundaries.
- Inner services receive a consistent authorization contract from the gateway.
- Any future expansion of forwarded headers or changes to redirect origin policy should be reviewed as architecture/security changes.

## References
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
- [ADR 0009: GraphQL Engine Placement](0009-graphql-hasura-gateway-placement.md)
