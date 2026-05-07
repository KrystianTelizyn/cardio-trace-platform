# ADR 0019: Sensor Hub Telemetry Ingress and Context Boundary

## Status
Proposed

## Context
Per [ADR 0014](0014-hasura-reads-drf-writes.md), ingestion is handled through backend write APIs. The Sensor Hub receives high-frequency telemetry frames from MQTT topics and forwards valid measurements to backend ingestion endpoints.

[ADR 0017](0017-device-assignment-lifecycle-eligibility-timeline.md) and [ADR 0018](0018-measurement-sessions-assignment-bound-timeline.md) define lifecycle semantics for eligibility and activity windows. MQTT telemetry frames do not carry complete backend domain context by default (for example assignment/session identifiers), so context resolution cannot be assumed at ingress.

The platform introduces Sensor Hub because raw telemetry ingestion has concerns that should be separated from domain write logic: sustained MQTT connectivity, payload normalization across device formats, and resilient handling of high-frequency transport events. Without this boundary, backend domain services would need to absorb transport/protocol variability and operational ingestion concerns that are not part of core domain ownership.

We need a clear boundary for what Sensor Hub owns at telemetry ingress and what remains backend-owned domain context.

## Decision
We define Sensor Hub as the telemetry ingress boundary that normalizes transport/payload context and delegates routing context resolution to backend-owned enrichment contracts.

### Ingress boundary intent

- Sensor Hub owns reliable ingestion of MQTT telemetry frames from configured topics
- Sensor Hub extracts transport-level identity available at ingress (for example tenant signal from topic and raw device identity from payload)
- Sensor Hub does not authoritatively infer domain lifecycle context (assignment/session state)
- Sensor Hub submits measurements only after backend routing context has been resolved through explicit enrichment contracts

### Ownership boundaries

| Concern | Sensor Hub | Backend domain service |
|---|---|---|
| MQTT connectivity, topic subscription, payload ingestion | Source of truth | — |
| Transport-level parsing and normalization | Source of truth | — |
| Tenant/device/session lifecycle resolution rules | Does not re-implement | Source of truth |
| Measurement acceptance policy | Sends candidate frames | Validates and applies domain rules |
| Domain unresolved outcomes | Applies caller behavior (drop/defer/retry by policy) | Defines domain outcome semantics |

### Expected behavior

- Sensor Hub acts as a stateless-to-contextual bridge between telemetry transport and backend ingestion APIs
- Frames that cannot be mapped to valid domain context are not ingested as accepted measurements
- Domain lifecycle decisions remain centralized in backend services
- Sensor Hub behavior remains compatible with backend contract evolution as long as contract semantics are preserved

### Failure semantics

- Transport/payload failures are treated as ingress processing failures
- Domain unresolved outcomes (for example unknown device, missing active session) are handled separately from transport failures
- Backend service failures are treated separately from domain unresolved outcomes so retry behavior can differ

## Consequences
- Sensor Hub scope is clear: telemetry ingress and normalization, not domain authority
- Domain lifecycle logic remains centralized and auditable in backend services
- Ingestion outcomes stay aligned with assignment/session ADRs without duplicating rules at the edge
- Additional telemetry producers can follow the same ingress-boundary pattern
- A dedicated enrichment contract ADR is required for backend context resolution details

## References
- [ADR 0005: Mulit-tenancy](0005-mulit-tenancy.md)
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
- [ADR 0012: Gateway-to-Backend Internal Trust Contract](0012-gateway-backend-trust-contract.md)
- [ADR 0014: Read/Write Split — Hasura for Reads, DRF for Writes](0014-hasura-reads-drf-writes.md)
- [ADR 0017: Device Assignment Lifecycle as Eligibility Timeline](0017-device-assignment-lifecycle-eligibility-timeline.md)
- [ADR 0018: Measurement Sessions as Assignment-Bound Timeline](0018-measurement-sessions-assignment-bound-timeline.md)
- [ADR 0020: Ingestion Context Enrichment Contract for External Telemetry Services](0020-ingestion-context-enrichment-contract-external-telemetry.md)
