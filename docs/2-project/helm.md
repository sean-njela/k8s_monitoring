# 📦 What is Helm?

Helm is the **package manager for Kubernetes**.
It manages **charts**, which are preconfigured collections of Kubernetes manifests.

Charts bundle deployments, services, config maps, secrets, ingress rules, and more into a single, reusable unit.

Helm makes installing, upgrading, rolling back, and managing complex Kubernetes apps predictable and repeatable.

---

## 🧐 Why Use Helm?

Kubernetes manifests are verbose and repetitive. Managing large apps by hand is error-prone. Helm provides:

* **Templating**: DRY YAML with variables and conditionals.
* **Versioning**: Rollback to earlier states easily.
* **Releases**: Track each installation separately.
* **Configuration**: Use `values.yaml` to override defaults.
* **Repositories**: Share charts publicly or privately.

👉 Helm = apt/yum/homebrew, but for Kubernetes.

---

## 🔧 Helm Architecture

```text
   +-------------+
   | Helm Client |
   +------+------+         +-----------------+
          |                |  Kubernetes API |
          |                +--------+--------+
          |                         |
   +------+-------------------------+------+
   |    Kubernetes Cluster (CRDs, RBAC,    |
   |   Deployments, Services, Secrets, etc)|
   +---------------------------------------+
```

* **Helm Client** → CLI tool (`helm`).
* **Release** → A chart installed into a namespace with a given name.
* **Chart** → A bundle of templates and default values.
* **Values** → User configuration overrides.

---

## 📝 Core Concepts

| Term           | Meaning                                                         |
| -------------- | --------------------------------------------------------------- |
| **Chart**      | Package of Kubernetes resources.                                |
| **Release**    | A deployed instance of a chart, identified by name + namespace. |
| **Values**     | User-supplied config to customise a chart.                      |
| **Repository** | Place where charts are stored and fetched (`helm repo add`).    |

---

## 🚀 Basic Commands

### Install a chart

```bash
helm install <release-name> <chart> -n <namespace>
```

### Upgrade a release

```bash
helm upgrade <release-name> <chart> -f values.yaml -n <namespace>
```

### Rollback to previous revision

```bash
helm rollback <release-name> <revision> -n <namespace>
```

### List releases

```bash
helm list -n <namespace>
```

### Uninstall a release

```bash
helm uninstall <release-name> -n <namespace>
```

---

## 📊 Managing Deployments After Install

1. **Scale workloads to zero (stop running without deletion)**

```bash
kubectl scale deployment <name> --replicas=0 -n <namespace>
kubectl scale statefulset <name> --replicas=0 -n <namespace>
```

2. **Pause rollout**

```bash
kubectl rollout pause deployment <name> -n <namespace>
```

3. **Change Helm values to persist stop**

```yaml
replicaCount: 0
```

```bash
helm upgrade <release-name> <chart> -f values.yaml -n <namespace>
```

---

## ⚙️ Helm Chart Structure

```text
mychart/
  Chart.yaml        # Metadata (name, version, description)
  values.yaml       # Default configuration values
  charts/           # Subcharts (dependencies)
  templates/        # Kubernetes manifests (templated YAML)
  templates/_helpers.tpl  # Helper template functions
```

---

## 🔍 Helm Values

Values control how templates render.

Example `values.yaml`:

```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.25.0"
service:
  type: ClusterIP
  port: 80
```

Override values:

```bash
helm install myapp ./mychart --set replicaCount=3 --set image.tag=1.26
```

---

## 🛠 Template Language

Helm templates use **Go templating**.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nginx
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

---

## 🧾 Helm Cheat Sheet

### ✅ Core Operations

| Task         | Command                                                 |
| ------------ | ------------------------------------------------------- |
| Install      | `helm install <release> <chart> -n <ns>`                |
| Upgrade      | `helm upgrade <release> <chart> -f values.yaml -n <ns>` |
| Rollback     | `helm rollback <release> <rev> -n <ns>`                 |
| Delete       | `helm uninstall <release> -n <ns>`                      |
| Diff changes | `helm diff upgrade <release> <chart>`                   |
| Dry run      | `helm install <release> <chart> --dry-run --debug`      |

---

### 🔧 Chart Dev Commands

| Task            | Command                            |
| --------------- | ---------------------------------- |
| Create chart    | `helm create <name>`               |
| Lint chart      | `helm lint ./mychart`              |
| Package chart   | `helm package ./mychart`           |
| Push chart      | `helm push ./mychart oci://<repo>` |
| Template render | `helm template ./mychart`          |

---

### 🛡️ Security Notes

* Do not store plaintext secrets in `values.yaml`. Use **Sealed Secrets** or **External Secrets Operator**.
* Restrict Tiller (Helm v2 only, obsolete). Helm v3 does not need server-side components.
* Control RBAC for Helm users carefully.

---

### ⚠️ Risks & Trade-offs

* **Scaling to zero via kubectl** → Helm unaware, overwritten on next upgrade.
* **Editing values.yaml** → More controlled, but requires upgrade.
* **Rollout pause** → Stops updates, not running pods.
* **Complex charts** → Hard to debug, encourage over-abstraction.

---

## Helm CLI cheat sheet

### Installation and context

* Show version.

```bash
helm version
```

* Show Helm env paths.

```bash
helm env
```

* Set kubeconfig and context per command.

```bash
helm --kubeconfig ~/.kube/config --kube-context myctx -n myns <subcommand>
```

## Repositories (classic index-based)

* Add, update, list, remove.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
helm repo remove bitnami
```

* Search for charts.

```bash
helm search repo nginx
helm search hub wordpress            # searches Artifact Hub
```

* Inspect a remote chart.

```bash
helm show chart bitnami/nginx
helm show values bitnami/nginx
helm show readme bitnami/nginx
helm show crds bitnami/nginx
```

* Download a chart locally.

```bash
helm pull bitnami/nginx --untar --untardir ./charts
```

Reference for repo and show commands.

## OCI registries (Helm 3.8+)

* Log in, push, pull.

```bash
helm registry login ghcr.io              # or ACR/ECR/Harbor
helm push ./mychart-0.1.0.tgz oci://ghcr.io/org/charts
helm pull oci://ghcr.io/org/charts/mychart --version 0.1.0 --destination ./charts
```

* Install from an OCI registry.

```bash
helm install myrel oci://ghcr.io/org/charts/mychart --version 0.1.0
```

Notes on `oci://` usage and tag inference.

## Install, upgrade, rollback, uninstall

* Install.

```bash
helm install myrel bitnami/nginx -n web \
  -f values.yaml \
  --set image.tag=1.25.5 \
  --set-string env.MODE=prod \
  --set-file cert=./tls.crt \
  --version 15.5.0 \
  --create-namespace \
  --wait --wait-for-jobs --timeout 10m \
  --atomic
```

* Upgrade or install if missing.

```bash
helm upgrade myrel bitnami/nginx -n web \
  -f values-prod.yaml \
  --reuse-values \
  --install \
  --wait --timeout 10m --atomic
```

* Roll back.

```bash
helm history myrel -n web
helm rollback myrel 4 -n web --wait --timeout 10m
```

* Uninstall.

```bash
helm uninstall myrel -n web                 # remove resources
helm uninstall myrel -n web --keep-history  # keep release secrets
```

Flags and behaviours.

## Listing and inspecting releases

* List releases.

```bash
helm list -n web
helm list -A --deployed --failed -d         # sort by date
helm list -n web --filter '^myrel$'
```

* Status and resources.

```bash
helm status myrel -n web --show-resources
```

* Get rendered manifest or values used.

```bash
helm get manifest myrel -n web
helm get values myrel -n web                # user-supplied only
helm get values myrel -n web -a             # computed values
helm get notes myrel -n web
```

Listing and get docs.

## Dry runs and diffs

* Client or server dry run plus debug.

```bash
helm install myrel ./chart --dry-run=client --debug
helm upgrade myrel ./chart --dry-run=server --debug
```

* Diff planned changes (plugin).

```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade myrel ./chart -f values.yaml -n web
helm diff revision myrel 5 6 -n web
```

Dry-run options and diff plugin.

## Values handling and precedence

* Multiple values files and flags.

```bash
helm install r ./chart -f base.yaml -f override.yaml \
  --set a.b=1 --set-string path="/var/log" --set-file big=./blob.txt \
  --set-json 'features=["a","b"]'
```

* Precedence order, last one wins for repeated keys:

  1. `--set*` and `--values` in right-most order.
  2. Files listed later override earlier.
  3. Chart defaults are lowest.

## Chart development

* Create, lint, render, package.

```bash
helm create mychart
helm lint ./mychart
helm template myrel ./mychart -f values.yaml > all.yaml
helm package ./mychart -d ./dist
```

* Sign and verify packages.

```bash
helm package --sign --key 'mykey' --keyring ~/.gnupg/pubring.gpg ./mychart
helm verify ./dist/mychart-0.1.0.tgz
```

* Dependencies.

```bash
# Chart.yaml: dependencies: [{ name, version, repository }]
helm dependency update ./mychart
helm dependency list ./mychart
```

Template, package, verify, and dependency docs.

## Registries and repos for your own charts

* Classic repo index (static HTTP).

```bash
helm package ./mychart -d ./repo
helm repo index ./repo --url https://example.com/charts
```

* OCI push to a cloud registry (example ECR).

```bash
aws ecr get-login-password | helm registry login --username AWS --password-stdin <acct>.dkr.ecr.<region>.amazonaws.com
helm push ./dist/mychart-0.1.0.tgz oci://<acct>.dkr.ecr.<region>.amazonaws.com/charts
```

OCI usage references.

## Post-renderers (Kustomize or custom binaries)

* Apply last-mile patches without forking charts.

```bash
helm install myrel oci://ghcr.io/org/charts/app \
  --post-renderer ./kustomize-wrapper.sh
```

* Works with `install`, `upgrade`, and `template`.
  Post-renderer capability.

## Testing releases

* Define test Pods in `templates/` with `helm.sh/hook: test`.
* Run tests for a deployed release.

```bash
helm test myrel -n web --logs
```

Chart tests and `helm test` usage.

## Security and integrity

* Verify signed charts before use.

```bash
helm install myrel ./signedchart.tgz --verify --keyring ~/.gnupg/pubring.gpg
```

* Prefer secrets managers for sensitive values. Use SOPS or an external secrets operator.
  Provenance and verify docs.

## Common operational patterns

* Idempotent deploy in CI.

```bash
helm upgrade --install myrel ./chart -n web -f values.yaml --wait --timeout 10m --atomic
```

* Render then apply with `kubectl` when needed.

```bash
helm template myrel ./chart -f values.yaml | kubectl apply -f -
```

* Stop workloads without deleting resources.

```bash
# Preferred: set replicaCount: 0 in values and upgrade so Helm tracks the state
helm upgrade myrel ./chart -n web -f values-stop.yaml
```

Install/upgrade flags and semantics.

## Troubleshooting

* See reasons for a failed or pending release.

```bash
helm status myrel -n web
helm history myrel -n web
kubectl describe pods,deploy,sts -n web
```

* Retry with clean failure handling.

```bash
helm upgrade myrel ./chart -n web --cleanup-on-fail --atomic --wait --timeout 10m
```

Upgrade flags reference.

## Useful globals and quality-of-life

* Completions.

```bash
helm completion bash|zsh|fish|powershell
```

* Plugins.

```bash
helm plugin list
helm plugin install https://github.com/databus23/helm-diff
helm plugin update diff
helm plugin uninstall diff
```

* Show all commands.

```bash
helm help
helm <subcommand> --help
```

Commands index and plugin guide.

## Flags worth remembering

* `--namespace/-n`, `--kube-context`, `--kubeconfig`, `--wait`, `--wait-for-jobs`, `--timeout`, `--atomic`, `--cleanup-on-fail`, `--devel`, `--version`, `--dependency-update`, `--skip-crds`, `--no-hooks`, `--post-renderer`, `--set`, `--set-string`, `--set-file`, `--set-json`, `-f/--values`.

## Risks and trade-offs

* `kubectl scale` changes drift from Helm state. Persist intent in values where possible.
* `--force` replaces resources and can briefly delete and recreate objects. Use with caution.