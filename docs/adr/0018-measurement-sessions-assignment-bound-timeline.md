# ADR 0018: Measurement Sessions as Assignment-Bound Timeline

## Status

Proposed

## Context

Per [ADR 0014](0014-hasura-reads-drf-writes.md), measurement ingestion and state-changing operations are handled by backend REST, while SPA reads are handled by Hasura.

[ADR 0017](0017-device-assignment-lifecycle-eligibility-timeline.md) defines device-assignment lifecycle as the eligibility boundary for device use. However, assignment eligibility alone does not indicate when a patient actually starts and stops wearing a heart rate monitoring device.

We introduce measurement sessions to capture this explicit patient signaled wear interval, so ingestion and downstream consumers can distinguish between:

- assignment eligibility window, and
- actual wear/measurement activity window.

Without this separation, frame validity and interpretation become ambiguous across periods when a device is assigned but not worn.

## Decision

We introduce `MeasurementSession` as the canonical patient-declared wear timeline, bounded by assignment eligibility.

### Session timeline model

- A patient explicitly starts a measurement session when beginning to wear the assigned heart rate monitoring device
- A patient explicitly stops the session when no longer wearing the device
- Session start and stop events define the valid measurement activity window
- Measurements are accepted only for active sessions in the correct tenant scope
- Measurements outside a valid active session window are rejected or dropped according to ingestion policy

### Boundary and responsibility


| Concern                                      | Backend API                             | Downstream consumers             |
| -------------------------------------------- | --------------------------------------- | -------------------------------- |
| Session lifecycle state (`active`/`stopped`) | Source of truth                         | Read-only consumption            |
| Assignment-to-session linkage                | Source of truth                         | Use as provided                  |
| Wear activity timeline                       | Captured from patient start/stop intent | Use as canonical activity signal |
| Frame acceptance window                      | Enforced at ingestion boundary          | Assume validated input           |


This preserves the write/read split from [ADR 0014](0014-hasura-reads-drf-writes.md): backend enforces lifecycle and invariants; Hasura and other readers expose and query resulting state.

### Expected behavior

- Starting a session creates a new active wear timeline for ingestion
- Stopping a session closes that wear timeline and prevents further ingestion into it
- Repeated stop requests are idempotent and do not create conflicting state
- Duplicate or stale measurement frames do not create conflicting timeline facts

## Consequences

- Measurement ingestion has deterministic, patient-signaled activity boundaries
- Assignment eligibility and actual wear activity are modeled separately and explicitly
- Session history is auditable as patient-declared wear periods
- External ingestion producers can rely on stable session semantics instead of inferring wear state heuristically
- Analytics and alerting pipelines consume cleaner, activity-bounded data

## References

- [ADR 0005: Mulit-tenancy](0005-mulit-tenancy.md)
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
- [ADR 0012: Gateway-to-Backend Internal Trust Contract](0012-gateway-backend-trust-contract.md)
- [ADR 0014: Read/Write Split — Hasura for Reads, DRF for Writes](0014-hasura-reads-drf-writes.md)
- [ADR 0017: Device Assignment Lifecycle as Eligibility Timeline](0017-device-assignment-lifecycle-eligibility-timeline.md)

