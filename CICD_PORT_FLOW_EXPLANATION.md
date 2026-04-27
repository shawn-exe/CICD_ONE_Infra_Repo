# End-to-End CICD Project One: Port Flow & Request Routing

## Architecture Overview

This document explains the complete flow of a browser request through your Kubernetes cluster, from the initial HTTP request to the response from your Flask application.

**Stack**: K8s Deployment + Service + Gateway API + HTTPRoute + Traefik + ArgoCD

---

## Visual Flow Diagram

```
Browser Request (GET my-cicd-one.com/api/v1/info)
       ↓
Host Port 80 (KinD mapping)
       ↓
Node Container Port 30080 (Traefik Gateway Listener)
       ↓
HTTPRoute: python-app-route (matches hostname + path)
       ↓
Service: python-app-service (Port 8080)
       ↓
Pod Load Balancing (Port 5000)
       ↓
Flask Application Processing
       ↓
Response flows back through all layers
       ↓
Browser receives JSON response
```

---

## Step-by-Step Breakdown

### **Step 1: Browser Request Initiation**

```
Browser: GET http://my-cicd-one.com/api/v1/info
         (resolves to your local machine via /etc/hosts or DNS)
```

The browser sends an HTTP request to `my-cicd-one.com` which resolves to `localhost` or your machine's IP.

---

### **Step 2: Host Port → K8s Node Port Mapping (80 → 30080)**

From `create_cluster.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cicd-one
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 80
    protocol: TCP
```

**What happens:**
- KinD cluster's **container port 30080** is mapped to your **host machine's port 80**
- All traffic arriving on host port 80 gets forwarded to the Kubernetes node's port 30080
- This is where Traefik's Gateway API controller is listening

---

### **Step 3: Traefik Gateway Controller Processing**

Traefik (running in the `traefik` namespace) acts as the **Gateway API ingress controller**. It:

1. **Listens on port 30080** (internally on the node)
2. **Reads the HTTPRoute resource** (`python-app-route`)
3. **Inspects the request**:
   - Hostname: `my-cicd-one.com` ✓ Matches HTTPRoute hostname
   - Path: `/api/v1/info` ✓ Matches PathPrefix: `/`
4. **Routes to the backend service**

---

### **Step 4: HTTPRoute → Service Routing**

Your HTTPRoute definition (`K8s-Files/httproute.yaml`):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: python-app-route
spec:
  parentRefs:
  - name: traefik-gateway
    namespace: traefik
  hostnames:
  - "my-cicd-one.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: python-app-service
      port: 8080
```

**What happens:**
- Traefik forwards the request to `python-app-service` on **port 8080**
- This is a Kubernetes Service, not a pod directly

---

### **Step 5: Service → Pod Port Translation (8080 → 5000)**

Your Service definition (`K8s-Files/service.yaml`):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: python-app-service
spec:
  selector:
    app: python-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 5000
```

**What happens:**
- Service receives traffic on **port 8080** (from Traefik/HTTPRoute)
- Service translates it to **port 5000** (the actual pod port)
- Service **load balances** the traffic across available pods matching `app: python-app`

---

### **Step 6: Pod Distribution & Flask Processing**

Your Deployment (`K8s-Files/deployment.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: python-app
  template:
    metadata:
      labels:
        app: python-app
    spec:
      containers:
      - name: python-app
        image: shawnprac/cicd_one_python_app:v1
        ports:
        - containerPort: 5000
```

**What happens:**
- **2 replicas** of the pod are running
- Each pod runs a Flask application container listening on **port 5000**
- The Service round-robins requests between the two pods
- Traffic arrives at one of the pods

---

### **Step 7: Flask Application Processing**

Your Flask app (`python_src/src/app.py`):

```python
from flask import Flask, jsonify
import datetime, socket

app = Flask(__name__)

@app.route('/api/v1/info')
def info():
    return jsonify({
        'time': datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        'hostname': socket.gethostname(),
        'env': 'dev'
    })

@app.route('/api/v1/healthz')
def health():
    return jsonify({'status': 'up'}), 200

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=5000)
```

**What happens:**
- Flask receives the request for `/api/v1/info`
- Calls the `info()` handler
- Returns JSON with:
  - Current timestamp
  - Pod hostname (e.g., `python-app-abc123xyz`)
  - Environment (`dev`)

---

### **Step 8: Response Journey (Reverse Path)**

The response travels back through the layers in reverse:

```
Flask Pod (port 5000) 
  → Service (translates 5000 → 8080)
  → Traefik Gateway (via HTTPRoute)
  → Node container port 30080
  → Host port 80
  → Browser
```

---

## Port Reference Table

| Layer | Port | Role | Direction |
|-------|------|------|-----------|
| **Host Machine** | `80` | External entry point | Inbound (browser) |
| **KinD Mapping** | `30080` | Container-to-host bridge | Internal |
| **Traefik Gateway** | `30080` | Gateway API listener | Listener |
| **Service (Virtual)** | `8080` | K8s load balancer | Abstraction |
| **Container (Flask)** | `5000` | Application listener | Actual app |

---

## Key Insights

1. **Port 5000 is never exposed externally** 
   - It's internal to the pod and only accessible within the cluster
   - The Service abstracts the pod's actual port

2. **Service port (8080) is virtual**
   - It's a Kubernetes abstraction for service discovery and load balancing
   - Multiple pods can be accessed through a single Service IP:port combination

3. **Traefik uses Gateway API (advanced)**
   - **HTTPRoute** provides Layer 7 (application layer) routing
   - It matches on hostnames, paths, headers, methods, etc., not just ports
   - More powerful than traditional Ingress resources

4. **Each pod gets a unique IP**
   - The Service abstracts the internal pod IPs
   - It distributes requests using round-robin load balancing

5. **Port translation occurs at the Service layer**
   - `port: 8080` is what external clients call
   - `targetPort: 5000` is what the container actually listens on

---

## ArgoCD Access (Different Flow)

ArgoCD uses traditional **Ingress** instead of Gateway API:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
spec:
  ingressClassName: traefik
  rules:
  - host: "argocd.my-cicd-one.com"
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

**Flow:**
```
Browser: argocd.my-cicd-one.com
  → Host Port 80
  → Node Port 30080
  → Traefik (matches hostname)
  → argocd-server Service (port 80)
  → ArgoCD application
```

This is simpler because ArgoCD's service already exposes port 80 directly.

---

## Common Questions

**Q: Why does the Service use port 8080 instead of 5000?**
A: It's a design choice to separate concerns. Services typically use convenient high-numbered ports (8080, 9000, etc.) as an abstraction layer, while applications listen on their default ports (5000 for Flask default, 3000 for Node.js, etc.).

**Q: What if I have 5 replicas instead of 2?**
A: The Service automatically load balances across all 5 pods. Each pod still listens on port 5000, and the Service abstracts them all behind a single port 8080 endpoint.

**Q: Can I access the pods directly on port 5000?**
A: Only from within the Kubernetes cluster. External access is only through the Service (port 8080) and then to the actual container port (5000).

**Q: What happens if a pod crashes?**
A: The Deployment controller automatically restarts it, and the Service stops routing traffic to it until it's ready again.

**Q: Why use HTTPRoute instead of Ingress?**
A: Gateway API (HTTPRoute) provides more advanced features like:
- Path matching with more options
- Header-based routing
- Weighted traffic splitting
- Better multi-namespace support
- Future-proofed for advanced routing scenarios

---

## Summary

The request flow is a careful choreography of port translations across multiple layers:

1. **Browser** sends to host port 80
2. **KinD** maps it to node port 30080
3. **Traefik** listens on 30080 and reads HTTPRoute rules
4. **HTTPRoute** specifies backend Service on port 8080
5. **Service** abstracts pods and translates 8080 ↔ 5000
6. **Flask** processes the request on port 5000
7. **Response** flows back through the same layers in reverse

Each layer has a specific responsibility, and this separation of concerns makes Kubernetes highly scalable and maintainable.
