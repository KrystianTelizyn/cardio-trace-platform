# ADR 0020: Ingestion Context Enrichment Contract for External Telemetry Services

## Status

Proposed

## Context

Per [ADR 0014](0014-hasura-reads-drf-writes.md), ingestion is handled through backend write APIs. External telemetry producers (for example Sensor Hub) submit high-frequency heart-rate frames and must map raw device identity into platform context before measurement ingestion.

[ADR 0017](0017-device-assignment-lifecycle-eligibility-timeline.md) defines assignment eligibility, and [ADR 0018](0018-measurement-sessions-assignment-bound-timeline.md) defines session activity windows. External telemetry services should not replicate tenant/device/session resolution logic independently, because this creates drift and inconsistent ingestion outcomes.

We need a stable contract that lets external telemetry services resolve ingestion context from device identity using platform-owned rules.

## Decision

We define a dedicated ingestion context enrichment contract between external telemetry services and the backend domain service.

### Contract intent

- External telemetry services provide raw device identity and tenant scope
- Backend resolves routing context needed for measurement ingestion
- Resolution result is returned in a stable, machine-consumable form
- External telemetry services use resolved context for subsequent measurement submission

### Ownership boundaries


| Concern                                          | Backend domain service             | External telemetry service     |
| ------------------------------------------------ | ---------------------------------- | ------------------------------ |
| Tenant/device/session resolution rules           | Source of truth                    | Does not re-implement          |
| Device identity collection from upstream streams | —                                  | Source of truth                |
| Measurement frame transport and retry policy     | Validates and applies domain rules | Owns delivery behavior         |
| Handling unresolved context outcomes             | Defines response semantics         | Applies fallback/skip behavior |


### Expected behavior

- Enrichment may return either a resolvable ingestion context or an unresolved outcome
- Resolved context is scoped to tenant and current lifecycle state (assignment/session)
- Unresolved outcomes are explicit and non-ambiguous, so callers can safely skip or defer ingestion
- Contract semantics remain stable as internal resolution logic evolves
- Measurement ingestion uses enrichment outputs rather than duplicated caller-side inference

### Failure semantics

- Validation failures are reported as contract errors
- Unknown device identity and missing active session are reported as resolvable domain outcomes, not transport failures
- Service failures are reported separately from domain outcomes so callers can distinguish retryable vs non-retryable paths

## Consequences

- External telemetry integrations become simpler and more consistent across producers
- Domain resolution logic remains centralized and auditable in one place
- Ingestion behavior aligns with assignment and session ADRs without duplicating lifecycle rules
- New telemetry producers can integrate faster by implementing one stable contract
- Contract versioning and compatibility become an explicit architecture concern

## References

- [ADR 0005: Mulit-tenancy](0005-mulit-tenancy.md)
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
- [ADR 0012: Gateway-to-Backend Internal Trust Contract](0012-gateway-backend-trust-contract.md)
- [ADR 0014: Read/Write Split — Hasura for Reads, DRF for Writes](0014-hasura-reads-drf-writes.md)
- [ADR 0017: Device Assignment Lifecycle as Eligibility Timeline](0017-device-assignment-lifecycle-eligibility-timeline.md)
- [ADR 0018: Measurement Sessions as Assignment-Bound Timeline](0018-measurement-sessions-assignment-bound-timeline.md)

