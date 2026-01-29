# Project Goal

- Build a Trust Graph with OpenTelemetry that captures who acted on behalf of whom in agent-based systems.

- Represent execution lineage as:

    - Principal → Agent → (Agent → …) → Resource


- Do this without instrumenting application code (no SDKs).

- Use standard infra components:

    - Envoy
    - OpenTelemetry Collector
    - Jaeger
    - Kubernetes (kind)

- Enable future:

    - provenance
    - auditability
    - agent accountability
    - policy enforcement



## Architecture We Chose (MVP)

- Ingress Envoy Gateway

    Sits between external clients and agents.
    Responsible for:

        routing
        starting traces
        emitting spans for Principal → Agent hops.

- OpenTelemetry Collector

    Central telemetry hub.
    Receives traces from Envoy.
    Normalizes trace data.
    Exports traces to Jaeger.
    
- Jaeger

    Trace storage and query.
    Used to visualize and validate lineage.
    Will later back Grafana trust-graph views.

- Dummy Agent Service

    Simple HTTP service in workloads namespace.
    Used to validate:
        
        Kubernetes DNS
        Envoy routing
        end-to-end request flow.

- Kubernetes (kind)

    Local dev cluster.

    Multiple namespaces to mirror production separation:

        ingress-gateway

        workloads

        observability

        egress gateway


## To-Do

1. Creat a clean kind Kubernetes cluster.
2. Establish a namespace model for ingress, workloads, and observability.
3. Deployed Jaeger all-in-one.
4. Deployed OpenTelemetry Collector with:

        OTLP receiver

        Zipkin receiver

        logging exporter for debugging

        OTLP exporter to Jaeger.

5. Deployed Envoy ingress gateway:

        Stable config (no crash loops).

        Routes traffic to agent service.

6. Deployed agent workload and validated:

        Service discovery

        DNS resolution

        Envoy → Agent routing.

7. Verified end-to-end traffic flow:

        curl → Envoy ingress → Agent → response


8. Integrated Envoy → OTel Collector tracing using Zipkin.

9. Confirmed telemetry path:

Envoy → OTel Collector → Jaeger
<!-- 

🔍 Currently validating that traces appear correctly in Jaeger UI.

🧭 What We Have Not Done Yet (Next Steps)

⏭ Add trust headers (x-principal-id, x-agent-id, etc.).

⏭ Add trust. span attributes*.

⏭ Introduce egress Envoy gateway for:

Agent → Resource hops.

⏭ Demonstrate multi-agent lineage in traces.

⏭ Add Grafana trust-graph views.

⏭ Harden config for production-grade usage.

⏭ Integrate with Kagenti / MCP semantics.

🧠 Big Picture

What you’ve built so far is:

A working, minimal, infra-only foundation for distributed trust lineage — proving that Envoy + OTel + Jaeger is sufficient to reconstruct agent execution paths without touching application code.

This is exactly the right MVP foundation to build the full Trust Graph on top of.

If you want, next I can:

map this to formal Trust Graph phases,

help you write a commit message / PR summary,

or define the trust header & span schema cleanly before implementing it. -->