# Cardio Trace Platform

Cardio Trace Platform is a prototype IoT platform demonstrating heart rate monitoring, HRV analysis, and AI integration. It showcases multi-tenant architecture, asynchronous processing, and simulated sensor data flows.

![Architecture Overview](docs/diagrams/architecture.png)

## Purpose

- Define the overall system architecture for Cardio Trace IoT platform.
- Document design decisions, technologies, and data flow.
- Serve as the source of truth for all platform components and services.
- Provide diagrams and guidance for implementing, integrating, and extending the platform.

## Related Repositories

The platform consists of multiple service repositories:

- [cardio-trace-iot-platform](https://github.com/KrystianTelizyn/cardio-trace-platform) — meta repository with system documentation (GitHub repo: `cardio-trace-platform`). You are here.
- [cardio-trace-iot-simulation](https://github.com/KrystianTelizyn/cardio-trace-iot-simulation) — **simulates** MQTT device traffic (publishers) for testing and demos.
- `cardio-trace-sensor-hub` — **ingests** MQTT measurements: validate, normalize, and forward sensor data into the platform.
- [cardio-trace-iot-backend-api](https://github.com/KrystianTelizyn/cardio-trace-backend-api) — core domain logic and data management.
- `cardio-trace-workers` — async processing, alerts, and AI tasks.
- `cardio-trace-ai-service` — AI insights, analysis, and chat functionality.
- [cardio-trace-gateway](https://github.com/KrystianTelizyn/cardio-trace-gateway) — FastAPI BFF and API edge: Auth0 flows, session cookie, REST proxy to the core backend, GraphQL proxy to Hasura (Hasura metadata/migrations live in this repo).
- `cardio-trace-frontend` — web interface for monitoring and visualization.
- `cardio-trace-deployment` — deployment configurations and scripts.

## Repository Structure

This documentation repo is usually cloned as `cardio-trace-platform` (GitHub default) or a variant such as `cardio-trace-platform-architecture`.

```bash
cardio-trace-platform   # or your local clone name
│
├─ docs/
│   ├─ adr/                 # Architecture Decision Records
│   │   └─ README.md
│   ├─ diagrams/            # System, data flow, deployment diagrams
│   │   └─ system-overview.drawio
│   └─ architecture.md      # System architecture description
│
├─ scripts/                 # Dev tools, helpers, analysis scripts (optional)
│
└─ README.md
```

## Getting Started

1. Review the **architecture overview** in `docs/architecture.md`.
2. Familiarize yourself with **ADRs** in `docs/adr/`.
3. Refer to **related service repositories** for implementation details.
4. Deployment instructions and environment setup are located in the `cardio-trace-deployment` repository.

## Contributing

- All architecture-related decisions must be documented in `docs/adr/`.
- Suggestions for architecture improvements, new diagrams, or ADRs are welcome via pull requests.
- Follow consistent naming and documentation conventions.

## Contact

Cardio Trace Platform is maintained by Krystian Teliżyn.  
For detailed information, refer to the documentation and ADRs.
