# ADR 0017: Device Assignment Lifecycle as Eligibility Timeline

## Status
Proposed

## Context
Per [ADR 0014](0014-hasura-reads-drf-writes.md), measurement ingestion and state-changing operations are handled by backend REST, while SPA reads are handled by Hasura.

The platform supports doctor-managed device assignment as a core domain workflow. Without an explicit lifecycle rule for assignments, device eligibility and ownership over time can become ambiguous (e.g. assignment ended, reassigned device, overlapping ownership assumptions).

We need a platform-level decision that defines assignment timeline semantics and eligibility boundaries.

## Decision
We treat `DeviceAssignment` lifecycle as the canonical eligibility timeline for device-to-patient use in tenant scope.

### Assignment lifecycle model

- A doctor creates an assignment between a device and a patient in tenant scope
- An assignment is active from `assigned_at` until `unassigned_at`
- Only active assignments are eligible for downstream workflows that depend on assignment validity
- Assignment state changes are the source of truth for eligibility boundaries

### Boundary and responsibility

| Concern | Backend API | Downstream consumers |
|---|---|---|
| Assignment lifecycle state (`active`/`inactive`) | Source of truth | Read-only consumption |
| Device-to-patient eligibility over time | Source of truth | Use as provided |
| Workflow eligibility that depends on assignment state | Enforced from assignment state | Assume validated input |
| Longitudinal analytics / visualization | Provides assignment-bounded facts | Build higher-level interpretation |

This preserves the write/read split from [ADR 0014](0014-hasura-reads-drf-writes.md): backend enforces lifecycle and invariants; Hasura and other readers expose and query resulting state.

### Expected behavior

- Doctor assignment enables downstream workflows for that device assignment
- Unassignment closes future eligibility for new dependent workflow starts on that assignment
- Workflow starts outside active assignment windows are rejected
- Reassignment creates a new eligibility window with explicit timeline continuity boundaries

## Consequences
- Assignment-based eligibility becomes deterministic and auditable
- Device ownership/use context over time becomes explicit, including reassignment boundaries
- Measurement session policy can be defined as a consequence of assignment policy in a dedicated ADR
- Ingestion and analytics can rely on assignment-scoped temporal semantics instead of inferred ownership
- Product flows must explicitly manage doctor assignment and unassignment lifecycle events

## References
- [ADR 0005: Mulit-tenancy](0005-mulit-tenancy.md)
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
- [ADR 0012: Gateway-to-Backend Internal Trust Contract](0012-gateway-backend-trust-contract.md)
- [ADR 0014: Read/Write Split — Hasura for Reads, DRF for Writes](0014-hasura-reads-drf-writes.md)