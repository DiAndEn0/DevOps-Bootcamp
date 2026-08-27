  **Up:** [[3. Monitoring]]
  
---

# Metrics

[Source](https://medium.com/flux-it-thoughts/logs-traces-and-metrics-5e6d92877c23)

Metrics are created as a quantifiable measure of an app's performance and quality and is needed by the development to accurately measure efficiency and reliability of the software. They are able to track multitude of things from the entire development pipeline. 

![[Pasted image 20260825124630.png]]

## Types of metrics:

### Counter - 
Counters are able used as cumulative objects; they count increasing metrics such as total requests, requests sent and etc. They are used *exclusively* for a value that can only increase and cannot decrease over time, there is a different metric which is able to measure that.

### Gauge - 
Gauges are used to measure an arbitrary value that can fluctuate between more or less. For example; being used for concurrent requests, response time, uptime and more.

### Histogram - 
Histograms are a visual representation of a distribution of values. It divides raw data into configurable "buckets" aka intervals and counts, sums or does other things to the data that falls within every interval or "bucket" which it exposes as a metric, like < histogram_>count or < histogram_>sum. 
![[Pasted image 20260825130031.png]]
# Logs

Logs are written by the software as detailed notes in order to help understand what it's doing step by step. If something fails, logs are usually a good place to check out for information about it. They're describing happening as the software runs, and they record a myriad of things; errors, startup messages, actions it performs and more. These logs are written as an `stdout` or to a local file. With logs we can:
- Find bugs.
- Monitor application's health
- Keep a history of important actions.
- Analyze performance.
- Comply with different security and industry regulations.

# Traces

**Traces** record the end-to-end behavior of a request, which may involve several services and technologies. They help identify bottlenecks, show which calls a service makes with what parameters, what it receives in return, how long it takes, and they simplify analysis in scenarios where services orchestrate others. Traces are collected at a sampling rate between 0 and 1, where 0 is no sampling, and 1 means sampling everything. When working with traces, each trace is linked to an event and is made up of several spans. These spans represent the time it takes for a part of the request to be processed by our system.

![[Pasted image 20260825141154.png]]

---
---

# OpenTelemetry 

## Observability

To first understand OpenTelemetry we will look on the principles that it was built on. Observability is the ability to understand the inner workings of a system by reading its outputs, without knowing the underlying code. In order to do so, the application must be **instrumented** to output things like metrics, traces and logs which then should be exported to the appropriate observability backends. 

## OpenTelemetry and What is it

OpenTelemetry is:

- An **[observability](https://opentelemetry.io/docs/concepts/observability-primer/#what-is-observability) framework and toolkit** designed to facilitate the
    
    - [Generation](https://opentelemetry.io/docs/concepts/instrumentation)
    - Export
    - [Collection](https://opentelemetry.io/docs/concepts/components/#collector)
    
    of [telemetry data](https://opentelemetry.io/docs/concepts/signals/) such as [traces](https://opentelemetry.io/docs/concepts/signals/traces/), [metrics](https://opentelemetry.io/docs/concepts/signals/metrics/), and [logs](https://opentelemetry.io/docs/concepts/signals/logs/).
    
- **Open source**, as well as **vendor- and tool-agnostic**, meaning that it can be used with a broad variety of observability backends, including open source tools like [Jaeger](https://www.jaegertracing.io/) and [Prometheus](https://prometheus.io/), as well as commercial offerings. OpenTelemetry is **not** an observability backend itself.
    

A major goal of OpenTelemetry is to enable easy instrumentation of your applications and systems, regardless of the programming language, infrastructure, and runtime environments used.

The backend (storage) and the frontend (visualization) of telemetry data are intentionally left to other tools.

## Main OpenTelemetry components[](https://opentelemetry.io/docs/what-is-opentelemetry/#main-opentelemetry-components)

OpenTelemetry consists of the following major components:

- A [specification](https://opentelemetry.io/docs/specs/otel/) for all components
- A standard [protocol](https://opentelemetry.io/docs/specs/otlp/) that defines the shape of telemetry data
- [Semantic conventions](https://opentelemetry.io/docs/specs/semconv/) that define a standard naming scheme for common telemetry data types
- APIs that define how to generate telemetry data
- [Language SDKs](https://opentelemetry.io/docs/languages) that implement the specification, APIs, and export of telemetry data
- A [library ecosystem](https://opentelemetry.io/ecosystem/registry/) that implements instrumentation for common libraries and frameworks
- Automatic instrumentation components that generate telemetry data without requiring code changes
- The [OpenTelemetry Collector](https://opentelemetry.io/docs/collector), a proxy that receives, processes, and exports telemetry data
- Various other tools, such as the [OpenTelemetry Operator for Kubernetes](https://opentelemetry.io/docs/platforms/kubernetes/operator/), [OpenTelemetry Helm Charts](https://opentelemetry.io/docs/platforms/kubernetes/helm/), and [community assets for FaaS](https://opentelemetry.io/docs/platforms/faas/)

---
--- 

# Observability Backends

Observability Backends are the ones Otel implementation exports its metrics to. They sit in the cluster or other ecosystems and listen to incoming traffic, which they then identify, interpret and analyze. With these backends it is possible to monitor software's health and other inner workings using only high-level infrastracture.

---
## Jaeger

Jaeger is a distributed tracing platform released as open source by [Uber Technologies](http://uber.github.io/) in 2016 and donated to [Cloud Native Computing Foundation](https://cncf.io/) where it is a graduated project.

With Jaeger you can:

- Monitor and troubleshoot distributed workflows
- Identify performance bottlenecks
- Track down root causes
- Analyze service dependencies


### Features

- [OpenTracing](https://opentracing.io/)-inspired data model
- [OpenTelemetry](https://opentelemetry.io/) compatible
- Multiple built-in storage backends:
    - [Elasticsearch](https://www.jaegertracing.io/docs/2.20/storage/elasticsearch/) and [OpenSearch](https://www.jaegertracing.io/docs/2.20/storage/opensearch/)
    - [Cassandra](https://www.jaegertracing.io/docs/2.20/storage/cassandra/)
    - [Badger](https://www.jaegertracing.io/docs/2.20/storage/badger/) (single node, local file storage)
    - [Kafka](https://www.jaegertracing.io/docs/2.20/storage/kafka/) (as an intermediate buffer)
    - [Memory storage](https://www.jaegertracing.io/docs/2.20/storage/memory/)
- Extensibility with custom backends via [Remote Storage API](https://www.jaegertracing.io/docs/2.20/storage/#remote-storage)
- System topology / service dependencies graphs
- Adaptive sampling
- Service Performance Monitoring (SPM)
- Post-collection data processing

See [Features](https://www.jaegertracing.io/docs/2.20/features/) page for more details.

### Relationship with OpenTelemetry

The Jaeger and [OpenTelemetry](https://opentelemetry.io/) projects have different goals. OpenTelemetry aims to provide APIs and SDKs in multiple languages to allow applications to export various telemetry data out of the process, to any number of metrics and tracing backends. The Jaeger project is primarily the tracing backend that receives tracing telemetry data and provides processing, aggregation, data mining, and visualizations of that data. 

--- 
## Loki


Unlike other logging systems, Loki is built around the idea of only indexing metadata about your logs’ labels (just like Prometheus labels). Log data itself is then compressed and stored in chunks in object stores such as Amazon Simple Storage Service (S3) or Google Cloud Storage (GCS), or even locally on the filesystem.

A typical Loki-based logging stack consists of 3 components:

- **Agent** - An agent or client, for example [Grafana Alloy](https://grafana.com/docs/alloy/latest/). The agent scrapes logs, turns the logs into streams by adding labels, and pushes the streams to Loki through an HTTP API.
    
- **Loki** - The main server, responsible for ingesting and storing logs and processing queries. It can be deployed in three different configurations, for more information see [deployment modes](https://grafana.com/docs/loki/latest/get-started/deployment-modes/).
    
- **[Grafana](https://github.com/grafana/grafana)** for querying and displaying log data. You can also query logs from the command line, using [LogCLI](https://grafana.com/docs/loki/latest/query/logcli/) or using the Loki API directly.

### Loki Features

- **Scalability** - Loki is designed for scalability, and can scale from as small as running on a Raspberry Pi to ingesting petabytes a day. Loki can run as a single binary for simple setups, in [HA monolithic mode](https://grafana.com/docs/loki/latest/get-started/deployment-modes/#ha-monolithic-mode) for moderate horizontal scalability without added operational complexity, or as fine-grained microservices designed to run natively within Kubernetes for the largest, highest-scale installations.

- **Multi-tenancy** - Loki allows multiple tenants to share a single Loki instance. With multi-tenancy, the data and requests of each tenant is completely isolated from the others. Multi-tenancy is [configured](https://grafana.com/docs/loki/latest/operations/multi-tenancy/) by assigning a tenant ID in the agent.
    
- **Third-party integrations** - Several third-party agents (clients) have support for Loki, via plugins. This lets you keep your existing observability setup while also shipping logs to Loki.
    
- **Efficient storage** - Loki stores log data in highly compressed chunks. Similarly, the Loki index, because it indexes only the set of labels, is significantly smaller than other log aggregation tools. By leveraging object storage as the only data storage mechanism, Loki inherits the reliability and stability of the underlying object store. It also capitalizes on both the cost efficiency and operational simplicity of object storage over other storage mechanisms like locally attached solid state drives (SSD) and hard disk drives (HDD).  
    The compressed chunks, smaller index, and use of low-cost object storage, make Loki less expensive to operate.
    
- **LogQL, the Loki query language** - [LogQL](https://grafana.com/docs/loki/latest/query/) is the query language for Loki. Users who are already familiar with the Prometheus query language, [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/), will find LogQL familiar and flexible for generating queries against the logs. The language also facilitates the generation of metrics from log data, a powerful feature that goes well beyond log aggregation.
    
- **Alerting** - Loki includes a component called the [ruler](https://grafana.com/docs/loki/latest/alert/), which can continually evaluate queries against your logs, and perform an action based on the result. This allows you to monitor your logs for anomalies or events. Loki integrates with [Prometheus Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/), or the [alert manager](https://grafana.com/docs/grafana/latest/alerting/) within Grafana.
    
- **Grafana integration** - Loki integrates with Grafana, Mimir, and Tempo, providing a complete observability stack, and seamless correlation between logs, metrics and traces.