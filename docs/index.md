# 🚀 Monitoring Kubernetes with Prometheus — A Personal Project

Welcome to my **Prometheus on Kubernetes documentation project** — a personal initiative to showcase deep expertise in **monitoring, observability, and cloud-native systems at scale**.

This site is structured as a **complete guide to running Prometheus in Kubernetes**, blending fundamentals, hands-on examples, best practices, and advanced topics.
It is designed for **engineers, SREs, and learners** who want to understand Kubernetes monitoring from **zero to production-grade scale**.

---

## 🎯 Project Purpose

* ✅ Demonstrate **expert knowledge** in Prometheus, Kubernetes, and cloud-native observability.
* 📚 Provide a **world-class learning resource** tailored to Kubernetes users, assuming no prior experience.
* 🧑‍💼 Serve as a **portfolio piece** highlighting my skills in **technical writing, system design, and DevOps practices**.

---

## 🧭 Quick Overview of Prometheus on Kubernetes

Prometheus is the **de-facto monitoring and alerting toolkit** in Kubernetes that:

* 🔧 Collects metrics from **nodes, pods, and services** via service discovery.
* 📊 Stores them in a purpose-built **time-series database (TSDB)**.
* 🔍 Uses **PromQL** to query cluster and application metrics.
* 🔔 Sends alerts via **Alertmanager**.
* ⚡ Integrates seamlessly with **Grafana** for dashboards.

---

## ⚡ Getting Started

Jump into the **[Quick Start](0-quickstart/1-getting-started.md)** guide to see Kubernetes monitoring in action:

* Deploy Prometheus on Kubernetes (Helm, Operator, or manifests).
* Scrape metrics from nodes, pods, and system components.
* Query cluster data using PromQL.
* Build dashboards in Grafana.
* Configure a basic alert in Alertmanager.

---

## 📐 Architecture & Components

This project explains Kubernetes monitoring from the ground up:

* **Core Concepts** → Metrics, exporters, service discovery, labels.
* **Prometheus Server** → Scraping workloads and cluster components.
* **Alertmanager** → Routing and managing cluster alerts.
* **Grafana** → Visualizing cluster health and application metrics.
* **Scaling** → Prometheus Operator, Thanos, Cortex, federation, long-term storage.

See detailed workflows in **[Architecture](1-architecture/0-overview.md)**.

---

## 📚 Documentation Roadmap

The guide follows a **step-by-step learning path**:

* **Quick Start** → Deploy Prometheus in Kubernetes and scrape your first metrics.
* **Architecture** → Understand internals, exporters, and service discovery.
* **PromQL & Features** → Querying cluster/app data, defining alerts.
* **Scaling & Best Practices** → HA setups, long-term storage, high-cardinality strategies.
* **About Me** → My background, expertise, and contact info.

---

## 🧪 Example Use Cases

Prometheus in Kubernetes enables:

* 🖥️ **Cluster Monitoring** → Node health, kubelet, API server, etcd.
* 📦 **Workload Monitoring** → Pod metrics, deployments, HPA scaling signals.
* 🛠️ **Application Monitoring** → Service SLIs, latency, error rates, traffic.
* 🔔 **Alerting** → Node pressure, pod crashes, SLA breaches, anomalies.

All use cases are illustrated with **hands-on Kubernetes demos**.

---

## 🔗 Useful Links

* ⚡ [Getting Started](0-quickstart/1-getting-started.md)
* 📊 [System Architecture](1-architecture/0-overview.md)
* 📝 [PromQL & Features](2-project/prometheus.md)
* 🧑‍💼 [About Me](4-about/0-about.md)

---

## 💡 About This Project

This documentation is a **personal showcase of Kubernetes monitoring expertise**.
It combines:

* Deep Kubernetes integration knowledge
* Hands-on best practices with Prometheus Operator & exporters
* Clear explanations for beginners and advanced practitioners

License: [MIT](https://github.com/sean-njela/k8s_monitoring/blob/main/LICENSE)

Maintained by **Sean Njela**.

---