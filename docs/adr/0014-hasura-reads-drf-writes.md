# ADR 0014: Read/Write Split — Hasura for Reads, DRF for Writes

## Status
Proposed

## Context
The platform exposes two data paths to the SPA through the gateway ([ADR 0008](0008-gateway-bff-api-aggregator.md)):

- **REST** (`/api/*`) → proxied to the Core Backend (Django DRF)
- **GraphQL** (`/graphql`) → proxied to Hasura ([ADR 0009](0009-graphql-hasura-gateway-placement.md))

Both can serve reads and writes. Without an explicit rule, the same data could be read from either path, leading to duplicated serialization logic, inconsistent filtering, and unclear ownership.

We need a clear principle for which path handles which operations.

## Decision
We adopt a **read/write split**:

- **All SPA data reads** are served by **Hasura** via GraphQL
- **All writes and domain logic** are served by **Django DRF** via REST
- **Internal service ingestion** (sensor-hub, workers) calls DRF REST directly

### What Hasura handles (reads)

Hasura auto-generates queries from the PostgreSQL schema with filtering, sorting, pagination, and nested relationship traversal. Row-level permissions enforce tenant isolation and role-based access using gateway-forwarded session variables ([ADR 0012](0012-gateway-backend-trust-contract.md)).

The SPA uses Hasura for all data fetching: patient lists, measurements, prescriptions, alerts, devices, reference data, and doctor profiles.

### What DRF handles (writes + identity)

DRF endpoints handle operations that require business logic, validation, or side effects:

- **User provisioning** (`POST /users`) — creates User + domain profile on first login ([ADR 0013](0013-user-provisioning-first-login.md))
- **Measurement ingestion** (`POST /measurements`) — resolves device serial number to patient and tenant
- **Prescription creation/update** — auto-sets doctor_id and date_assigned from auth context, validates drug reference
- **Dose event recording** — validates prescription ownership
- **Alert creation** — internal ingestion from workers pipeline
- **Alert acknowledgement** — domain state change with optional doctor note
- **Device registration** — uniqueness validation on serial_number
- **Device assignment/removal** — enforces one-active-assignment-per-device rule

DRF does **not** implement read-only list or detail endpoints for data that Hasura already serves.

### SPA data flow

1. SPA calls **gateway `GET /me`** on init → receives auth identity (role, email, picture) from gateway's own session data
2. SPA queries **Hasura** for all data reads (profiles, measurements, prescriptions, alerts, etc.) using session-variable-based permissions
3. SPA calls **DRF REST** endpoints (via gateway `/api/*`) only for write operations

## Consequences
- Clear ownership: one path per operation type, no duplication
- DRF codebase is smaller — only write endpoints, no read serializers or filter logic for data Hasura already serves
- Hasura handles the complex read patterns (nested joins, filtering, pagination) that would require significant DRF code
- Adding new read queries is a Hasura metadata change, not a DRF code change
- Adding new write operations with business logic goes through DRF, where Django's ORM, validation, and transaction support are available
- The SPA must use two protocols (GraphQL for reads, REST for writes) — this is a trade-off, but both go through the same gateway origin
- If a write operation needs to return complex nested data in its response, DRF returns a flat confirmation and the SPA can refetch via Hasura

## References
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
- [ADR 0009: GraphQL Engine Placement](0009-graphql-hasura-gateway-placement.md)
- [ADR 0012: Gateway-to-Backend Internal Trust Contract](0012-gateway-backend-trust-contract.md)
- [Backend API Specification](../api/backend-api-spec.md)
- [GraphQL Specification](../api/hasura-graphql-spec.md)
