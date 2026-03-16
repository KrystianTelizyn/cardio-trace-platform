# ADR 0001: Use MQTT as Sensor Communication Protocol

## Status
Proposed

## Context
The **Cardio Trace Platform** collects real-time heart rate (HR) and heart rate variability (HRV) data, together with records of cardiovascular medication administration. The system operates in **multi-tenant mode**, supporting multiple doctors and patients simultaneously.  

In the medical IoT domain, heart rate monitoring devices are commonly integrated using **MQTT** for reliable, low-overhead, asynchronous data delivery.  

The platform aims to **simulate sensor-device traffic** over MQTT to reproduce realistic data flows for testing, validation, and demonstration purposes, without requiring real medical devices.

## Decision
We will use **MQTT** as the protocol to simulate sensor data delivery in the platform.  

Key points:

- Each simulated device publishes to a dedicated topic, e.g., `/sensors/<device_id>/measurements`.
- Later system services will be able to ingest, validate, and normalize data (sensor-hub).
- MQTT QoS 1 (at least once) is used to balance reliability and performance.
- Retained messages are used only for simulated device status or metadata.
- This approach aligns with **typical medical IoT architectures** where HR devices communicate via MQTT.

## Consequences
- No actual medical devices are required; sensor traffic is **simulated**.
- Need to introduce sensor-hub, the key point for ingestion, validation, and forwarding of simulated messages.
- Future integration with real devices would be straightforward, as the simulated topics mirror real MQTT structures.

## References
- MQTT Protocol Specification: [OASIS MQTT v5.0](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
- Cardio Trace Architecture Overview (docs/diagrams/cardio-trace-architecture.md)
- Best practices for Medical IoT device integration
