# CICD Project One — Infrastructure Repository

A complete GitOps-based CI/CD infrastructure setup for a Python application, deployed on a local Kubernetes cluster using **Kind**, exposed via **Traefik** (Gateway API + HTTPRoute), managed with **Helm**, and continuously delivered using **ArgoCD**.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Repository Structure](#repository-structure)
3. [Prerequisites](#prerequisites)
4. [Phase 1 — Docker Image](#phase-1--docker-image)
5. [Phase 2 — Kind Cluster Setup](#phase-2--kind-cluster-setup)
6. [Phase 3 — Raw Kubernetes Resources](#phase-3--raw-kubernetes-resources)
7. [Phase 4 — Traefik + Gateway API](#phase-4--traefik--gateway-api)
8. [Phase 5 — Helm Chart](#phase-5--helm-chart)
9. [Phase 6 — ArgoCD Setup (GitOps CD)](#phase-6--argocd-setup-gitops-cd)
10. [Port Forwarding & Local Access](#port-forwarding--local-access)
11. [Common Debugging Commands](#common-debugging-commands)
12. [Known Issues & Fixes](#known-issues--fixes)
13. [Architecture Summary](#architecture-summary)

---

## Project Overview

This project sets up an end-to-end CI/CD pipeline for a Python web application:

- **Source code** lives in `python_src/` (separate repo) — see that repo's README for app setup and CI details
- **Infrastructure specs** live in this repo (`Infra_specs/`)
- The app is containerized, pushed to Docker Hub (`shawnprac/cicd_one_python_app:v1`)
- Deployed on a local **Kind** (Kubernetes-in-Docker) cluster
- Traffic routing handled by **Traefik** using the **Gateway API** and **HTTPRoute**
- Packaged into a **Helm chart** for templated, reproducible deployments
- **ArgoCD** watches this repo and auto-syncs the cluster state (GitOps)

---

## Repository Structure

```
.
├── argocd/
│   ├── argocd_ingress.yaml         # Ingress resource to expose ArgoCD UI
│   └── argo_cd_ui_pass.txt         # Retrieved initial admin password (gitignored ideally)
├── CICD_PORT_FLOW_EXPLANATION.md   # Notes on port forwarding flow
├── create_cluster.yaml             # Kind cluster config (node ports, workers, etc.)
├── helm-charts/
│   └── infra-helm-python-app/      # Main Helm chart for the Python app
│       ├── Chart.yaml
│       ├── values.yaml             # All configurable values (image, replicas, ports, etc.)
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── httproute.yaml
│           ├── ingress.yaml
│           ├── hpa.yaml
│           ├── serviceaccount.yaml
│           ├── _helpers.tpl
│           └── NOTES.txt
├── K8s-Files/                      # Raw (non-Helm) K8s manifests — used for initial testing
│   ├── deployment.yaml
│   ├── service.yaml
│   └── httproute.yaml
├── history.txt
└── tree.txt
```

---

## Prerequisites

Make sure the following tools are installed before starting:

| Tool | Purpose |
|------|---------|
| `docker` | Build and run containers |
| `kind` | Create local K8s clusters using Docker |
| `kubectl` | Interact with Kubernetes clusters |
| `helm` | Package manager for Kubernetes |
| `argocd` CLI (optional) | Interact with ArgoCD from terminal |

Useful alias set during development:
```bash
alias k=kubectl
```

Python-Src repo -> [github/shawn-exe/CICD_ONE_PYTHON_SRC](https://github.com/shawn-exe/CICD_ONE_Python_src.git)


---

## Phase 1 — Docker Image

The Docker image is built from the Python source and pushed to Docker Hub.

```bash
docker build -t shawnprac/cicd_one_python_app:v1 .
docker push shawnprac/cicd_one_python_app:v1
```

> Image is public at: `docker.io/shawnprac/cicd_one_python_app:v1`

The app inside the container listens on port **5000**.

---

## Phase 2 — Kind Cluster Setup

Instead of a plain `kind create cluster`, a custom config file is used for control over node ports.

```bash
kind create cluster --config create_cluster.yaml
```

**`create_cluster.yaml`** specifies things like:
- Cluster name
- Extra port mappings (e.g., `30080` on the host → NodePort on the cluster)
- Worker node count

Verify the cluster is running:
```bash
kind get clusters
kubectl config get-contexts
kubectl config use-context kind-<cluster-name>
```

### Cluster Management Notes

During development, several clusters were created and deleted while iterating on the config:

```bash
kind delete cluster --name kind          # delete default cluster
kind create cluster --name cicd-one      # named cluster attempt
kind delete cluster --name cicd-one
kind create cluster --config create_cluster.yaml   # final config-based creation
```

---

## Phase 3 — Raw Kubernetes Resources

Before moving to Helm, raw manifests in `K8s-Files/` were applied manually to test the setup.

### Deployment (`K8s-Files/deployment.yaml`)
- Deploys the Python app using the Docker Hub image
- Sets container port to **5000**

### Service (`K8s-Files/service.yaml`)
- Named `python-app-service`
- Routes internal traffic to the app pod on port **5000**

### HTTPRoute (`K8s-Files/httproute.yaml`)
- Defines routing rules using the Gateway API
- Routes traffic from the Traefik Gateway to the `python-app-service`

**Apply order:**
```bash
k apply -f K8s-Files/service.yaml
k apply -f K8s-Files/deployment.yaml
k apply -f K8s-Files/httproute.yaml
```

**Verify:**
```bash
k get pods
k get svc
k get httproute
k get deploy
```

**Port-forward for quick local testing (before Traefik):**
```bash
k port-forward python-app-<pod-id> 8081:5000
```

---

## Phase 4 — Traefik + Gateway API

Traefik is used as the ingress/gateway controller, leveraging the **Kubernetes Gateway API** instead of the traditional Ingress API.

### Step 1: Install Gateway API CRDs

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
```

Verify CRDs are installed:
```bash
k get crds
k get gatewayclass
```

### Step 2: Install Traefik via Helm

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --skip-crds \
  --set providers.kubernetesGateway.enabled=true \
  --set service.type=NodePort \
  --set ports.web.nodePort=30080
```

- `providers.kubernetesGateway.enabled=true` — enables Gateway API support in Traefik
- `service.type=NodePort` — exposes Traefik on a NodePort (needed for Kind)
- `ports.web.nodePort=30080` — maps to port 30080 on Kind nodes (which maps to host via `create_cluster.yaml`)

Verify Traefik is running:
```bash
k get pods -n traefik
k get svc -n traefik
k get gateway -n traefik
```

### Step 3: Apply HTTPRoute

```bash
k apply -f K8s-Files/httproute.yaml
k get httproute
k describe httproute python-app-route
```

### Step 4: /etc/hosts Entry

To access the app via hostname locally, add an entry to `/etc/hosts`:

```bash
sudo vi /etc/hosts
```

Add a line like:
```
127.0.0.1   python-app.test.com
```

Test with curl:
```bash
curl -v http://192.168.49.2:30080 -H "Host: python-app.test.com"
```

---

## Phase 5 — Helm Chart

To make deployments repeatable and configurable, a Helm chart was created.

```bash
mkdir helm-charts
cd helm-charts
helm create infra-helm-python-app
```

The generated chart was customized:
- `values.yaml` — set image repo, tag, service port, replica count, HTTPRoute config
- `templates/httproute.yaml` — added Gateway API HTTPRoute template
- `templates/deployment.yaml` — uses values for image, ports, etc.
- `templates/service.yaml` — ClusterIP service pointing to app pods

**Install the chart:**
```bash
helm install cicd-project-one infra-helm-python-app
```

**Upgrade after changes:**
```bash
# From inside the chart directory:
helm upgrade cicd-project-one . -f values.yaml

# From the parent directory:
helm upgrade cicd-project-one infra-helm-python-app -f infra-helm-python-app/values.yaml
```

**Verify:**
```bash
k get all
k get httproute
helm list
```

---

## Phase 6 — ArgoCD Setup (GitOps CD)

ArgoCD is used to watch this Git repo and automatically sync the cluster state with what's defined in `helm-charts/`.

### Installation

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd
```

Verify pods are running:
```bash
k get pods -n argocd
k get all -n argocd
```

### Get Initial Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Save this — it's the password for the `admin` user on the ArgoCD UI.

### Access ArgoCD UI

**Via port-forward:**
```bash
kubectl port-forward svc/argocd-server -n argocd 8082:80
```

Then open: `http://localhost:8082`

### Fix: Disable TLS (server.insecure)

ArgoCD by default redirects HTTP → HTTPS, which causes issues when accessed via plain HTTP (especially through Traefik without TLS). The fix is to set `server.insecure = "true"` in the ArgoCD config map:

```bash
kubectl edit configmap argocd-cmd-params-cm -n argocd
```

Add under `data`:
```yaml
data:
  server.insecure: "true"
```

Then restart the ArgoCD server deployment:
```bash
kubectl rollout restart deployment argocd-server -n argocd
```

### ArgoCD Ingress

An Ingress resource was created to expose ArgoCD via Traefik at a hostname.

**`argocd/argocd_ingress.yaml`:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - host: argocd.test.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

```bash
k apply -f argocd/argocd_ingress.yaml
k get ingress -n argocd
k describe ingress argocd-ingress -n argocd
```

Add to `/etc/hosts`:
```
127.0.0.1   argocd.test.com
```

---

## Port Forwarding & Local Access

Since this runs on **Kind** (not a cloud provider), there's no external LoadBalancer. Here's how traffic flows:

```
Browser (localhost:30080)
        ↓
Kind cluster NodePort (30080) — mapped in create_cluster.yaml
        ↓
Traefik Service (NodePort)
        ↓
Traefik Pod (Gateway Controller)
        ↓
HTTPRoute (matches hostname/path)
        ↓
python-app Service (ClusterIP)
        ↓
python-app Pod (port 5000)
```

For ArgoCD:
```
Browser (argocd.test.com → 127.0.0.1 via /etc/hosts)
        ↓
Traefik (NodePort 30080)
        ↓
Ingress → argocd-server Service → ArgoCD Pod
```

---

## Common Debugging Commands

```bash
# Check all resources
k get all
k get all -n argocd
k get all --all-namespaces

# Pods
k get pods
k get pods -A
k get pods -n traefik
k get pods -n argocd

# Services
k get svc
k get svc -n traefik
k get svc -n argocd

# Routing
k get httproute
k describe httproute <name>
k get ingress -A
k get gateway -n traefik
k describe gateway traefik-gateway -n traefik
k get gatewayclass

# Logs
kubectl logs pod/<pod-name>

# Helm
helm list
helm list -n argocd

# Node info (useful for Kind IP)
k get nodes -o wide

# Connectivity test
curl -v http://192.168.49.2:30080 -H "Host: argocd.test.com"
```

---

## Known Issues & Fixes

### 1. ArgoCD HTTP Redirect Loop
**Problem:** ArgoCD redirects HTTP → HTTPS, causing issues through Traefik without TLS.  
**Fix:** Set `server.insecure: "true"` in the `argocd-cmd-params-cm` ConfigMap, then rollout restart `argocd-server`.

### 2. Kind Cluster Name with Underscore
**Problem:** `kind create cluster --name cicd_one` failed — Kind doesn't support underscores in cluster names.  
**Fix:** Use hyphens: `kind create cluster --name cicd-one`.

### 3. HTTPRoute Not Routing Traffic
**Problem:** HTTPRoute was applied but traffic wasn't reaching the app.  
**Fix:** Verified that the Gateway resource in the `traefik` namespace had the correct `listeners` configuration and the HTTPRoute `parentRefs` matched the correct Gateway name and namespace.

### 4. Port-Forward to Wrong Port
**Problem:** App runs on port **5000** inside the container, not 8080.  
**Fix:** `k port-forward <pod> 8081:5000` (local:container).

### 5. Helm Upgrade Path Issues
**Problem:** Running `helm upgrade` from the wrong directory caused "chart not found" errors.  
**Fix:** Run from the `helm-charts/` parent directory:
```bash
helm upgrade cicd-project-one infra-helm-python-app -f infra-helm-python-app/values.yaml
```

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────┐
│                     Developer Machine                    │
│                                                          │
│  GitHub Repo (python_src)                                │
│         │  (CI handled in python_src repo)               │
│         │                    Docker Hub                  │
│         │                    shawnprac/cicd_one_python_app│
│         ↓                                                │
│  GitHub Repo (Infra_specs) ←── ArgoCD watches           │
│                                    │                     │
│                                    ↓                     │
│  Kind Cluster (Docker)                                   │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Namespace: default                                │  │
│  │    python-app Deployment (image from Docker Hub)   │  │
│  │    python-app Service (ClusterIP :5000)            │  │
│  │    HTTPRoute → traefik-gateway                     │  │
│  │                                                    │  │
│  │  Namespace: traefik                                │  │
│  │    Traefik Deployment                              │  │
│  │    Gateway (traefik-gateway)                       │  │
│  │    Service (NodePort :30080)                       │  │
│  │                                                    │  │
│  │  Namespace: argocd                                 │  │
│  │    ArgoCD Server + Components                      │  │
│  │    Ingress → argocd.test.com                       │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  /etc/hosts:                                             │
│    127.0.0.1  python-app.test.com                        │
│    127.0.0.1  argocd.test.com                            │
└─────────────────────────────────────────────────────────┘
```

---

## Git Commit History Summary

| Commit Message | What Changed |
|---|---|
| `Kind Cluster creation, K8s Resource files` | create_cluster.yaml, K8s-Files/ added |
| `Created Helm Structure for Infra-Specs` | helm-charts/ directory and chart scaffolding |
| `ArgoCD setup, argocd ingress` | argocd/ directory, ingress yaml |
| `Test: ArgoCD` | ArgoCD application config tested |
| `Removed github workflows - to skip analysis tests` | .github/ placeholder only, workflows removed |

---

*This README was reconstructed from terminal history and project structure. For detailed file contents, refer to the individual YAML files in the repo.*