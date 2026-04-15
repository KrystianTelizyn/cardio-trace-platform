# ADR 0012: Gateway-to-Backend Internal Trust Contract

## Status
Proposed

## Context
Multiple ADRs establish that the gateway authenticates browser traffic and proxies to internal services with "minimal trusted request context" ([ADR 0003](0003-auth-token-handling.md), [ADR 0004](0004-backend-auth-separation.md), [ADR 0008](0008-gateway-bff-api-aggregator.md), [ADR 0011](0011-gateway-redirect-and-header-policy.md)). However, no ADR defines the **concrete headers** that form this contract.

The Core Backend (Django DRF) and Hasura both need to know exactly which headers carry identity, tenant, and role information so they can enforce domain authorization and tenant isolation without parsing JWTs or session cookies.

Internal platform services (sensor-hub, workers) also call the Core Backend directly, bypassing the gateway. Their authentication model needs to be explicit.

## Decision
We define the following internal trust contract.

### Gateway → Core Backend (browser-originated traffic)

The gateway decrypts the session, validates the token, and attaches these headers before proxying to `/api/*`:

| Header         | Value                                     | Source                              |
|----------------|-------------------------------------------|-------------------------------------|
| `X-User-Id`    | Auth0 user ID (`sub`, e.g. `auth0|...`)   | From `sub` claim                    |
| `X-Tenant-Id`  | Auth0 organization ID (`org_id`)          | From `org_id` claim                 |
| `X-Role`       | Single role, e.g. `doctor`                | From `https://cardio-trace.com/roles` claim |

The Core Backend reads these headers and trusts them unconditionally. It does **not** validate JWTs, parse cookies, or call Auth0.

### Gateway → Hasura (browser-originated traffic)

The gateway attaches Hasura session variables before proxying to the GraphQL engine:

| Header              | Value                                     | Source                              |
|---------------------|-------------------------------------------|-------------------------------------|
| `X-Hasura-User-Id`  | Local user ID (int)                       | Resolved from `sub` claim           |
| `X-Hasura-Org-Id`   | Local tenant ID (int)                     | Resolved from `org_id` claim        |
| `X-Hasura-Role`     | Single role, e.g. `doctor`                | From `https://cardio-trace.com/roles` claim |

Hasura uses these session variables in its row-level permission rules.

### Internal services → Core Backend (service-to-service)

Platform services (sensor-hub, workers) call the Core Backend directly over the internal Docker network. They do **not** send gateway trust headers. Instead:

- Requests are authenticated by **network isolation**: only containers on the internal Docker network can reach the backend.
- Internal endpoints (e.g. `POST /users`, `POST /measurements`, `POST /alerts`) accept tenant and entity identifiers in the request body, not headers.
- For additional defense in depth, an optional shared secret (`X-Internal-Token`) may be introduced later without changing the API contract.

### Network boundary requirement

The Core Backend and Hasura must **not** be directly reachable from outside the internal Docker/container network. Only the gateway is exposed to browsers and the public internet. If the backend were accidentally exposed, the header-trust model would allow spoofing. Network isolation is the primary security control.

## Consequences
- Core Backend and Hasura have a concrete, documented header contract to implement against
- Gateway is the single point of header injection for browser traffic; backend code is simpler (no JWT library, no Auth0 dependency)
- Network isolation is a hard requirement, not optional — it must be verified in deployment configuration
- Changing the header names or adding new context (e.g. permissions) requires updating this ADR and both the gateway and backend
- The contract is the same for REST (DRF) and GraphQL (Hasura) in intent, but uses different header names to match Hasura's session variable convention

## References
- [ADR 0003: Authentication Token Handling Strategy](0003-auth-token-handling.md)
- [ADR 0004: Separation of Gateway Auth from Core Backend](0004-backend-auth-separation.md)
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
- [ADR 0011: Gateway Redirect Safety and Proxy Header Policy](0011-gateway-redirect-and-header-policy.md)
