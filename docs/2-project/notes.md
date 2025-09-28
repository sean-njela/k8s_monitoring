# General notes before we begin

There are 3 Ways to Monitor Your App in Kubernetes

### 1. **Basic container & node monitoring (built-in)**

Kubernetes already integrates with:

* **kubelet/cAdvisor** → container & pod-level resource usage (CPU, memory, network, filesystem).
* **Node Exporter** → node-level OS metrics.

If you deploy Prometheus (via Helm or the Operator), these metrics are automatically discovered and scraped — no extra setup required.



### 2. **App-specific exporters (most common in K8s)**

Many apps expose metrics directly, or you deploy an **exporter sidecar or Service**:

* **Nginx** → [nginx-prometheus-exporter](https://github.com/nginxinc/nginx-prometheus-exporter)
* **Postgres** → [postgres\_exporter](https://github.com/prometheus-community/postgres_exporter)
* **MySQL** → [mysqld\_exporter](https://github.com/prometheus/mysqld_exporter)
* **Redis** → [redis\_exporter](https://github.com/oliver006/redis_exporter)

Example: **Nginx Exporter Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      - name: nginx-exporter
        image: nginx/nginx-prometheus-exporter:0.11.0
        args:
          - -nginx.scrape-uri=http://127.0.0.1:80/stub_status
        ports:
        - containerPort: 9113

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-exporter
spec:
  selector:
    matchLabels:
      app: nginx
  endpoints:
  - port: 9113
```

The `ServiceMonitor` CRD (from the Prometheus Operator) ensures Prometheus automatically discovers and scrapes the exporter.

To list all available CRDs use:

```bash
kubectl get crd
```
![Output](../assets/servicemonitor.png)



### 3. **Custom metrics (for apps you write)**

If you’re building your own application, you can instrument it with OpenTelemetry or a Prometheus client library:

* Go → [prometheus/client\_golang](https://github.com/prometheus/client_golang)
* Python → [prometheus\_client](https://github.com/prometheus/client_python)
* Node.js → [prom-client](https://github.com/siimon/prom-client)

Example: Python Flask app exposing `/metrics`

```python
from flask import Flask
from prometheus_client import Counter, generate_latest

app = Flask(__name__)
requests_total = Counter('app_requests_total', 'Total requests')

@app.route('/')
def hello():
    requests_total.inc()
    return "Hello, Kubernetes!"

@app.route('/metrics')
def metrics():
    return generate_latest(), 200, {'Content-Type': 'text/plain'}
```

Deploy the app with a Kubernetes `Deployment` and expose `/metrics`. Then define a `ServiceMonitor` to let Prometheus scrape it.

* **Infra metrics** → kubelet/cAdvisor + Node Exporter.
* **App metrics** → Exporters (Nginx, DBs, etc.) or custom instrumentation.
* Prometheus scrapes everything; Grafana dashboards visualize it.



### What OpenTelemetry Adds

* **Instrumentation SDKs** → for Go, Python, Java, Node.js, .NET, etc.
* Collects **application metrics**, **traces**, and optionally **logs**.
* Works via an **OpenTelemetry Collector** deployed as:

  * A **sidecar** (per pod)
  * A **DaemonSet** (per node)
  * A **Deployment** (central collector)

Backends supported:

* Prometheus (metrics)
* Jaeger / Tempo (traces)
* Loki / ELK (logs)
* Grafana Cloud or OTLP-compatible vendors



### 🛠 How It Works in Kubernetes

Right now you likely have:

* **Prometheus + Grafana** → metrics + visualization
* **kubelet/cAdvisor + Node Exporter** → infra metrics

If you add **OpenTelemetry for your app**:

1. Instrument your code with the OTel SDK.
2. Deploy an **OpenTelemetry Collector** (Deployment or DaemonSet).
3. Configure it to export metrics in Prometheus format (scrapable endpoint).



### Example: OTel Collector in Kubernetes

Deploy the collector:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.95.0
        args: ["--config=/etc/otel/otel-collector-config.yaml"]
        volumeMounts:
        - name: config
          mountPath: /etc/otel
      volumes:
      - name: config
        configMap:
          name: otel-collector-config

apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:

    exporters:
      prometheus:
        endpoint: "0.0.0.0:9464"

    service:
      pipelines:
        metrics:
          receivers: [otlp]
          exporters: [prometheus]

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: otel-collector
spec:
  selector:
    matchLabels:
      app: otel-collector
  endpoints:
  - port: 9464
```

Now:

* Your app sends OTLP data to the Collector (`4317/4318`).
* The Collector exposes Prometheus metrics on `:9464`.
* Prometheus scrapes them via the `ServiceMonitor`.



### Why OTel is a Good Idea in Kubernetes

* **Cloud-native standard** → portable across Prometheus, Grafana, Jaeger, Tempo, etc.
* **Future-proof** → works with managed observability platforms (Datadog, New Relic, Grafana Cloud, etc.).
* **Full observability** → unifies **metrics, traces, and logs**, not just CPU/memory.



## Adding any app to the stack

How to instrument *any app* for this stack (kube-prometheus-stack, Loki, Alertmanager, Grafana). This is the **app-side view** only.

### 1. Logs → Loki (via Promtail)

* **Best practice**: apps write logs to `stdout/stderr`.

  * Kubernetes writes container logs to `/var/log/containers/*.log`.
  * Promtail DaemonSet (from Loki Helm chart) already tails these files.

* **Labels**: Promtail auto-labels with `namespace`, `pod`, `container`, and `app_kubernetes_io/name`.

* **App requirements**:

  * Use structured logging (JSON preferred).
  * Include severity, service name, request ID, etc. as fields. Promtail can parse JSON and promote fields to labels.

* **Trade-off**:

  * Too many labels = high cardinality → Loki performance issues.
  * Use pipeline stages (`drop`, `replace`, `json`) to balance.



### 2. Metrics → Prometheus

* kube-prometheus-stack deploys Prometheus Operator with **ServiceMonitor/PodMonitor CRDs**.
* **Expose `/metrics` endpoint** in the app:

  * Prometheus client libraries exist for Go, Python, Java, Node.js, etc.
  * Instrument key business metrics (requests, errors, durations).

* **App requirements**:

  * Export metrics in Prometheus text format.
  * Add a Kubernetes `Service` with proper label selectors.
  * Create a `ServiceMonitor` that points to the Service.

* **Alerting**:

  * Prometheus alerts are routed to Alertmanager, which you have configured.
  * Ensure metric names and labels are consistent with your alert rules.



### 3. Traces → OpenTelemetry (OTel)

* Use the **OpenTelemetry SDK** in your app to generate spans.
* Export traces to the **OTel Collector** running in your cluster.
* Collector pipelines:

  * Metrics → Prometheus remote-write (kube-prometheus-stack).
  * Logs → Loki (via OTLP → Loki exporter, or side-car Promtail).
  * Traces → Tempo or Jaeger (Grafana Tempo is the common pair with Loki).

* **App requirements**:

  * Add OTel SDK instrumentation for HTTP, gRPC, DB calls.
  * Configure OTLP exporter to send to Collector endpoint.



### 4. Dashboards → Grafana

* Grafana in kube-prometheus-stack provisions Prometheus and Loki as datasources.
* For app dashboards:

  * Metrics: build panels from `/metrics` data.
  * Logs: query `{app="myapp"}` in Explore or add log panels.
  * Traces: if Tempo integrated, add trace panels.

* **Repeatability**:

  * Store dashboards as JSON in Git.
  * Provision them via ConfigMap + sidecar (`grafana_dashboard=1`) or GitOps pipeline.



### 5. Alerting

* Already handled via Alertmanager.
* App instrumentation must expose metrics that can be turned into SLO alerts, e.g.:

  * `http_requests_total` (rate of requests).
  * `http_request_duration_seconds` (latency).
  * `http_requests_errors_total` (error rate).

* Alerts are defined as Prometheus rules, not app-side.



### 6. DevOps repeatability checklist

For *any app*:

1. **Logging**:

   * stdout/stderr JSON logs.
   * Use structured fields: `level`, `service`, `trace_id`.
   * Confirm Promtail pipeline parses correctly.

2. **Metrics**:

   * Add Prometheus client library.
   * Expose `/metrics`.
   * Apply Service + ServiceMonitor.

3. **Tracing** (optional but recommended):

   * Add OTel SDK.
   * Configure OTLP export → OTel Collector.

4. **Dashboards**:

   * Provision JSON dashboards via ConfigMaps or `grafana.dashboards` Helm values.
   * Tie logs, metrics, traces together using consistent labels (e.g. `service_name`).

5. **Alerts**:

   * Define PrometheusRules based on app metrics.
   * Ensure routes in Alertmanager are set to notify the right channel/team.



This makes any app pluggable into your stack: Prometheus Operator scrapes metrics, Promtail collects logs, OTel provides traces, Grafana visualises, Alertmanager handles alerts.

Here is a **golden template** you can drop into any microservice repo to instrument it for your stack (kube-prometheus-stack, Loki, Grafana, Alertmanager, OpenTelemetry).

It is language-agnostic in structure, then broken into logging, metrics, and tracing.



## 📄 Golden Template for App Instrumentation

### 1. Logging → Loki

**App code**:

* Always log to `stdout/stderr`.
* Use JSON format.
* Include consistent fields:

```json
{
  "timestamp": "2025-09-27T20:15:32Z",
  "level": "info",
  "service_name": "myapp",
  "trace_id": "abc123",
  "span_id": "def456",
  "message": "user login success",
  "http_method": "POST",
  "http_status": 200
}
```

**Why**:

* Promtail DaemonSet already tails `/var/log/containers/*.log`.
* JSON fields can be promoted to Loki labels (`http_status`, `level`, `service_name`).

**Promtail pipeline (values.yaml snippet)**:

```yaml
extraScrapeConfigs: |
  - job_name: kubernetes-pods-apps
    pipeline_stages:
      - json:
          expressions:
            level: level
            service_name: service_name
            http_status: http_status
      - labels:
          level:
          service_name:
          http_status:
```



### 2. Metrics → Prometheus

**App code**:

* Import Prometheus client library.
* Expose `/metrics` endpoint.

**Example (Python / FastAPI)**:

```python
from prometheus_client import Counter, Histogram, generate_latest
from fastapi import FastAPI, Response, Request
import time

app = FastAPI()

REQ_COUNT = Counter("http_requests_total", "Total requests", ["method", "endpoint", "http_status"])
REQ_LATENCY = Histogram("http_request_duration_seconds", "Request latency", ["method", "endpoint"])

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    REQ_COUNT.labels(request.method, request.url.path, response.status_code).inc()
    REQ_LATENCY.labels(request.method, request.url.path).observe(duration)
    return response

@app.get("/metrics")
async def metrics():
    return Response(content=generate_latest(), media_type="text/plain")
```

**Kubernetes Service**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  ports:
    - name: http
      port: 80
      targetPort: 8080
  selector:
    app: myapp
```

**ServiceMonitor**:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```



### 3. Tracing → OpenTelemetry

**App code (Python / FastAPI example)**:

```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Configure tracer
provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4318/v1/traces"))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

# Instrument FastAPI
FastAPIInstrumentor().instrument_app(app)
```

**Collector config (values.yaml for opentelemetry-collector)**:

```yaml
config:
  receivers:
    otlp:
      protocols:
        http:
        grpc:

  exporters:
    loki:
      endpoint: http://loki-gateway.k8s-monitoring-ns.svc.cluster.local/loki/api/v1/push
    prometheus:
      endpoint: "0.0.0.0:8889"
    tempo:
      endpoint: http://tempo:4317
      insecure: true

  service:
    pipelines:
      traces:
        receivers: [otlp]
        exporters: [tempo]
      metrics:
        receivers: [otlp]
        exporters: [prometheus]
      logs:
        receivers: [otlp]
        exporters: [loki]
```

### 4. Dashboards → Grafana

* Store dashboards as JSON in Git.
* Provision using ConfigMap with `grafana_dashboard: "1"` label.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-dashboard
  labels:
    grafana_dashboard: "1"
data:
  myapp-dashboard.json: |
    { "title": "MyApp Dashboard", "panels": [...] }
```



### 5. Alerts → Alertmanager

* Write PrometheusRule CRs for your app metrics.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-rules
spec:
  groups:
    - name: myapp.rules
      rules:
        - alert: HighErrorRate
          expr: rate(http_requests_total{http_status=~"5.."}[5m]) > 0.05
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "High error rate for myapp"
            description: "More than 5% of requests are failing"
```

### Checklist for any app

* [ ] Logs in JSON to `stdout`.
* [ ] Prometheus client library, `/metrics` endpoint.
* [ ] OTel SDK exporting to Collector.
* [ ] Service + ServiceMonitor in cluster.
* [ ] Dashboard JSON under Git.
* [ ] PrometheusRule alerts defined.

Later you clould package this into a **starter Helm chart skeleton for apps** so you can just scaffold new services with logging, metrics, and tracing prewired?



