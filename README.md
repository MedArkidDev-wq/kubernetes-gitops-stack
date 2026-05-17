# Kubernetes GitOps with ArgoCD

Change a YAML file, push to Git, and the cluster updates itself. No `kubectl apply`, no SSH, no manual anything.

This project runs a k3s cluster with ArgoCD watching this repo. When you change something (like bumping replicas from 2 to 4), ArgoCD picks it up within 3 minutes and syncs the cluster. It also auto-fixes drift, so if someone manually changes something in the cluster, ArgoCD reverts it back to what's in Git.

The stack includes a frontend (Nginx), a backend (Flask API that auto-scales from 2 to 10 pods), and full monitoring with Prometheus, Grafana, and Loki.

## What's in here

```
kubernetes-gitops-stack/
├── apps/
│   ├── namespace.yaml              # production namespace
│   ├── frontend/
│   │   ├── deployment.yaml         # 2 Nginx replicas
│   │   └── service.yaml            # LoadBalancer
│   └── backend/
│       ├── deployment.yaml         # 2 replicas, resource limits, health probes
│       ├── service.yaml            # ClusterIP (internal traffic only)
│       └── hpa.yaml                # Scales 2-10 pods based on CPU/memory
├── infrastructure/
│   ├── argocd/application.yaml     # Tells ArgoCD to watch the apps/ folder
│   └── monitoring/                 # Helm install commands for the monitoring stack
├── helm/my-app/
│   ├── Chart.yaml
│   ├── values.yaml                 # All the knobs you can turn
│   └── templates/                  # Deployment, Service, HPA templates
└── README.md
```

## How GitOps actually works here

ArgoCD watches the `apps/` directory. When you push a change:

```bash
# Say you want to scale the backend to 4 replicas
# Edit apps/backend/deployment.yaml: replicas: 2 -> replicas: 4
git commit -am "scale backend to 4"
git push

# ArgoCD detects the change within 3 minutes
# Pods go from 2 to 4 automatically
# You never ran a single kubectl command
```

If someone manually scales to 6 pods using `kubectl`, ArgoCD will notice the drift and scale it back to 4 (because that's what Git says). That's the selfHeal setting.

## The backend deployment

The backend has proper production settings. Resource limits so a runaway pod can't eat the whole node, health probes so K8s knows when to restart or stop sending traffic, and Prometheus annotations so metrics get scraped automatically.

```yaml
resources:
  requests: { memory: "64Mi", cpu: "50m" }
  limits: { memory: "128Mi", cpu: "200m" }
livenessProbe:
  httpGet: { path: /health, port: 5000 }
readinessProbe:
  httpGet: { path: /health, port: 5000 }
```

## Auto-scaling

The HPA watches CPU and memory. When CPU goes above 70% or memory above 80%, it adds pods. When load drops, it scales back down (slowly, with a 5-minute cooldown to avoid flapping).

Min 2 pods, max 10. It adds up to 2 pods per minute when scaling up, and removes 1 pod every 2 minutes when scaling down.

## The Helm chart

There's also a full Helm chart in `helm/my-app/` if you want to deploy with Helm instead of raw manifests. All the settings (replicas, resources, probes, autoscaling) are configurable through `values.yaml`.

```bash
helm install my-app helm/my-app -n production
helm upgrade my-app helm/my-app --set replicaCount=3
helm rollback my-app 1
```

## Get it running

```bash
# Install k3s (takes about 30 seconds)
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get the admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# Open the ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# Go to https://localhost:8080, login with admin + the password above

# Tell ArgoCD to watch this repo
kubectl create namespace production
kubectl apply -f infrastructure/argocd/application.yaml
```

After that, any push to this repo automatically updates the cluster.

## Testing the auto-scaler

```bash
# Generate load to trigger HPA
kubectl run load-test --image=busybox --rm -it --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://backend-api-service/slow; done"

# Watch pods scale up in another terminal
kubectl get hpa -n production --watch
```

## Cost

$0. k3s runs on any Linux machine, your laptop, a VM, even a Raspberry Pi.

## About

Built by [Mohamed Arkid](https://moarkid.com). DevOps engineer. I automate things so I don't have to touch them again.

MIT License
