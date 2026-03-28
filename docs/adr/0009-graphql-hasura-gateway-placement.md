# ADR 0009: GraphQL Engine Placement (Hasura with the Gateway)

## Status
Proposed

## Context
The platform exposes **GraphQL** to the SPA using **Hasura** as the engine. Hasura provides:

- Schema generation from the database
- Role-based access control using **JWT claims** (aligned with **gateway-injected** `Authorization: Bearer` in [ADR 0003](0003-auth-token-handling.md))
- Declarative **metadata** and **migrations**

We still need to decide **how Hasura is owned in the architecture**:

- As a **separate repository** and independently lifecycle-managed service, or
- As a **first-class companion** of **`cardio-trace-gateway`**: same repository and local/production layout, while remaining a **separate runtime** from the FastAPI process (not “GraphQL inside the Python app”)

Considerations for a portfolio-oriented codebase:

- Simplicity of **development** and **local** onboarding
- Clear story for the **edge**: one public API face ([ADR 0008](0008-gateway-bff-api-aggregator.md))
- **Operations**: fewer moving parts at the start, with a credible path to split later
- **Separation of concerns**: BFF code vs Hasura metadata vs Core Backend domain logic

## Decision
Hasura is an **internal component of the gateway *stack***: it is **versioned and operated together with** `cardio-trace-gateway`, not a second browser-facing product.

- **Repository**: Hasura **metadata, migrations, and related config** live in the **`cardio-trace-gateway` repository**, with a **clear layout** (e.g. `app/` or equivalent for FastAPI vs `hasura/` for engine assets) so concerns do not blur.
- **Runtime**: Hasura runs as its **own process** (e.g. Docker Compose service) next to the gateway; the FastAPI app **proxies** to it—it does not embed the engine.
- **Exposure**: The SPA uses **`/graphql`** (or the chosen path) on the **gateway origin only**; Hasura’s HTTP port is **not** exposed to browsers or the public internet in the default deployment model.
- **Gateway duties** (consistent with ADR 0003 and ADR 0008): authenticate and validate at the edge, then call Hasura with **`Authorization: Bearer <access_token>`** and any other required headers—**added by the gateway** from the decrypted session, not by the browser.

This **does not** move domain business logic into the gateway: Hasura continues to express **data access, permissions, and relationships** via metadata; the Core Backend and database ownership boundaries stay as defined elsewhere.

## Consequences
- **Local dev**: one clone and a **unified** compose (or equivalent) can start gateway + Hasura together.
- **Narrative**: the “edge layer” is one repo: BFF + GraphQL engine behind the same public entrypoint.
- **Operations**: initially simpler than a separate Hasura repo and pipeline; gateway and Hasura releases can still be coordinated in one place.
- **Security**: Hasura stays off the public route table; only the gateway is directly client-reachable.
- **Evolution**: Hasura can later be **extracted** to its own repository or deployment if scale or team boundaries require it—proxy and path contracts stay stable at the gateway.
- **Discipline**: the repo must keep **directories and ownership** explicit so FastAPI code and Hasura assets do not become a single undifferentiated lump.

## References
- [ADR 0003: Auth token handling](0003-auth-token-handling.md)
- [ADR 0008: Gateway as BFF and API Aggregator](0008-gateway-bff-api-aggregator.md)
