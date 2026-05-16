# Production-Grade Kubernetes GitOps Stack

> **Git is the single source of truth.** Every change to this repo is automatically applied to the cluster via ArgoCD. Full metrics + logging observability included.

![Kubernetes](https://img.shields.io/badge/Kubernetes-k3s-326CE5?style=flat&logo=kubernetes&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-EF7B4D?style=flat&logo=argo&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-Charts-0F1689?style=flat&logo=helm&logoColor=white)
![Last Commit](https://img.shields.io/github/last-commit/MedArkidDev-wq/kubernetes-gitops-stack)

## What This Project Does

A Kubernetes cluster where Git is the single source of truth. Push a change to this repo — ArgoCD detects it, syncs it to the cluster, and your application updates automatically. Includes HPA auto-scaling, Helm charts, and full Prometheus/Grafana/Loki observability. Zero `kubectl apply` needed after initial setup.

## Architecture

```
Developer pushes to Git
         |
         v
  +------+------+
  |   ArgoCD    |  Watches repo every 3 min
  |   GitOps    |  Detects drift, auto-syncs
  +------+------+
         |
         v
  +------+------+
  |  Kubernetes |  k3s cluster
  |   Cluster   |
  +------+------+
         |
    +----+----+----+
    |    |    |    |
    v    v    v    v
  Front  Back  DB  Monitoring
  end    end       (Prometheus
  (2)   (2-10)     + Grafana
                   + Loki)
```

## Stack

| Component | Technology | Purpose |
|---|---|---|
| **Cluster** | k3s | Lightweight Kubernetes on any machine |
| **GitOps** | ArgoCD | Auto-sync from Git to cluster |
| **Scaling** | HPA (Horizontal Pod Autoscaler) | 2-10 replicas based on CPU/memory |
| **Packaging** | Helm | Templated, reusable Kubernetes charts |
| **Metrics** | kube-prometheus-stack | Prometheus + Grafana + AlertManager |
| **Logs** | Loki + Promtail | Log aggregation and search |
| **Frontend** | Nginx (2 replicas) | Static content + load balancing |
| **Backend** | Flask API (2-10 replicas) | REST API with Prometheus metrics |

## Project Structure

```
kubernetes-gitops-stack/
├── apps/
│   ├── frontend/
│   │   ├── deployment.yaml         # 2 replicas, nginx:1.25-alpine
│   │   └── service.yaml            # LoadBalancer type
│   ├── backend/
│   │   ├── deployment.yaml         # 2 replicas, health probes, resource limits
│   │   ├── service.yaml            # ClusterIP (internal only)
│   │   └── hpa.yaml                # Auto-scale 2-10 pods on CPU > 70%
│   └── database/
│       ├── statefulset.yaml
│       └── service.yaml
├── infrastructure/
│   ├── argocd/
│   │   └── application.yaml        # GitOps application definition
│   └── monitoring/
│       └── kube-prometheus-stack.yaml
├── helm/
│   └── my-app/
│       ├── Chart.yaml              # Helm chart metadata
│       ├── values.yaml             # Default configuration
│       └── templates/
│           └── deployment.yaml     # Templated deployment
└── README.md
```

## Backend Deployment (Key Features)

```yaml
spec:
  replicas: 2
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"     # Auto-discovered by Prometheus
        prometheus.io/port: "5000"
    spec:
      containers:
        - name: backend
          resources:
            requests: { memory: "64Mi", cpu: "50m" }
            limits: { memory: "128Mi", cpu: "200m" }
          livenessProbe:
            httpGet: { path: /health, port: 5000 }
          readinessProbe:
            httpGet: { path: /health, port: 5000 }
```

## HPA Auto-Scaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
    - type: Resource
      resource:
        name: memory
        target: { type: Utilization, averageUtilization: 80 }
```

## ArgoCD GitOps Configuration

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    repoURL: https://github.com/MedArkidDev-wq/kubernetes-gitops-stack.git
    targetRevision: HEAD
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Delete resources removed from Git
      selfHeal: true     # Auto-fix manual cluster changes
```

**The GitOps magic:**
```bash
# Change replicas: 2 -> 4 in apps/backend/deployment.yaml
git commit -am "scale: increase backend to 4 replicas"
git push

# ArgoCD detects change within 3 minutes
# Pods scale from 2 to 4 automatically
# Zero kubectl commands needed
```

## Monitoring Stack (Helm)

```bash
helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword="DevOps123!" \
  --set prometheus.prometheusSpec.retention=15d

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true
```

## Custom Helm Chart

```yaml
# helm/my-app/values.yaml
replicaCount: 2
image:
  repository: my-devops-app
  tag: latest
service:
  type: ClusterIP
  port: 80
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
monitoring:
  enabled: true
  serviceMonitor: true
```

```bash
helm lint helm/my-app                           # Validate
helm install my-app helm/my-app -n production   # Deploy
helm upgrade my-app helm/my-app --set replicaCount=3  # Scale
helm rollback my-app 1                          # Rollback
```

## Quick Start

```bash
# 1. Install k3s
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# 2. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Get ArgoCD password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# 4. Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# https://localhost:8080

# 5. Apply the GitOps application
kubectl create namespace production
kubectl apply -f infrastructure/argocd/application.yaml

# 6. Install monitoring
helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack -n monitoring
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring &
```

## Testing GitOps

```bash
# Verify ArgoCD sync
argocd app list
# production-app — Synced — Healthy

# Test auto-scaling
kubectl run load-test --image=busybox --rm -it --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://backend-api-service/slow; done"

# Watch HPA add pods
kubectl get hpa -n production --watch
```

## Cost

**$0** — k3s runs on any machine (laptop, VM, Raspberry Pi).

## Author

**Mohamed Arkid** — DevOps Engineer and Cloud Consultant

- [moarkid.com](https://moarkid.com)
- [LinkedIn](https://www.linkedin.com/in/mohamed-arkid)

## License

MIT
