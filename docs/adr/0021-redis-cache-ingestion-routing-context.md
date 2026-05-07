# ADR 0021: Redis Cache for Ingestion Routing Context

## Status

Proposed

## Context

Per [ADR 0019](0019-sensor-hub-telemetry-ingress-context-boundary.md), Sensor Hub owns telemetry ingress and delegates domain context resolution to backend-owned contracts. Per [ADR 0020](0020-ingestion-context-enrichment-contract-external-telemetry.md), external telemetry services resolve routing context (tenant/device/session) through a dedicated enrichment contract before measurement ingestion.

Telemetry ingestion is high-frequency and repetitive: many consecutive frames often map to the same routing context for a period of time. Resolving full context through domain resolution on every frame increases backend load and adds avoidable latency in the ingestion path.

We need an acceleration layer for ingestion routing context that improves throughput while preserving backend domain services as the source of truth.

## Decision

We introduce Redis as a cache for ingestion routing context used by external telemetry services and ingestion pipelines.

### Cache intent

- Cache frequently reused routing context for tenant/device/session resolution
- Reduce repeated backend resolution calls for stable telemetry streams
- Improve ingestion path latency and throughput under sustained frame rates
- Preserve backend enrichment/domain services as authoritative sources of truth

### Boundary and responsibility


| Concern                                             | Redis cache layer                     | Backend domain service   | Telemetry ingress caller  |
| --------------------------------------------------- | ------------------------------------- | ------------------------ | ------------------------- |
| Routing context authority                           | Not authoritative                     | Source of truth          | Consumes results          |
| Repeated lookup acceleration                        | Fast lookup layer for recent outcomes | —                        | Uses cache-first strategy |
| Lifecycle correctness (assignment/session validity) | Best-effort mirror                    | Source of truth          | Trusts domain outcomes    |
| Cache miss / stale recovery                         | Falls back to enrichment contract     | Resolves canonical state | Retries through contract  |


### Expected behavior

- Callers may use cache-first routing resolution with contract fallback on cache miss or uncertainty
- Cache entries are bounded by explicit expiration policies
- Negative outcomes (for example unknown device or no active session) may be cached for short periods to reduce repeated unresolved lookups
- Lifecycle transitions (assignment/session changes) are reflected through cache refresh and expiration behavior
- Correctness is preserved by fallback to backend contract whenever cache cannot provide trustworthy context

### Failure semantics

- Cache unavailability must degrade gracefully to contract-only resolution
- Cache errors are operational failures, not domain truth
- Stale cache outcomes must be recoverable through enrichment contract calls
- Ingestion correctness must not depend on cache availability

## Consequences

- Lower backend load for repeated routing lookups in high-frequency telemetry streams
- Improved ingestion responsiveness under sustained traffic
- Additional operational component and observability needs (cache health, hit ratio, staleness risk)
- Need to define and tune expiration policies against lifecycle-change frequency
- Potential short-lived inconsistency risk from stale cache entries, mitigated by fallback and bounded TTLs

## References

- [ADR 0005: Mulit-tenancy](0005-mulit-tenancy.md)
- [ADR 0014: Read/Write Split — Hasura for Reads, DRF for Writes](0014-hasura-reads-drf-writes.md)
- [ADR 0017: Device Assignment Lifecycle as Eligibility Timeline](0017-device-assignment-lifecycle-eligibility-timeline.md)
- [ADR 0018: Measurement Sessions as Assignment-Bound Timeline](0018-measurement-sessions-assignment-bound-timeline.md)
- [ADR 0019: Sensor Hub Telemetry Ingress and Context Boundary](0019-sensor-hub-telemetry-ingress-context-boundary.md)
- [ADR 0020: Ingestion Context Enrichment Contract for External Telemetry Services](0020-ingestion-context-enrichment-contract-external-telemetry.md)

