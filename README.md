# GitOps Kubernetes Configuration with ArgoCD

This repository contains Kubernetes manifests and custom Helm charts to deploy and manage a monitoring stack (Prometheus & Grafana) using ArgoCD (GitOps).

## Directory Structure

```
.
├── apps/                         # ArgoCD Application manifests
│   ├── prometheus.yaml           # Deploying Prometheus chart
│   └── grafana.yaml              # Deploying Grafana chart
└── charts/                       # Custom Helm charts
    ├── prometheus/               # Prometheus deployment manifests
    └── grafana/                  # Grafana deployment manifests
```

---

## 1. Helm Charts

### Prometheus (`charts/prometheus`)
A custom lightweight Helm chart deploying:
- **Namespace**: `monitoring`
- **Service Account, ClusterRole, ClusterRoleBinding**: Grants Prometheus read access to pods, nodes, services, and endpoints.
- **ConfigMap**: Standard Prometheus scrape configuration (prometheus.yml) pre-configured with Kubernetes service discovery (self-scrape, apiservers, nodes, cAdvisor, and annotated pods).
- **Deployment**: Single replica deployment of Prometheus (`prom/prometheus`) with retention settings and liveness/readiness probes.
- **Service**: Exposed via **NodePort** on port **30090**.

### Grafana (`charts/grafana`)
A custom lightweight Helm chart deploying:
- **Namespace**: `monitoring` (shared with Prometheus)
- **Secret**: Encrypted admin credentials (`admin` / `admin123` by default).
- **ConfigMap**: Automates provisioning of:
  - Prometheus datasource (points to internal Prometheus service: `http://prometheus.monitoring.svc.cluster.local:9090`).
  - Dashboard provider (configured to auto-load JSON dashboards from `/var/lib/grafana/dashboards`).
- **PersistentVolumeClaim (PVC)**: Retains SQLite configuration db, plugins, and uploaded dashboards.
- **Deployment**: Single replica deployment of Grafana (`grafana/grafana`) referencing credentials and mounting config/storage.
- **Service**: Exposed via **NodePort** on port **30030**.

---

## 2. ArgoCD GitOps Deployment

To deploy this monitoring stack via ArgoCD:

### Prerequisites
- ArgoCD running on your cluster.
- A public or private repository URL that points to this configuration repo.

### Step 1: Update Repo URL
In [apps/prometheus.yaml](file:///c:/Users/Borhan/Desktop/study/gitops/k8s-config-argocd/apps/prometheus.yaml) and [apps/grafana.yaml](file:///c:/Users/Borhan/Desktop/study/gitops/k8s-config-argocd/apps/grafana.yaml), replace the `repoURL` value with your actual repository URL:
```yaml
spec:
  source:
    repoURL: https://github.com/YOUR_USERNAME/k8s-config-argocd.git
```

### Step 2: Apply ArgoCD Applications
Deploy the applications into your ArgoCD namespace (usually `argocd`):
```bash
kubectl apply -f apps/prometheus.yaml -n argocd
kubectl apply -f apps/grafana.yaml -n argocd
```

ArgoCD will automatically:
1. Create the `monitoring` namespace.
2. Synchronize all Helm charts under `charts/`.
3. Auto-heal any drift or manual cluster modifications back to the Git state.

---

## 3. Accessing Services

Once deployed and in `Synced`/`Healthy` state, you can access the dashboards via NodePort:

- **Prometheus UI**: `http://<NODE_IP>:30090`
- **Grafana UI**: `http://<NODE_IP>:30030`
  - **Username**: `admin`
  - **Password**: `admin123` (configured in values)
