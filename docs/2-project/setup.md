# Our Setup

By default kubernetes doesn't automatically ship cluster level metrics (deployments, pods, nodes, jobs) so we will have to deploy a **kube-state-metrics** container, which Exposes the **state of Kubernetes objects** (deployments, pods, nodes, jobs).

Every node should run a **node_exporter** as a **daemonSet** (pod that runs on every node in the cluster), such taht we won't need to remember to deploy one whenever we add a node to the cluster.

We will deploy **Prometheus** using **HELM**(a package manager for K8s) to make the process repeatable. The **Kube-Prometheus-Stack** HELM chart will be used. It makes use of the **Prometheus Operator**. A K8s operator is an application-specific controller that extends the K8s API to create/configure/manage instances of complex applications (the complete lifecycle for Prometheus in our case). 

The prometheus operator has several custom resources to aid the deployment and management of a Prometheues instance. (See Image below)

![Prometheus Operator](../assets/prometheus-operator.png)

This makes life a lot easier as we will have a higher level abstraction of the underlying resources.

### Add the Chart

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack
#                 ^
#                 |
#    this name can be anything
```

We can edit the default configuratioin of the chart by editing the values:

```bash
helm show values prometheus-community/kube-prometheus-stack > prometheus-helm-values.yaml
# and after editing a value:
helm install prometheus prometheus-community/kube-prometheus-stack -f prometheus-helm-values.yaml
```

By default prometheus deploys the following:

```bash

NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          14m
pod/prometheus-grafana-674cf8cb44-l8479                      3/3     Running   0          14m
pod/prometheus-kube-prometheus-operator-6694cc948f-7xlh8     1/1     Running   0          14m
pod/prometheus-kube-state-metrics-7c5fb9d798-82n44           1/1     Running   0          14m
pod/prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          14m
pod/prometheus-prometheus-node-exporter-zck8r                1/1     Running   0          14m

NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                     ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   14m
service/prometheus-grafana                        ClusterIP   10.96.2.11      <none>        80/TCP                       14m
service/prometheus-kube-prometheus-alertmanager   ClusterIP   10.96.226.179   <none>        9093/TCP,8080/TCP            14m
service/prometheus-kube-prometheus-operator       ClusterIP   10.96.72.246    <none>        443/TCP                      14m
service/prometheus-kube-prometheus-prometheus     ClusterIP   10.96.37.230    <none>        9090/TCP,8080/TCP            14m
service/prometheus-kube-state-metrics             ClusterIP   10.96.112.1     <none>        8080/TCP                     14m
service/prometheus-operated                       ClusterIP   None            <none>        9090/TCP                     14m
service/prometheus-prometheus-node-exporter       ClusterIP   10.96.92.210    <none>        9100/TCP                     14m

NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/prometheus-prometheus-node-exporter   1         1         1       1            1           kubernetes.io/os=linux   14m

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-grafana                    1/1     1            1           14m
deployment.apps/prometheus-kube-prometheus-operator   1/1     1            1           14m
deployment.apps/prometheus-kube-state-metrics         1/1     1            1           14m

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-grafana-674cf8cb44                    1         1         1       14m
replicaset.apps/prometheus-kube-prometheus-operator-6694cc948f   1         1         1       14m
replicaset.apps/prometheus-kube-state-metrics-7c5fb9d798         1         1         1       14m

NAME                                                                    READY   AGE
statefulset.apps/alertmanager-prometheus-kube-prometheus-alertmanager   1/1     14m
statefulset.apps/prometheus-prometheus-kube-prometheus-prometheus       1/1     14m
```

Take note of a few things:

* All Services are of type **ClusterIP**.
* `deployment.apps/prometheus-grafana` - The chart also spins up a preconfigured **Grafana** instance.
* `deployment.apps/prometheus-kube-prometheus-operator` is the Prometheus operator we talked about.
* `deployment.apps/prometheus-kube-state-metrics` exposes object metrics (nodes, pods, deployments, jobs).
* `daemonset.apps/prometheus-prometheus-node-exporter` makes sure we have a node exporter on all modes.
* `pod/prometheus-prometheus-node-exporter-zck8r ` will be one per every node (we only have one node but if we had another we would have another e.g `pod/prometheus-prometheus-node-exporter-xntb2` )


### Connecting to Prometheus

We could setup an **Ingress** to access the prometheus service, but in this demo we will make use of port forwarding.

`kubectl port-forward` works for **both Pods and Services**

* **Pod:**
  You can forward directly to a Pod by its name:

  ```bash
  kubectl port-forward pod/my-pod 8080:80
  ```

  This connects your local port `8080` to port `80` inside the Pod.

* **Service:**
  You can also forward to a Service:

  ```bash
  kubectl port-forward svc/my-service 8080:80
  ```

  In this case, `kubectl` resolves the Service to one of its backing Pods and forwards traffic to it. (It doesn’t load-balance across Pods — just picks one.)

So:

* Pod = direct connection to that Pod.
* Service = convenience, so you don’t need to look up Pod names.



So we run:

```bash
kubectl get service
# then port forward the Prometheus service
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090
```

### Deploying the application instance

We placed our `demo_app_deployment.yaml` and `demo_app_svc.yaml` in the `k8s` folder then ran:

```bash
kubectl create -f k8s/ # creates 2 pods and a service
# Followed up with:
task status
```

Nice, the app is now running in our cluster.

We will have to setup additional prometheus configs so that the app can be scrapped by prometheus.

The recommended way to set up monitoring in this stack, is to make use of **Service Monitors**.

### Setup App Metrics Scrapping

A **Service Monitor** defines a set of targets for prometheus to monitor and scrape.

That way we avoid touching the default configs directly, instead we will be using the CRD to append new targets for Prometheus to scrape.

A service monitor points to a **Service**.

![Service Monitor Yaml Configuration](../assets/servicemonitor-2.png) ![continuation](../assets/servicemonitor-3.png) ![continuation](../assets/servicemonitor-4.png) ![continuation](../assets/servicemonitor-5.png)

The `service-monitor.yaml` file is added alongside the app deployment files in the `k8s/` folder.

```bash
task apply-app
# then
kubectl get servicemonitor # to confirm if our service monitor is deployed
# output:
NAME                                                 AGE
prometheus-demo-app-svc-monitor                      2m13s
```

Also confirm in the Prometheus web app at `/targets`:

![Prometheus Web UI showing Service Monitor in UP state](../assets/servicemonitor-6.png)

You can also query `{job="prometheus-demo-app-svc"}`:

![Prometheus Web UI showing query results](../assets/servicemonitor-7.png)

### Adding Rules

Prometheus also has a CRD for handling rules named `prometheusrule`.


![Screenshot of PrometheusRules Yaml file](../assets/prometheusrules-1.png)

![Screenshot of PrometheusRules Release matchlabel](../assets/prometheusrules-2.png)

The `prometheusrules.yaml` file is added alongside the app deployment files in the `k8s/` folder.

```bash
task apply-app
# then
kubectl get prometheusrule # to confirm if our service monitor is deployed
# output:
NAME                                                              AGE
prometheus-demo-app-rules                                         60s
```

Also confirm in the Prometheus web app at `/rules`:

![Prometheus Web UI showing /rules section](../assets/prometheusrules-3.png)

### Adding Alert Manager Rules

First we need to note an important point when it comes to specifying custom values.yaml

Helm merges your custom `values.yaml` with the chart’s default values.  
It uses a **deep merge**, so you must preserve the full key path when overriding values.  

- If you only define part of the path, Helm will ignore it.  
- You must follow the parent hierarchy exactly as defined in the chart’s `values.yaml`.  
- Always check the chart’s defaults with:  

```bash
  helm show values <chart>
```

If you try to set this at the root:

```yaml
alertmanagerConfigSelector:
  matchLabels:
    resource: prometheus
```

…it will fail, because `alertmanagerConfigSelector` is **not** a root key.

In the **kube-prometheus-stack** chart, the correct path is under `prometheus.prometheusSpec`:

```yaml
prometheus:
  prometheusSpec:
    alertmanagerConfigSelector:
      matchLabels:
        resource: prometheus
```

* Helm only overrides keys at the correct hierarchy.
* Wrong placement = no effect.
* Always trace the full path from the chart’s default `values.yaml`.

### Continuing with our setup

Prometheus also has a CRD for handling alert manager rules named `AlertmangerConfig`.

![Screenshot of AlertmanagerConfig Yaml file](../assets/alertmanagerconfig-1.png)

![Screenshot of differences in the Alertmanager Yaml config](../assets/alertmanagerconfig-3.png)

![Screenshot of AlertmangerConfig Selector](../assets/alertmanagerconfig-2.png)

By default this selector is not in prometheus' `values.yaml` so we have to add it.

```bash
task prom-helm-values
```

Then search for `alertmanagerConfigSelector`, copy the snippet and save in a custom values file as `k8s/prometheus-helm-values.yaml`:

```yaml
alertmanagerConfigSelector:
  matchLabels:
    resource: prometheus
```

Then apply:

```bash
task helm-upgrade-prom
```

![Screenshot showing the commands in action](../assets/alertmanagerconfig-4.png)

The `alertmanager-config.yaml` file is added alongside the app deployment files in the `k8s/` folder.

```bash
task apply-app

# then
kubectl get alertmanagers.monitoring.coreos.com -o yaml # to confirm if our service monitor is deployed

# output:
# serach for:
...
    alertmanagerConfigSelector:
      matchLabels:
        resource: prometheus
...

# and
kubectl get alertmanagerconfig

# output:
NAME                               AGE
prometheus-demo-app-alert-config   7m59s

# then:
task port-fwd-alertm
# and visit /localhost:9093
```

!!!tip 
    When port fowarding, the port number is always the one exposed by the service or Pod or you can specify as `HOST:SVC/POD`

Also confirm in the Prometheus web app at `/rules`:

![Prometheus Web UI showing /rules section](../assets/alertmanager.png)


### Grafana Dashboards

The Helm chart already has pre-configured dashboards.

We port forward the Grafana UI:

```bash
task port-fwd-graf
```

!!!tip 
    When port fowarding, the port number is always the one exposed by the service or Pod or you can specify as `HOST:SVC/POD`

We then add other dashboards namely: `15661`.

![Screenshot of the Dashboard](../assets/dashboard.png)

We can also add instrumentation for our app and add to prometheus scrape config and also add relevant dashboards.

This confirms our setup is complete !

```bash
> task status
task: [status] kubectl get all -n k8s-monitoring-ns
[status] NAME                                                         READY   STATUS    RESTARTS      AGE
[status] pod/alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   2 (58m ago)   116m
[status] pod/prometheus-demo-app-794dc7c997-bz9fp                     1/1     Running   1 (58m ago)   107m
[status] pod/prometheus-demo-app-794dc7c997-g4275                     1/1     Running   1 (58m ago)   107m
[status] pod/prometheus-grafana-674cf8cb44-v8vlb                      3/3     Running   3 (58m ago)   117m
[status] pod/prometheus-kube-prometheus-operator-7984bfd549-g9gnx     1/1     Running   2 (57m ago)   117m
[status] pod/prometheus-kube-state-metrics-7c5fb9d798-xwdpq           1/1     Running   2 (57m ago)   117m
[status] pod/prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   2 (58m ago)   116m
[status] pod/prometheus-prometheus-node-exporter-scsd8                1/1     Running   1 (58m ago)   117m
[status]
[status] NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
[status] service/alertmanager-operated                     ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   116m
[status] service/prometheus-demo-app-svc                   ClusterIP   10.96.139.140   <none>        3000/TCP                     107m
[status] service/prometheus-grafana                        ClusterIP   10.96.199.255   <none>        80/TCP                       117m
[status] service/prometheus-kube-prometheus-alertmanager   ClusterIP   10.96.210.241   <none>        9093/TCP,8080/TCP            117m
[status] service/prometheus-kube-prometheus-operator       ClusterIP   10.96.75.79     <none>        443/TCP                      117m
[status] service/prometheus-kube-prometheus-prometheus     ClusterIP   10.96.219.164   <none>        9090/TCP,8080/TCP            117m
[status] service/prometheus-kube-state-metrics             ClusterIP   10.96.12.61     <none>        8080/TCP                     117m
[status] service/prometheus-operated                       ClusterIP   None            <none>        9090/TCP                     116m
[status] service/prometheus-prometheus-node-exporter       ClusterIP   10.96.96.156    <none>        9100/TCP                     117m
[status]
[status] NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
[status] daemonset.apps/prometheus-prometheus-node-exporter   1         1         1       1            1           kubernetes.io/os=linux   117m
[status]
[status] NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
[status] deployment.apps/prometheus-demo-app                   2/2     2            2           107m
[status] deployment.apps/prometheus-grafana                    1/1     1            1           117m
[status] deployment.apps/prometheus-kube-prometheus-operator   1/1     1            1           117m
[status] deployment.apps/prometheus-kube-state-metrics         1/1     1            1           117m
[status]
[status] NAME                                                             DESIRED   CURRENT   READY   AGE
[status] replicaset.apps/prometheus-demo-app-794dc7c997                   2         2         2       107m
[status] replicaset.apps/prometheus-grafana-674cf8cb44                    1         1         1       117m
[status] replicaset.apps/prometheus-kube-prometheus-operator-7984bfd549   1         1         1       117m
[status] replicaset.apps/prometheus-kube-state-metrics-7c5fb9d798         1         1         1       117m
[status]
[status] NAME                                                                    READY   AGE
[status] statefulset.apps/alertmanager-prometheus-kube-prometheus-alertmanager   1/1     116m
[status] statefulset.apps/prometheus-prometheus-kube-prometheus-prometheus       1/1     116m
```

### Setting up AlertManager with Slack

First head to [api.slack.com/apps](https://api.slack.com/apps)

Add a new app named `alert-manager` and add it to a workspace.

We already have a workspace named [devopssean](https://app.slack.com/client/T096B2LBJKY). 

in the features, enable an **Incoming Webhook** and select the channel to post to. In our case we have the [alert-manager channel.](https://app.slack.com/client/T096B2LBJKY/C09H1M75UG3)

!!! info
    Incoming webhooks are a simple way to post messages from external sources into Slack. They make use of normal HTTP requests with a JSON payload, which includes the message and a few other optional details. You can include message attachments to display richly-formatted messages.

    Adding incoming webhooks requires a bot user. If your app doesn't have a bot user, slack will add one for you.

    Each time your app is installed, a new Webhook URL will be generated.

If you want to test intergration run the curl command in the UI inside your terminal e.g:

```bash
curl -X POST -H 'Content-type: application/json' --data '{"text":"Alert Manager Curl Test!"}' <slack-channel-webhook>
```

Take note of the webhook URL.

We add that to the alertmanager-config.yaml under a slackConfig receiver, as raw string or more preferrably as a Secret.

In this demo we use a plain string secret.

To test the alert, shutdown the app instance by either deleting the whole thing or scaling the app's deployment down to 0 pods and then check the slack channel.

![Screenshot of a Slack Alert](../assets/slack.png)

### Set up Loki

We wan't Logs for our application to also be shown in Grafana dashboards.

It will make it easier to diagonise spikes and dips in resource usage metrics.

Loki is part of the **Grafana Helm chart**. 

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

We then save a custom values to the `k8s/values` directory.

We only specify the sections we need to change.

* We chose `deployment: SingleBinary` because it deploys every single component that loki needs as a single binary. This deploymentonly allows a loki to process a few gigabytes of logs a day whilst others result in Terrabytes albeit with a much more complex deployment. The deployyment mode should always match the environment that you the cluster operates.
* Write, read and backend replicas are set to 0 as we only have a single deployment
* Storage is set to filesystem instead of a cloud storage option.
* Disabled caching and helm testing

```bash
kubens k8s-monitoring-ns
helm install loki grafana/loki -f k8s/values/graf-helm-values.yaml
```

!!! info
    During  `helm install`, it's important to always state the version of helm chart that you are using by making use of the `--version` flag. A repeatable way is to use a tool like `Terraform`

These are the current versions of the charts we used in this tutorial:

```bash
NAME           CHART                            APP VERSION
loki           loki-6.41.1                      3.5.5
prometheus     kube-prometheus-stack-77.11.1    v0.85.0
```

We then include a grafana section in the prometheus values file in the `k8s/values` directory, to include an additional data source named loki.

We apply the changes.

```bash
task helm-upgrade-all
```

Inside the grafana UI we head to the data sources section to confirm that loki is there.

![Screenshot of Grafana Datasources](../assets/grafana-datasources.png)

### Generating Logs

We can make use of a fake log generator container `mingrammer/flog`.

In the repository we also have a Application deployment in `k8s/fake-logs.yaml` coupled with the fake log generator outputting the logs to a **PVC**, a promtail log collector mounted to the same **PVC** and a configMap to then send those over to loki.

For **testing on a single-node cluster**, this manifest is okay. It will:

* Generate fake JSON logs into a shared PVC.
* Have Promtail tail the log file, parse JSON, and push entries to Loki.
* Attach `http_method` and `http_status` as labels.
* Use an ephemeral `emptyDir` for positions, good enough for short tests.

Caveats you accept when testing this way:

* **Not production-ready**: multiple replicas or nodes will break because of the RWO PVC.
* **Data replay**: if Promtail restarts, it replays logs since positions are not persisted to durable storage.
* **Log source unrealistic**: real apps log to `stdout`, not a shared file.
* **Performance**: flog will generate constant noise. That’s fine for load testing Loki, not for validating app-level observability.

So: for verifying that your stack is wired correctly (Promtail → Loki → Grafana queries), this setup is good enough.

### Visualising in Grafana

We then import the `grafana/dashboards/fake-logs-dashboard.json` file to visualise in grafana.

![Screenshot of Fake Logs Dashboard](../assets/fake-logs-dashboard.png)

And with that done, so is our setup !

## Some Caveats

It is not production-ready by default. The stack (kube-prometheus-stack, Loki, Grafana, Alertmanager, OpenTelemetry) provides the core observability components, but production readiness depends on how it is configured, secured, and operated.

### Gaps and risks:

1. **Persistence**

      * Default Helm charts often use emptyDir volumes. Data loss occurs on pod rescheduling.
      * Loki, Prometheus, and Alertmanager require durable storage (PVCs on reliable storage classes).

2. **High availability (HA)**

      * kube-prometheus-stack deploys single replicas by default. Prometheus, Alertmanager, and Loki should run in HA mode with replication and sharding.

3. **Scaling**

      * Loki requires proper deployment mode (distributor, ingester, querier, etc.) for large clusters. Monolithic mode will not handle production workloads.
      * Prometheus requires federation or Cortex/Thanos for horizontal scalability.

4. **Security**

      * RBAC must be minimal and auditable.
      * Grafana and Alertmanager must not expose admin endpoints publicly without authentication and TLS.
      * Secrets in Helm values should not be stored in plaintext.

5. **Resource limits**

      * Default charts lack tuned CPU/memory requests and limits. These must be set based on workload.

6. **Backup and retention**

      * No backup strategy for Prometheus TSDB, Loki indexes, or Alertmanager silences.
      * Retention periods must be tuned for cost and compliance.

7. **Networking**

      * Ingress configuration requires TLS termination and WAF/firewall integration.
      * Multi-tenant isolation may be required if different teams access Grafana or Loki.

8. **Alerting pipeline**

      * Alertmanager configuration must be HA and integrated with real notification systems (PagerDuty, OpsGenie, email, Slack).
      * Silencing, inhibition, and deduplication must be tuned.

9. **OpenTelemetry integration**

      * Only provides traces/metrics/logs pipelines. Backend storage (Tempo, Jaeger, or vendor) is required.
      * Exporters must be configured per language/runtime.

10. **Operations**

      * Helm upgrades can break CRDs. Proper GitOps or CI/CD integration is required.
      * Dashboards and alerts must be version-controlled, not edited manually in Grafana.

### Minimal production-ready setup requires:

* HA Prometheus, Loki, and Alertmanager.
* Persistent storage with backups.
* TLS, RBAC, and external auth (OIDC) for Grafana.
* Proper retention, capacity planning, and scaling.
* Automated upgrades via GitOps (e.g., ArgoCD, Flux).
* Centralised alert routing with redundancy.
* Integration with a tracing backend (Tempo or Jaeger) if OpenTelemetry is used fully.

## Using **Grafana Cloud**


### Effect on our proposed stack

1. **Prometheus / kube-prometheus-stack**

      * You no longer run Prometheus in-cluster for long-term storage.
      * Instead, you deploy the **Grafana Agent** (or Prometheus remote-write) in your cluster. It scrapes Kubernetes metrics and forwards them to Grafana Cloud’s Mimir backend.
      * kube-prometheus-stack is unnecessary except if you still want local alerting or dashboards inside the cluster.

2. **Loki**

      * You do not run Loki in-cluster unless you want a local buffer.
      * Logs are shipped with Grafana Agent, Fluent Bit, or Promtail to Grafana Cloud Loki.

3. **Grafana**

      * You do not deploy Grafana in-cluster. Dashboards are hosted in Grafana Cloud.
      * You can still federate multiple data sources (cloud Loki, cloud Prometheus, external APIs).

4. **Alertmanager**

      * You do not run Alertmanager. Alert routing is handled in Grafana Cloud’s alerting service.
      * You must re-implement silences, inhibition, and notification channels in Grafana Cloud.

5. **OpenTelemetry**

      * You deploy the OpenTelemetry Collector in your cluster.
      * The collector exports traces and metrics directly to Grafana Cloud Tempo (for traces) and Mimir (for metrics).
      * No need to deploy Tempo or Jaeger locally.



### Setup steps
1. **Sign up and create a Grafana Cloud stack**  
      - Provision an account with metrics, logs, and traces enabled.  
      - Obtain API keys (for metrics, logs, traces).  

2. **Deploy Grafana Agent in Kubernetes**  
      - Install via Helm or YAML manifests.  
      - Configure it to:  
         - Scrape kubelet, kube-state-metrics, node exporter.  
         - Remote-write to Grafana Cloud Prometheus endpoint.  
         - Forward logs via Loki endpoint.  
         - Ship traces via Tempo endpoint (optionally through OTLP).  

3. **Deploy OpenTelemetry Collector**  
      - Configure it to receive OTLP (from application SDKs).  
      - Export traces and metrics to Grafana Cloud endpoints.  

4. **Remove redundant in-cluster components**  
      - No need for Prometheus statefulset, Loki statefulset, Grafana deployment, or Alertmanager.  
      - Keep kube-state-metrics and node exporter, since they are needed for scraping.  

5. **Configure dashboards and alerting in Grafana Cloud**  
      - Import Kubernetes dashboards (pre-built).  
      - Define alert rules centrally in Grafana Cloud.  
      - Integrate notification channels (PagerDuty, Slack, Teams, email).  

6. **Secure connections**  
      - Use Grafana Cloud TLS endpoints.  
      - Store API keys as Kubernetes secrets.  
      - Apply RBAC to agents and collectors.  

### Trade-offs

**Advantages**

* No stateful services to manage in-cluster.
* Scalability and HA are handled by Grafana Cloud.
* Integrated UI for metrics, logs, traces, and alerts.
* Faster onboarding and less operational risk.

**Disadvantages**

* Vendor lock-in to Grafana Cloud APIs and billing model.
* Costs scale with data ingestion volume.
* Less control over retention and storage backends.
* Alerts require re-implementation in Grafana Cloud (cannot reuse Alertmanager config directly).
* If compliance requires on-prem retention, this may fail audits.

## Adding Sentry

### What Grafana Cloud covers

* **Infrastructure metrics** (CPU, memory, node, pod health).
* **Application metrics** (via Prometheus exporters or OpenTelemetry).
* **Logs** (via Loki).
* **Traces** (via Tempo).
* **Alerting and dashboards** across metrics, logs, traces.

This gives a wide view of system health and performance.



### What Sentry covers

* **Error monitoring and crash reporting** at application level.
* **Contextual stack traces**: full code paths, local variables, user sessions.
* **Release tracking**: errors linked to git commits and releases.
* **User impact analysis**: how many users are affected by an error.
* **Frontend and mobile coverage**: JavaScript, iOS, Android SDKs that Grafana does not provide out of the box.



### Trade-offs

* You can approximate some of Sentry’s functions with OpenTelemetry tracing + Grafana Tempo, but:

  * You will not get the same developer-friendly stack traces.
  * You will not get release/version tracking without extra work.
  * You will not get automatic issue grouping and user impact analysis.
* If you only care about infra + service health, Sentry is redundant.
* If you need application-level error insight (especially for frontend and user-facing code), Sentry remains valuable.



### Practical setup

* Keep **Grafana Cloud** (metrics/logs/traces/infra alerting).
* Use **Sentry** specifically for application exceptions and end-user error reporting.
* Link them:

  * Send Sentry issues into Grafana alerting or PagerDuty.
  * Add trace IDs in both Sentry and OpenTelemetry spans so developers can pivot between them.

Integration is done at two layers:

1. **Application layer (code instrumentation)** — Sentry SDK inside Django.
2. **Observability layer (linking Sentry to Grafana Cloud)** — connect Sentry issues with Grafana Cloud traces/logs via trace IDs.



## 1. Application layer: Django + Sentry

Install the SDK:

```bash
pip install --upgrade sentry-sdk
```

Minimal safe configuration in `settings.py`:

```python
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration
from sentry_sdk.integrations.celery import CeleryIntegration
from sentry_sdk.integrations.logging import LoggingIntegration
import os

sentry_sdk.init(
    dsn=os.environ.get("SENTRY_DSN"),  # DSN from Sentry project settings
    integrations=[
        DjangoIntegration(),
        CeleryIntegration(),  # include if Celery is used
        LoggingIntegration(  # capture logs as breadcrumbs
            level=None,       # disable default logging capture
            event_level="ERROR",  # send only errors as events
        ),
    ],
    environment=os.environ.get("APP_ENV", "production"),
    release=os.environ.get("GIT_COMMIT_SHA", "unknown"),  # track release
    traces_sample_rate=0.1,  # adjust rate for performance monitoring
    send_default_pii=False,  # protect user data unless explicitly needed
)
```

### Security and performance considerations

* `send_default_pii=False` ensures no sensitive PII is leaked.
* Control sampling via `traces_sample_rate` to manage ingestion costs.
* Always inject `release` and `environment` for issue grouping.
* Store DSN in Kubernetes Secrets, not in code.



## 2. Observability layer: Linking Sentry and Grafana Cloud

The goal is correlation. Grafana Cloud handles metrics/logs/traces; Sentry handles exceptions. To connect them:

### a. Enable distributed tracing

* Use **Sentry performance monitoring** with Django.
* If you also instrument your Django API with **OpenTelemetry**, you can **propagate trace IDs** so that a request has both:

  * A trace in Grafana Cloud Tempo.
  * An error in Sentry referencing the same trace ID.

Example middleware to inject the OTel trace ID into Sentry context:

```python
from sentry_sdk import configure_scope
from opentelemetry import trace

class TraceIdMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        tracer = trace.get_tracer(__name__)
        span = trace.get_current_span()
        trace_id = span.get_span_context().trace_id
        if trace_id:
            with configure_scope() as scope:
                scope.set_tag("otel_trace_id", format(trace_id, "032x"))
        return self.get_response(request)
```

Now an exception in Sentry will include the OTel trace ID. In Grafana Cloud you can query traces by ID and cross-link.

### b. Alerts integration

* Configure **Sentry alert rules** (e.g. “More than 50 errors in 5 minutes”).
* Forward them to the same incident pipeline as Grafana Cloud (Slack, PagerDuty, OpsGenie).
* Result: both infra and application issues flow to the same channel.

### c. Dashboards

* Sentry has its own UI.
* For unified view: add **Sentry plugin datasource** to Grafana, or embed Sentry issue widgets in Grafana dashboards.



## 3. Kubernetes deployment steps

1. **Provision Sentry project** → get DSN.

2. **Create Kubernetes Secret** with DSN, e.g.:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: sentry-dsn
   type: Opaque
   stringData:
     SENTRY_DSN: "https://public_key@o0.ingest.sentry.io/0"
   ```

3. **Inject into Django Deployment** via environment variable.

4. **Redeploy Django pods**.

5. **Confirm events** appear in Sentry dashboard.

6. **Validate cross-linking** by checking that `otel_trace_id` tags appear in Sentry issues and can be matched with Grafana Cloud Tempo traces.



### Outcome

* Grafana Cloud handles **metrics, logs, distributed traces, and alerting**.
* Sentry handles **rich error context, stack traces, and release tracking**.
* Trace IDs bridge them, so developers can pivot from an API error in Sentry to the full request trace in Grafana Cloud.
