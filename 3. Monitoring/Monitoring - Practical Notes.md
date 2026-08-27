
## Application Instrumentation

First, I needed to expose application metrics and generate trace data. Since I've built my applications (`simple-web-app` and `simple-app-sender`) using Spring Boot with Java, I utilized Micrometer and the Prometheus client to expose the metrics.

  

By adding the appropriate dependencies and configurations, I was able to expose the `/metrics` endpoint.

These are a few links I used as reference during the development for the Java metrics:
- [https://www.baeldung.com/java-prometheus-client](https://www.baeldung.com/java-prometheus-client)
- [https://www.baeldung.com/micrometer](https://www.baeldung.com/micrometer)
For traces, I instrumented the HTTP handlers to generate trace data and export it via OTLP directly to my OpenTelemetry Collector, which would then handle routing it to Jaeger and Prometheus.

  

## Infrastructure Setup

Since I want my images and infrastructure to be managed natively in my Kubernetes cluster, I moved away from manual deployments and fully configured Argo CD using Helm.

Here are the commands I used to set up Argo CD in the cluster using Helm, extract the values, and create the necessary secrets for GitLab:
Bash

```
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --set server.service.type=LoadBalancer

helm show values argo/argo-cd > values-argocd.yaml

helm upgrade argocd argo/argo-cd \
  --namespace argocd \
  -f infrastructure/argocd/values-argocd.yaml

kubectl create secret docker-registry gitlab-registry-secret \
  --docker-server=registry.gitlab.com \
  --docker-username=<YOUR_GITLAB_USERNAME> \
  --docker-password=<YOUR_REGISTRY_TOKEN> \
  -n bootcamp-v1
```

### Deploying the OpenTelemetry Collector

To get traces and metrics flowing, I deployed the OpenTelemetry Collector via a Helm chart. This required creating a custom `ConfigMap` to define the pipelines.

Initially, I ran into an issue where Prometheus was reporting the target as "UP" but returning no metrics. I realized Prometheus was scraping the default internal telemetry port (`8888`) instead of the custom exporter port I set up (`8889`). I also realized that I needed both the `spanmetrics` connector (for node data) and the `servicegraph` connector (for edge/line data) to visualize traffic properly later on.

  

Here is the final, working `ConfigMap` I applied to route OTLP traces to Jaeger and expose metrics on port 8889 for Prometheus:

  

YAML

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: bootcamp-v1
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    
    connectors:
      spanmetrics:
        histogram:
          explicit:
            buckets: [2ms, 10ms, 100ms, 500ms, 1s, 10s]
      servicegraph:
        dimensions:
          - http.method

    exporters:
      prometheus:
        endpoint: "0.0.0.0:8889"
      otlp/jaeger:
        endpoint: "jaeger-service-jaeger-ui:4317"
        tls:
          insecure: true

    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [otlp/jaeger, spanmetrics, servicegraph] 
        metrics:
          receivers: [spanmetrics, servicegraph]
          exporters: [prometheus]
```

_Note: Because Kubernetes doesn't automatically restart pods when a ConfigMap updates, I had to ensure Argo CD synced the changes and then manually run `kubectl rollout restart deployment otel-collector -n bootcamp-v1` to force the collector to read the new pipeline._

Afterwards, I've reconfigured both of my applications to export their metrics directly to the Otel Collector. I've had to install it on them using Dockerfile:

An example for `simple_web_app`:
```
FROM eclipse-temurin:17-jdk-alpine  
WORKDIR /usr/local/app  
  
RUN apk update && \  
    apk add redis && \  
    redis-server --version && \  
    apk cache -v sync  
  
RUN addgroup -S appgroup && adduser -S appuser -G appgroup  
  
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar /opentelemetry-javaagent.jar  
RUN chmod 644 /opentelemetry-javaagent.jar  
  
COPY --chown=appuser:appgroup target/simple_web_app-1.0-SNAPSHOT.jar ./app.jar  
  
ENV JAVA_TOOL_OPTIONS="-javaagent:/opentelemetry-javaagent.jar"  
EXPOSE 7070  
  
USER appuser  
  
ENTRYPOINT ["java", "-jar", "./app.jar", "--server.port=7070"]
```
  

### Prometheus

In order for Prometheus to ingest Otel Collector's traffic it exports we must add it as a job to our **configMap**:

```
      - job_name: 'otel-collector-spanmetrics'
        static_configs:
          - targets: ['otel-collector-otel-chart-service.bootcamp-v1.svc.cluster.local:8889']
```

I've added this job and pointed the target to be the service that exposes otel to the cluster. Using its internal cluster address is a lot better and does not require a nodePort.

### Loki

I've installed Monolithic Loki by following their [guide](https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/), I've done no changes to its values as it worked great without touching any of its variables:


### Jaeger

Installing Jaeger v2 was the same, I've followed the [guide](https://github.com/jaegertracing/helm-charts/tree/main/charts/jaeger) on its github page and afterwards I started tweaking it. Firstly, I've exported its values into a local file, and afterwards I've added prometheus annotations so it could be scraped by it.  I've added a Deployment with Service in which I exposed Jaeger's UI on a nodePort to access it via a browser. 

```
jaeger:
  service:
    annotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "8888"
      prometheus.io/path: "/metrics"
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8888"
    prometheus.io/path: "/metrics"

userconfig:
  service:
    extensions: [jaeger_storage, jaeger_query, remote_sampling, healthcheckv2, pprof]
    pipelines:
      traces:
        receivers: [otlp, jaeger, zipkin]
        processors: [batch, adaptive_sampling]
        exporters: [jaeger_storage_exporter]
    telemetry:
      resource:
        service.name: jaeger
      metrics:
        level: detailed
        readers:
          - pull:
              exporter:
                prometheus:
                  host: 0.0.0.0
                  port: 8888
      logs:
        level: debug

  extensions:
    healthcheckv2:
      endpoint: 0.0.0.0:13133
      http:
        endpoint: 0.0.0.0:13133

    pprof:
      endpoint: 0.0.0.0:1777

    jaeger_query:
      storage:
        traces: some_store
        traces_archive: another_store
      ai:
        agent_url: ws://localhost:16688
      max_clock_skew_adjust: 0s

    jaeger_storage:
      backends:
        some_store:
          memory:
            max_traces: 100000
        another_store:
          memory:
            max_traces: 100000

    remote_sampling:
      adaptive:
        sampling_store: some_store
        initial_sampling_probability: 0.1
      http:
      grpc:
        




  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
    jaeger:
      protocols:
        grpc:
          endpoint: 0.0.0.0:14250
        thrift_http:
          endpoint: 0.0.0.0:14268
        thrift_compact:
          endpoint: 0.0.0.0:6831
        thrift_binary:
          endpoint: 0.0.0.0:6832
    zipkin:

  processors:
    batch:
    adaptive_sampling:

  exporters:
    jaeger_storage_exporter:
      trace_storage: some_store

serviceJaeger:
  type: NodePort
  selector:
    app.kubernetes.io/instance: jaeger
    app.kubernetes.io/name: jaeger
  ports:
    - name: jaeger-ui
      nodePort: 30016
      port: 16686
      protocol: TCP 
      targetPort: 16686
    - name: otlp-grpc
      nodePort: 30017
      port: 4317
      protocol: TCP 
      targetPort: 4317
    - name: otlp-http
      nodePort: 30018
      port: 4318
      protocol: TCP 
      targetPort: 4318
   
   
  
```


### Grafana

All of this will be visualized by Grafana which I installed using this [guide](https://grafana.com/docs/grafana/latest/setup-grafana/installation/helm/). 

## Dashboarding

Once the data was flowing into Prometheus, Loki, and Jaeger, I logged into Grafana to create a unified dashboard. The most challenging part of this phase was building the topology map using Grafana's Node Graph panel to visualize the applications communicating.

  

### The Node Graph Struggle

Grafana's Node Graph engine is incredibly strict regarding data formats. During development, I encountered several "No data" errors or screens with floating circles and no connecting lines.

To bypass the UI limitations entirely, I dropped the "Nodes" query altogether. By supplying a single, perfectly formatted "Edges" query, Grafana automatically generates the nodes for you.

I wrote a nested PromQL query using `label_join` and `label_replace` to force Prometheus to natively output the exact schema Grafana demands (`id`, `source`, `target`):

Code snippet

```
sum by (id, source, target) (
  label_replace(
    label_replace(
      label_join(
        traces_service_graph_request_total{client!="", server!=""}, 
        "id", "-", "client", "server"
      ), 
      "source", "$1", "client", "(.*)"
    ), 
    "target", "$1", "server", "(.*)"
  )
)
```

Inside Grafana's **Transform data** tab, all I had to do was use the **Organize fields** transformation to:
- Hide the `Time` column.
- Rename `Value` to `mainStat`.
- Leave `id`, `source`, and `target` exactly as they were.

Once the "Table view" toggle was turned off, the map perfectly rendered the circles for `simple-app-sender` and `simple-web-app`, complete with directional arrows proving the HTTP requests were flowing correctly across my cluster. I've used this to visualize Spans across both applications.