# ADR 0015: Edge RBAC Enforcement at the Gateway

## Status
Proposed

## Context
[ADR 0006](0006-role-permission-model.md) establishes that roles embedded in Auth0 JWTs are the source of truth and that the gateway "enforces route-level authorization before proxying." [ADR 0008](0008-gateway-bff-api-aggregator.md) positions the gateway as the single edge for authenticated SPA traffic while explicitly deferring domain business rules to downstream services.

Today the gateway validates JWTs and extracts a `TrustContext` (user, tenant, role) that is forwarded to upstream services via trust headers ([ADR 0012](0012-gateway-backend-trust-contract.md)). However, **no route-level authorization check exists** — every authenticated user, regardless of role, can reach any `/api/{path}` or `/graphql` endpoint. Authorization is entirely deferred to DRF domain logic and Hasura row-level permissions.

This gap means:

- A `patient` can call doctor-only DRF write endpoints (e.g. prescription creation) and rely solely on downstream validation to reject the request
- Unauthorized traffic reaches internal services unnecessarily, increasing attack surface
- There is no centralized audit trail of role-vs-route access attempts

## Decision
We add **coarse, allowlist-based RBAC** at the gateway edge, evaluated after JWT validation and before any request is proxied. The enforcement engine is **Casbin** (pycasbin), using a declarative model and policy file.

### Policy model

Each rule maps an **(HTTP method, path pattern)** pair to the roles permitted to access it. Rules are defined in a Casbin CSV policy file shipped with the gateway source. The Casbin model uses `keyMatch2` for path matching, supporting wildcards (e.g. `/patients/*` matches `/patients/123`). **Default-deny** applies when no rule matches a request.

The policy file is the single source of truth for edge RBAC rules. Adding, removing, or changing rules is a policy file edit — no Python code change is required.

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

Enforcement mode is controlled by the `RBAC_ENFORCEMENT_MODE` environment variable:

- **`audit`** (default) — violations are logged but requests are **not** blocked. Allows validation of the policy against real traffic before enforcement.
- **`enforce`** — violations return 403 and the request is not proxied.

**Phase 1:** Deploy with `audit`. Monitor logs for false positives. Adjust policy as needed.
**Phase 2:** Flip to `enforce` once validated. Rollback is a single env var change back to `audit`.


## Consequences
- Unauthorized traffic is stopped at the edge, reducing load and attack surface on internal services
- Audit mode enables safe rollout and ongoing monitoring without blocking traffic
- Downstream authorization is preserved unchanged; the gateway does not replace domain rules
- Adding new DRF endpoints requires a corresponding policy rule — this is an intentional forcing function for security review
- The trust-header forwarding contract ([ADR 0012](0012-gateway-backend-trust-contract.md)) is unchanged; RBAC runs before headers are injected
- Policy changes are isolated to a CSV file, reviewable in standard code review without touching Python source
- The `admin` role is not included in the initial policy; it can be added when the role exists in Auth0

## References
- [ADR 0006: Role & Permission Model](0006-role-permission-model.md)
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
- [ADR 0012: Gateway-to-Backend Internal Trust Contract](0012-gateway-backend-trust-contract.md)
- [ADR 0014: Read/Write Split — Hasura for Reads, DRF for Writes](0014-hasura-reads-drf-writes.md)
