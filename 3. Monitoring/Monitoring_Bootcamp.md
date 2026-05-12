# Monitoring & Observability Bootcamp

## 📖 Phase 1: Theoretical Foundation
* **The Three Pillars of Observability:** Understand the distinct roles of:
    * *Metrics:* Aggregated data over time (e.g., CPU usage, request rate).
    * *Logs:* Discrete events and text records.
    * *Traces:* The lifecycle of a single request across multiple services.
* **OpenTelemetry (OTEL) & OTLP:** Learn about the OpenTelemetry project and the OpenTelemetry Protocol (OTLP) as the industry standard for transmitting telemetry data.
* **Monitoring Stack Overview:** Research the architecture and use-cases of Prometheus, Elastic Search, Loki, Jaeger, and Grafana.

---

## 💻 Phase 2: Practical Lab - Application Instrumentation
**Objective:** Make your simple app observable.

1.  Take the simple app you developed in the CI/CD bootcamp and instrument it.
2.  **Metrics:** Expose an endpoint (e.g., `/metrics`) that outputs application metrics (like request count or error rate) in a Prometheus-compatible format.
3.  **Logs:** Modify your app to output serialized, structured logs (e.g., JSON format) to `stdout` or a specific file. 
4.  **Traces:** Instrument your app's HTTP handler to generate trace data and export it via OTLP or directly to a Jaeger backend.

---

## 🚀 Phase 3: Practical Lab - Infrastructure Setup
**Objective:** Deploy a full observability stack inside Kubernetes.

1.  Deploy the following tools into your K3s cluster:
    * **Prometheus:** Configure it to scrape the `/metrics` endpoint of your newly instrumented application.
    * **Loki:** Set up Loki to ingest the serialized logs from your application's deployment (capturing `stdout` or log files).
    * **Jaeger:** Deploy Jaeger and ensure your application is correctly sending its distributed trace data to the Jaeger collector.
    * **Grafana:** Deploy Grafana and connect it to your three data sources: Prometheus, Loki, and Jaeger.

---

## 📊 Phase 4: Practical Lab - Dashboarding
**Objective:** Visualize your application's health and performance.

1.  Log into Grafana.
2.  Create a unified, basic dashboard for your application that includes:
    * Graphs showing metrics from Prometheus (e.g., request rate).
    * A panel displaying the serialized logs from Loki.
    * A table or panel linking to traces from Jaeger.
