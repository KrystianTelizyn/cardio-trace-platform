# ADR 0016: Dedicated Repository for Hasura Configuration

## Status
Proposed

## Context
The platform architecture establishes Hasura as the GraphQL read path behind the gateway ([ADR 0009](0009-graphql-hasura-gateway-placement.md), [ADR 0014](0014-hasura-reads-drf-writes.md)). Hasura behavior is driven by metadata, permission rules, actions, and environment-specific configuration.

Today there is no explicit architectural decision about where this configuration should live. Keeping Hasura configuration mixed with other service code can blur ownership boundaries, make schema and permission changes harder to review, and couple GraphQL changes to unrelated backend release cycles.

We need a clear, version-controlled home for Hasura configuration that aligns with the gateway trust contract and tenant/role authorization model ([ADR 0012](0012-gateway-backend-trust-contract.md), [ADR 0015](0015-edge-rbac-enforcement.md)).

## Decision
We create and maintain a dedicated repository, `cardio-trace-graphql`, as the single source of truth for Hasura configuration.

The repository owns:

- Hasura metadata (tables, relationships, permissions, actions, remote schemas)
- GraphQL-facing configuration required to run Hasura across environments
- CI checks for metadata consistency

### Ownership and boundaries

- GraphQL schema exposure and permission rules are owned in `cardio-trace-graphql`
- Database schema migrations remain owned by `backend-api` and are not stored in `cardio-trace-graphql`
- Domain write logic and business invariants remain in Django DRF per [ADR 0014](0014-hasura-reads-drf-writes.md)
- Gateway authentication, trust-header injection, and edge policy remain owned by gateway repositories per [ADR 0012](0012-gateway-backend-trust-contract.md) and [ADR 0015](0015-edge-rbac-enforcement.md)

### Change and release model

Hasura metadata changes are proposed, reviewed, and released from `cardio-trace-graphql` independently from DRF and gateway code changes, while still coordinating through standard architecture and API review when contracts change. Database schema changes continue to be proposed and released from `backend-api`, then reflected in Hasura metadata.

## Consequences
- Clear ownership of GraphQL/Hasura concerns with a dedicated review surface
- Independent release cadence for GraphQL metadata and permissions
- Improved auditability of tenant and role permission changes
- Additional operational overhead from an extra repository and CI pipeline
- Cross-repository coordination is required when changes span database migrations (`backend-api`), gateway trust headers, and Hasura metadata

## References
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
- [ADR 0009: GraphQL Engine Placement](0009-graphql-hasura-gateway-placement.md)
- [ADR 0012: Gateway-to-Backend Internal Trust Contract](0012-gateway-backend-trust-contract.md)
- [ADR 0014: Read/Write Split — Hasura for Reads, DRF for Writes](0014-hasura-reads-drf-writes.md)
- [ADR 0015: Edge RBAC Enforcement at the Gateway](0015-edge-rbac-enforcement.md)
