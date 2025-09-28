# Taskfile Documentation

This document explains the structure, conventions, and usage of the main `Taskfile.yaml`. It serves as the primary automation entrypoint for development, deployment, monitoring, and utility workflows in this project.

## Purpose

The Taskfile provides a consistent command-line interface for common project operations. It eliminates the need to memorise long `kubectl`, `helm`, or `docker` commands, ensures reproducibility, and encodes conventions (naming, namespaces, repo URLs, etc.) in a single place.

All tasks can be executed with:

```bash
task <task-name>
```

## Global Configuration

### Version

The Taskfile uses **version 3** syntax.

### Output

```yaml
output: prefixed
```

Logs are prefixed with task names to make parallel output easier to follow.

### Environment

* `.env` and `.env.local` files are automatically loaded.
* Global environment variables (`ENV`, `DEBUG`) can be overridden per-task.

### Variables

Several key variables are declared for reuse across tasks:

* **Cluster and namespace:**

    * `KIND_CLUSTER_NAME`: Kind cluster name
    * `DEV_CLUSTER`: Development cluster logical name
    * `DEV_NS`: Kubernetes namespace for monitoring

* **Services & ports:**

    * `PROM_SVC`, `GRAF_SVC`, `APP_SVC`, `ALERTM_SVC`
    * `PROM_PORT`, `APP_PORT`, `ALERTM_PORT`, `GRAF_PORT`

* **Helm:**

    * `HELM_REPO_URL`, `HELM_CHART_NAME`, `CUSTOM_HELM_REPO_NAME`, `CUSTOM_HELM_CHART_NAME`
    * Values files for Helm (`FINAL_HELM_VALUES_FILE`, `READ_HELM_VALUES_FILE`)

* **Metadata:**

    * `PROJECT_NAME`, `TIMESTAMP` (dynamic), and app Docker image name.

This ensures **one change in variables propagates everywhere** (DRY principle).

## Includes

```yaml
includes:
  common:
    taskfile: ./Taskfile.gitflow.yaml
    flatten: true
```

A shared GitFlow Taskfile is included, so git-related automation is accessible directly without namespace prefixes.

## Core Tasks

### Default

Lists all tasks:

```bash
task
```

### Setup & Development

* **`setup`**: Creates cluster, loads images, installs Prometheus via Helm, and deploys the app.
* **`dev`**: Starts development services, retrieves Grafana credentials, and forwards ports.
* **`create-cluster-dev`**: Creates Kind cluster, switches context, creates namespace, and sets it active.
* **`prod`**: Placeholder for production deployment pipeline.

### Monitoring & Observability

* **`helm-install-prom`**: Adds Helm repo, installs Prometheus/Grafana/Alertmanager stack, saves chart values.

* **`helm-upgrade-prom`**: Upgrades the stack with current values.

* **`prom-helm-values`**: Dumps Helm chart values for inspection.

* **Port forward tasks:**

    * `port-fwd-prom`, `port-fwd-app`, `port-fwd-alertm`, `port-fwd-graf`

* **`get-graf-passwd`**: Fetches Grafana admin password from Kubernetes secret.

### Application Deployment

* **`deploy-app`**: First-time creation of manifests from `k8s/`.
* **`apply-app`**: Applies updates to manifests.
* **`delete-app`**: Removes app resources.

### System & Info

* **`ports`**: Lists open ports.
* **`info`**: Prints key service URLs.
* **`status`**: Runs `kubectl get all` in the namespace.
* **`versions`**: Uses `mike list` to check docs versions.

### Cleanup

* **`cleanup-dev`**: Tears down only development stack (Helm release, Kind cluster, k8s resources, repo references, images). Designed to fail gracefully if resources are missing.
* **`cleanup-prod`**: Placeholder for production cleanup.
* **`cleanup-all`**: Invokes both dev and prod cleanup.

### Utilities

#### Kind

* **`kind-load-images`**: Ensures application image is pulled locally and loaded into Kind cluster (faster than registry pulls).

#### Documentation

* **`docs`**: Runs `mkdocs serve` via Poetry, serving docs at [http://127.0.0.1:8000/docs/](http://127.0.0.1:8000/docs/).

#### Compression

* **`compress`**: Downscales and compresses MP4 to `assets/demo-video-small.mp4`. Supports runtime overrides (`crf`, `preset`).
* **`compress-gif`**: Optimises and compresses GIFs into `assets/demo-video-small.gif`. Generates and applies a palette for quality.
* **`mkdir-assets`**: Ensures `assets/` directory exists on Linux/macOS or Windows before compression tasks.

## Workflow Examples

1. **Bootstrap monitoring stack:**

   ```bash
   task setup
   ```

2. **Iterate locally:**

   ```bash
   task dev
   ```

3. **View monitoring endpoints:**

   ```bash
   task info
   ```

4. **Tear down environment:**

   ```bash
   task cleanup-dev
   ```

## Design Conventions

* **Fail gracefully**: Tasks often use `|| echo "error ..."` to avoid halting entire pipelines.
* **Idempotency**: Many tasks can be rerun without harm (e.g., `kubectl create namespace`).
* **Cross-platform**: Some tasks (`mkdir-assets`) include OS-specific logic.
* **Separation of concerns**: App vs. infra vs. docs tasks are grouped logically.
* **DRY configuration**: Variables centralise cluster, service, and chart settings.

## Notes & Recommendations

* To check all available tasks:

  ```bash
  task --list-all
  ```

!!! warning
    * Always verify Helm values files (`k8s/values` vs `k8s/read_values`) before upgrades.
    * Use port-forward tasks only for local debugging. For real environments, prefer ingress or LoadBalancer services.
    * `prod`, `cleanup-prod`, and other stubs are intentionally left for extension in real deployments.