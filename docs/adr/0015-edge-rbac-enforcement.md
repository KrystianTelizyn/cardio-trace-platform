# ADR 0015: Edge RBAC Enforcement at the Gateway

## Status
Proposed

## Context
[ADR 0006](0006-role-permission-model.md) establishes that roles embedded in Auth0 JWTs are the source of truth and that the gateway "enforces route-level authorization before proxying." [ADR 0008](0008-gateway-bff-api-aggregator.md) positions the gateway as the single edge for authenticated SPA traffic while explicitly deferring domain business rules to downstream services.

Today the gateway validates JWTs and forwards trusted identity and tenancy headers to upstream services ([ADR 0012](0012-gateway-backend-trust-contract.md)). However, **no route-level authorization check exists** — every authenticated user, regardless of role, can reach any `/api/{path}` or `/graphql` endpoint. Authorization is entirely deferred to DRF domain logic and Hasura row-level permissions.

This gap means:

- A `patient` can call doctor-only DRF write endpoints (e.g. prescription creation) and rely solely on downstream validation to reject the request
- Unauthorized traffic reaches internal services unnecessarily, increasing attack surface
- There is no centralized audit trail of role-vs-route access attempts

## Decision
We enforce **coarse, allowlist-based RBAC** at the gateway edge, evaluated after authentication and before any request is proxied upstream.

### Policy model

For each externally exposed endpoint class, the gateway defines which roles are permitted. Requests are allowed only when the authenticated role is explicitly permitted for the requested endpoint.

This is an explicit allowlist model with **default deny**: when no allow rule applies, the request is rejected at the edge.

### GraphQL policy

Per [ADR 0014](0014-hasura-reads-drf-writes.md), all SPA reads go through Hasura and all writes go through DRF. The gateway allows all authenticated roles (`doctor`, `patient`) to reach `/graphql`. Hasura enforces row-level and column-level permissions using the forwarded `X-Hasura-Role` session variable.

Operation-level restrictions (e.g. blocking GraphQL mutations at the edge) are deferred until the architecture requires mutations through Hasura.

### Enforcement boundary

| Concern | Gateway (edge RBAC) | Downstream |
|---|---|---|
| Can this role call this endpoint? | Coarse allowlist | — |
| Can this user access this record? | — | DRF domain logic, Hasura row permissions |
| Tenant isolation | Forwards `X-Tenant-Id` / `X-Hasura-Org-Id` | DRF filters by tenant; Hasura `org_id` permission |
| Field-level access | — | Hasura column permissions |

The gateway is a coarse gate. Downstream services remain the authority on domain authorization. This preserves the separation described in [ADR 0008](0008-gateway-bff-api-aggregator.md).

### Failure semantics

Denied requests receive HTTP 403 with a JSON body containing a stable error code. 

### Rollout strategy

Enforcement is active by default for protected traffic: requests with disallowed roles are rejected at the gateway and are not proxied.

Rollout proceeds incrementally by validating role-to-endpoint access behavior in lower environments and promoting once access outcomes match expectations.


## Consequences
- Unauthorized traffic is stopped at the edge, reducing load and attack surface on internal services
- Downstream authorization is preserved unchanged; the gateway does not replace domain rules
- Introducing new exposed endpoints requires an explicit role-access decision — this is an intentional forcing function for security review
- The trust-header forwarding contract ([ADR 0012](0012-gateway-backend-trust-contract.md)) is unchanged; RBAC runs before headers are injected
- RBAC policy changes remain visible in normal architecture and code review workflows
- The `admin` role is not included in the initial policy; it can be added when the role exists in Auth0

## References
- [ADR 0006: Role & Permission Model](0006-role-permission-model.md)
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
- [ADR 0012: Gateway-to-Backend Internal Trust Contract](0012-gateway-backend-trust-contract.md)
- [ADR 0014: Read/Write Split — Hasura for Reads, DRF for Writes](0014-hasura-reads-drf-writes.md)
