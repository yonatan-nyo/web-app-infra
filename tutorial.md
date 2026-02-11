# Complete Multi-Cluster ArgoCD Setup Tutorial

A step-by-step guide to set up a complete GitOps environment with ArgoCD managing multiple Kubernetes clusters.

**Prerequisites:**

- GitHub repository: `web-app-infra` (ready with Helm charts)
- Docker Hub account (for storing container images)
- Git installed and configured
- Basic understanding of Kubernetes concepts

---

## Part 1: Environment Setup

### Step 1: Start Colima (Docker Runtime)

Colima provides a Docker-compatible container runtime on macOS.

```bash
# Start Colima with Docker runtime
colima start --runtime docker --cpu 4 --memory 6

# Verify Colima is running
colima status

# Verify Docker is accessible
docker ps
```

**Expected Output:**

```
Colima is running using macOS Virtualization.Framework
arch: aarch64
runtime: docker
mountType: virtiofs
```

### Step 1b: Install Required CLI Tools

Install Cilium CLI and Helm for managing Kubernetes resources.

**Install Cilium CLI:**

Cilium is a modern CNI (Container Network Interface) that provides networking, security, and observability for Kubernetes. We'll use it instead of the default Flannel CNI.

```bash
# Install Cilium CLI using Homebrew
brew install cilium-cli

# Verify installation
cilium version --client
```

**Expected Output:**

```
cilium-cli: v0.19.0
```

**Install Helm (if not already installed):**

Helm is a package manager for Kubernetes that simplifies application deployment.

```bash
# Check if Helm is installed
which helm

# If not installed, install via Homebrew
brew install helm

# Verify installation
helm version
```

---

## Part 2: Create Kubernetes Clusters with Cilium

### Step 2: Create Control Plane Cluster (my-lab)

This cluster will host ArgoCD and manage deployments. We'll create it without the default CNI to use Cilium instead.

```bash
# Create the control plane cluster without CNI
minikube start \
  --profile=my-lab \
  --cpus=2 \
  --memory=3g \
  --cni=false \
  --driver=docker

# Verify cluster creation
minikube profile list
kubectl config get-contexts

# Switch to the cluster
kubectl config use-context my-lab

# Verify cluster is working
kubectl get nodes
```

**📝 Note about `--cni=false`:**

Using `--cni=false` creates a cluster without any CNI, allowing us to install Cilium with full features including Hubble UI and advanced observability.

### Step 3: Create Remote Managed Cluster (my-lab-2)

This cluster will be managed by ArgoCD from my-lab. Also created without the default CNI.

```bash
# Create the second cluster without CNI
minikube start \
  --profile=my-lab-2 \
  --cpus=2 \
  --memory=3g \
  --cni=false \
  --driver=docker

# List all clusters
minikube profile list

# Verify you can switch contexts
kubectl config use-context my-lab-2
kubectl get nodes

# Switch back to control plane
kubectl config use-context my-lab
```

### Step 4: Get Cluster Information

**📝 Note:** Minikube clusters run on the same Docker network by default, making multi-cluster communication easier.

```bash
# Get cluster IPs (save these for ArgoCD configuration)
MYLAB_IP=$(minikube ip -p my-lab)
MYLAB2_IP=$(minikube ip -p my-lab-2)

echo "my-lab IP: $MYLAB_IP"
echo "my-lab-2 IP: $MYLAB2_IP"

# Verify both clusters are running
minikube profile list
```

**Expected Output:**

```
|-----------|-----------|---------|--------------|------|---------|---------|-------|--------|
|  Profile  | VM Driver | Runtime |      IP      | Port | Version | Status  | Nodes | Active |
|-----------|-----------|---------|--------------|------|---------|---------|-------|--------|
| my-lab    | docker    | docker  | 192.168.49.2 | 8443 | v1.33.0 | Running |     1 | *      |
| my-lab-2  | docker    | docker  | 192.168.58.2 | 8443 | v1.33.0 | Running |     1 |        |
|-----------|-----------|---------|--------------|------|---------|---------|-------|--------|
```

**📝 Note:** Save the my-lab-2 IP address - you'll need it when adding the cluster to ArgoCD. Both clusters use Docker's bridge network and can communicate with each other.

### Step 5: Install Cilium with Hubble UI

**Install Cilium 1.19.0 using Cilium CLI:**

The Cilium CLI will install Cilium with all features including Hubble UI, Hubble Relay, and advanced networking capabilities.

```bash
# Install on my-lab (Control Plane)
kubectl config use-context my-lab

# Install Cilium 1.19.0 with Hubble UI
cilium install --version 1.19.0

# Wait for Cilium to be ready
cilium status --wait

# might be installing. if u wanna see more info
kubectl get pods -n kube-system -l k8s-app=cilium -o wide
# ge the name of the pod, e.g. the name is cilium-btljq REPLACE cilium-btljq with your pod name
kubectl describe pod -n kube-system cilium-btljq | tail -50

# Enable hubble UI
cilium hubble enable
cilium hubble enable --ui

# Check all pods are running
kubectl get pods -n kube-system | grep -E 'cilium|hubble'

# Install on my-lab-2 (Remote Cluster)
kubectl config use-context my-lab-2

# Install Cilium 1.19.0 with Hubble UI
cilium install --version 1.19.0

# Wait for Cilium to be ready
cilium status --wait

# Enable hubble UI
cilium hubble enable
cilium hubble enable --ui

# Check all pods are running
kubectl get pods -n kube-system | grep -E 'cilium|hubble'

# Switch back to control plane
kubectl config use-context my-lab
```

**Expected Pods (with Hubble UI):**

```
NAME                              READY   STATUS    RESTARTS   AGE
cilium-xxxxx                      1/1     Running   0          2m
cilium-operator-xxxxxxx           1/1     Running   0          2m
hubble-relay-xxxxxxx              1/1     Running   0          2m
hubble-ui-xxxxxxx                 2/2     Running   0          2m
```

**Expected Cilium Status:**

```bash
$ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

DaemonSet              cilium             Desired: 1, Ready: 1/1, Available: 1/1
Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-relay       Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-ui          Desired: 1, Ready: 1/1, Available: 1/1
```

#### Verify Cilium Features

```bash
# Check Cilium status with all features
cilium status --wait

# Verify connectivity between nodes (optional, takes 5-10 minutes)
cilium connectivity test

# Test Hubble flow observation
kubectl exec -n kube-system ds/cilium -- cilium-dbg hubble observe --last 10
```

**� Why Cilium?**

- **Advanced Networking:** eBPF-based networking for better performance
- **Security:** Network policies and encryption at the kernel level
- **Observability:** Built-in network monitoring with Hubble
- **Service Mesh:** Native support for service mesh features
- **Multi-Cluster:** Better support for multi-cluster networking

**🔧 Troubleshooting Cilium:**

1. **Check pod status:**

   ```bash
   kubectl get pods -n kube-system -l k8s-app=cilium
   ```

2. **View Cilium agent logs:**

   ```bash
   kubectl logs -n kube-system ds/cilium --tail=50
   ```

3. **Restart Cilium if needed:**
   ```bash
   kubectl rollout restart ds/cilium -n kube-system
   kubectl wait --for=condition=Ready pods -l k8s-app=cilium -n kube-system --timeout=120s
   ```

**📝 Understanding How Hubble Works:**

Hubble is Cilium's observability layer. Here's how the components work together:

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐                                        │
│  │  Hubble UI   │ ◄──── Web Browser (you)                │
│  │  (Pod)       │       http://localhost:12000           │
│  └──────┬───────┘                                        │
│         │                                                │
│         ▼                                                │
│  ┌──────────────┐                                        │
│  │ Hubble Relay │ ◄──── Aggregates flows from agents    │
│  │  (Pod)       │                                        │
│  └──────┬───────┘                                        │
│         │                                                │
│    ┌────┴────┬────────┬────────┐                        │
│    ▼         ▼        ▼        ▼                        │
│ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐            │
│ │Cilium  │ │Cilium  │ │Cilium  │ │Cilium  │            │
│ │Agent 1 │ │Agent 2 │ │Agent 3 │ │Agent N │            │
│ │(Pod)   │ │(Pod)   │ │(Pod)   │ │(Pod)   │            │
│ └────────┘ └────────┘ └────────┘ └────────┘            │
│      │          │          │          │                 │
│      └──────────┴──────────┴──────────┘                 │
│                    │                                     │
│        Collect network flows from                       │
│        all pods in the cluster                          │
└─────────────────────────────────────────────────────────┘
```

**Key Points:**

1. **Cilium Agents** - Run on each node, collect network flow data from all pods
2. **Hubble Relay** - Aggregates flows from all Cilium agents in the cluster
3. **Hubble UI** - Web interface that queries Hubble Relay to visualize flows

**Important:** Each cluster has its own independent Hubble stack:

- ✅ **my-lab** has: Cilium agents → Hubble Relay → Hubble UI (port 12000)
- ✅ **my-lab-2** has: Cilium agents → Hubble Relay → Hubble UI (port 12001)

**Can I view both clusters in one Hubble UI?**

**Short answer:** No, not with the default setup. Each Hubble UI only shows traffic from its own cluster.

**Why?** Each Hubble Relay only connects to Cilium agents in its own cluster. The clusters are isolated.

**Options for viewing both clusters:**

1. **Two browser tabs (simplest)** - Open both UIs side-by-side:

   ```bash
   # Terminal 1: Port-forward my-lab Hubble UI
   kubectl config use-context my-lab
   kubectl port-forward -n kube-system svc/hubble-ui 12000:80
   # Keep this terminal open, then open: http://localhost:12000

   # Terminal 2: Port-forward my-lab-2 Hubble UI
   kubectl config use-context my-lab-2
   kubectl port-forward -n kube-system svc/hubble-ui 12001:80
   # Keep this terminal open, then open: http://localhost:12001
   ```

   Now open both URLs in separate browser tabs:
   - http://localhost:12000 (my-lab)
   - http://localhost:12001 (my-lab-2)

2. **Use Hubble CLI** - Switch contexts and view flows:

   ```bash
   # View my-lab flows
   kubectl config use-context my-lab
   kubectl exec -n kube-system ds/cilium -- cilium hubble observe --last 20

   # View my-lab-2 flows
   kubectl config use-context my-lab-2
   kubectl exec -n kube-system ds/cilium -- cilium hubble observe --last 20
   ```

3. **ClusterMesh (advanced)** - Connect clusters at the network level:
   - Enables true multi-cluster networking
   - One Hubble Relay can observe all clusters
   - More complex setup (beyond this tutorial)
   - Docs: https://docs.cilium.io/en/stable/network/clustermesh/

**For this tutorial, use two browser tabs.** It's simple and works well for observing both clusters.

### Step 5b: Access Hubble UI for Both Clusters

**Access Hubble UI from both clusters:**

```bash
# Terminal 1: Port-forward my-lab Hubble UI
kubectl config use-context my-lab
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
# Open in browser: http://localhost:12000

# Terminal 2: Port-forward my-lab-2 Hubble UI (different port!)
kubectl config use-context my-lab-2
kubectl port-forward -n kube-system svc/hubble-ui 12001:80
# Open in browser: http://localhost:12001
```

**Now you can observe both clusters simultaneously:**

- 🌐 **http://localhost:12000** - Shows my-lab cluster traffic
- 🌐 **http://localhost:12001** - Shows my-lab-2 cluster traffic

**What you'll see in Hubble UI:**

- 🌐 Service map with traffic flows between pods
- 📊 Real-time network metrics (HTTP, DNS, TCP)
- 🔒 Network policy verdicts (allowed/denied connections)
- 🔍 Layer 7 protocol visibility
- ⚠️ Dropped packets and security events

**Why enable Hubble UI on both clusters?**

- **Independent observability** - Each cluster's network activity is isolated
- **Debugging multi-cluster issues** - See if traffic reaches cluster boundaries
- **Compare behavior** - Observe how the same application behaves in different clusters
- **Production monitoring** - If my-lab-2 represents prod, you want dedicated monitoring

**📝 Note:** If you only care about observing the control plane (my-lab), you only need to enable Hubble UI there. For full multi-cluster visibility, enable on both.

**Use Hubble CLI for observing network flows:**

You can also observe flows via CLI from Cilium agents:

```bash
# Watch live network flows (all protocols)
kubectl exec -n kube-system ds/cilium -- cilium hubble observe --follow

# Filter by namespace
kubectl exec -n kube-system ds/cilium -- cilium hubble observe --namespace default

# Watch DNS queries
kubectl exec -n kube-system ds/cilium -- cilium hubble observe --protocol DNS

# Check dropped packets
kubectl exec -n kube-system ds/cilium -- cilium hubble observe --verdict DROPPED

# See network policy verdicts
kubectl exec -n kube-system ds/cilium -- cilium hubble observe --type policy-verdict

# Show last 100 flows
kubectl exec -n kube-system ds/cilium -- cilium hubble observe --last 100

# JSON output for processing
kubectl exec -n kube-system ds/cilium -- cilium hubble observe -o json
```

**Observing flows from specific clusters via CLI:**

```bash
# Observe my-lab cluster flows
kubectl config use-context my-lab
kubectl exec -n kube-system ds/cilium -- cilium hubble observe --last 20

# Observe my-lab-2 cluster flows
kubectl config use-context my-lab-2
kubectl exec -n kube-system ds/cilium -- cilium hubble observe --last 20
```

**📝 Note:** With `cilium install`, you get Hubble UI and Hubble Relay out of the box for full observability!

### Step 5c: Test Basic Cilium Features

**Check Cilium Configuration:**

```bash
# View Cilium ConfigMap
kubectl get configmap cilium-config -n kube-system -o yaml

# Check kube-proxy replacement status
kubectl exec -n kube-system ds/cilium -- cilium status | grep KubeProxyReplacement
```

**Check eBPF Maps:**

```bash
# View eBPF program statistics
kubectl exec -n kube-system ds/cilium -- cilium bpf stats

# View service load balancing maps
kubectl exec -n kube-system ds/cilium -- cilium bpf lb list

# View connection tracking entries
kubectl exec -n kube-system ds/cilium -- cilium bpf ct list global
```

---

## Part 3: Build and Push Application Images

### Step 6: Build Frontend Image

```bash
# Navigate to frontend code directory
cd web-app-code

# Build the frontend image
docker build -t <your-dockerhub-username>/my-app:v1.0.0 .

# Example: docker build -t skimpjr/my-app:v1.0.0 .

# Verify image was built
docker images | grep my-app
```

### Step 7: Build Backend Image

```bash
# Navigate to backend code directory
cd ../web-app-server

# Build the backend image
docker build -t <your-dockerhub-username>/my-server:v1.0.0 .

# Example: docker build -t skimpjr/my-server:v1.0.0 .

# Verify image was built
docker images | grep my-server
```

### Step 8: Push Images to Docker Hub

```bash
# Login to Docker Hub
docker login
# Enter your username and password

# Push frontend image
docker push <your-dockerhub-username>/my-app:v1.0.0

# Push backend image
docker push <your-dockerhub-username>/my-server:v1.0.0

# Verify images on Docker Hub
# Visit: https://hub.docker.com/r/<your-username>/
```

### Step 9: Update Helm Values with Your Images

Update the Helm chart values to use your Docker Hub images:

**Edit `web-app-infra/charts/my-web-app/values/dev/frontend.yaml`:**

```yaml
image:
  repository: <your-dockerhub-username>/my-app
  tag: "v1.0.0"
```

**Edit `web-app-infra/charts/my-web-app/values/dev/server.yaml`:**

```yaml
image:
  repository: <your-dockerhub-username>/my-server
  tag: "v1.0.0"
```

**Commit and push changes:**

```bash
cd web-app-infra
git add charts/my-web-app/values/dev/
git commit -m "Update Docker Hub image references"
git push
```

---

## Part 4: Install and Configure ArgoCD

### Step 10: Install ArgoCD on Control Plane

```bash
# Make sure you're on the control plane cluster
kubectl config use-context my-lab

# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD pods to be ready (this may take 2-3 minutes)
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Verify all pods are running
kubectl get pods -n argocd
```

**Expected Output:** All pods should show `1/1 READY` and `Running` status.

### Step 11: Configure Permanent ArgoCD Access

Instead of port-forwarding every time, set up an Ingress for permanent access.

**Create `web-app-infra/argocd-config.yaml`:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cmd-params-cm
    app.kubernetes.io/part-of: argocd
data:
  # Run server without TLS
  server.insecure: "true"
  # Set base URL for routing through Ingress
  server.basehref: "/argocd"
  server.rootpath: "/argocd"
```

**Create `web-app-infra/argocd-ingress.yaml`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik
  rules:
    - host: localhost
      http:
        paths:
          - path: /argocd
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

**Apply the configuration:**

```bash
kubectl apply -f argocd-config.yaml
kubectl apply -f argocd-ingress.yaml

# Restart ArgoCD server to apply config
kubectl rollout restart deployment argocd-server -n argocd

# Wait for restart to complete
kubectl rollout status deployment argocd-server -n argocd --timeout=60s
```

### Step 12: Get ArgoCD Admin Password

```bash
# Retrieve the admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
```

**Save this password!** You'll need it to login.

### Step 13: Access ArgoCD UI

**Option 1: Port-Forward (Simpler, Always Works)**

If the ingress doesn't work (no ingress controller installed), use port-forward:

```bash
# Port-forward ArgoCD server (keep this terminal open)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open your browser and navigate to:

```
https://localhost:8080
```

**⚠️ Important:** Use **https** (not http). Your browser will show a security warning about self-signed certificate - click "Advanced" → "Proceed to localhost (unsafe)".

**Login credentials:**

- Username: `admin`
- Password: (the password from Step 12)

---

**Option 2: Ingress (Requires Ingress Controller)**

If you prefer using ingress, first enable minikube's ingress addon:

```bash
# Enable NGINX ingress addon
minikube addons enable ingress -p my-lab

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

Then navigate to:

```
http://localhost:8080/argocd
```

**📝 Recommendation:** Use **Option 1 (Port-Forward)** for this tutorial - it's simpler and doesn't require additional setup.

**📝 Note:** The rest of this tutorial assumes you're using **Option 1 (Port-Forward)**, so ArgoCD will be accessible at **https://localhost:8080**.

---

## Part 5: Deploy Applications to Control Plane

### Step 14: Create ArgoCD Application for Control Plane

**Create `web-app-infra/application.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-web-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/web-app-infra
    targetRevision: main
    path: charts/my-web-app
    helm:
      valueFiles:
        - values/dev/frontend.yaml
        - values/dev/server.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Apply the application:**

```bash
kubectl apply -f application.yaml

# Watch the application sync
kubectl get applications -n argocd -w
# Press Ctrl+C to stop watching
```

**Verify in ArgoCD UI:**

- Navigate to https://localhost:8080 (via port-forward from Step 13)
- You should see `my-web-app` application
- Status should be: **Synced** and **Healthy**

### Step 15: Verify Application Deployment

```bash
# Check pods are running
kubectl get pods

# Check services
kubectl get svc

# Check ingress
kubectl get ingress

# Test the application
curl http://localhost:8080
```

You should see your web application's frontend!

---

## Part 6: Add Remote Cluster to ArgoCD

### Step 16: Create ServiceAccount on Remote Cluster

Switch to the remote cluster and create the necessary credentials:

```bash
# Switch to my-lab-2
kubectl config use-context my-lab-2

# Create namespace for ArgoCD service account
kubectl create namespace argocd-manager

# Create service account
kubectl create serviceaccount argocd-manager -n argocd-manager

# Grant cluster-admin permissions
kubectl create clusterrolebinding argocd-manager-role \
  --clusterrole=cluster-admin \
  --serviceaccount=argocd-manager:argocd-manager
```

### Step 17: Generate Long-lived Token

Create a secret for the service account token:

**Create `token-secret.yaml` (temporary file, DO NOT COMMIT):**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-manager-token
  namespace: argocd-manager
  annotations:
    kubernetes.io/service-account.name: argocd-manager
type: kubernetes.io/service-account-token
```

```bash
# Apply the token secret
kubectl apply -f token-secret.yaml

# Wait a moment for token to be generated
sleep 5

# Extract the token
TOKEN=$(kubectl get secret argocd-manager-token -n argocd-manager -o jsonpath='{.data.token}' | base64 -d)

echo "Token: $TOKEN"
# Save this token - you'll need it in the next step!
```

### Step 18: Get Remote Cluster IP Address

You noted this earlier in Step 4, but let's verify it:

```bash
# Get the internal IP address of my-lab-2
MYLAB2_IP=$(minikube ip -p my-lab-2)
echo "my-lab-2 IP: $MYLAB2_IP"
# Example: 192.168.49.3
```

**Example output:**

```
my-lab-2 IP: 192.168.49.3
"IPAddress": "172.18.0.4"
```

### Step 19: Create Cluster Secret in ArgoCD

Switch back to the control plane and create the cluster secret:

```bash
# Switch back to control plane
kubectl config use-context my-lab
```

**Create `add-cluster-token.yaml` (DO NOT COMMIT - contains secrets):**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-lab-2-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: my-lab-2
  server: https://172.18.0.4:6443 # Use the IP from Step 18
  config: |
    {
      "bearerToken": "<PASTE_YOUR_TOKEN_HERE>",
      "tlsClientConfig": {
        "insecure": true
      }
    }
```

**Replace `<PASTE_YOUR_TOKEN_HERE>` with the token from Step 17, then apply:**

```bash
kubectl apply -f add-cluster-token.yaml

# Wait 10 seconds for ArgoCD to discover the cluster
sleep 10

# List clusters
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster
```

### Step 20: Verify Cluster Connection

**In ArgoCD UI:**

1. Navigate to https://localhost:8080 (via port-forward from Step 13)
2. Go to **Settings** → **Clusters**
3. You should see two clusters:
   - `in-cluster` (my-lab) - ✅ Successful
   - `my-lab-2` - ✅ Successful

**Via CLI:**

```bash
kubectl get applications -n argocd

# All should show connection status
```

---

## Part 7: Deploy Application to Remote Cluster

### Step 21: Create Application for Remote Cluster

**Create `web-app-infra/application-lab-2.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-web-app-lab-2
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/web-app-infra
    targetRevision: main
    path: charts/my-web-app
    helm:
      valueFiles:
        - values/dev/frontend.yaml
        - values/dev/server.yaml
  destination:
    server: https://172.18.0.4:6443 # Use the IP from Step 18
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Apply the application:**

```bash
kubectl apply -f application-lab-2.yaml

# Watch the sync
kubectl get applications -n argocd
```

### Step 22: Verify Remote Deployment

```bash
# Check resources on my-lab-2 cluster
kubectl get pods --context my-lab-2
kubectl get svc --context my-lab-2
kubectl get ingress --context my-lab-2

# Test the application on my-lab-2
curl http://localhost:9080
```

You should see the application running on the remote cluster!

---

## Part 8: Update .gitignore and Commit

### Step 23: Protect Sensitive Files

**Update `web-app-infra/.gitignore`:**

```gitignore
# Cluster secrets (contain bearer tokens)
add-cluster.yaml
add-cluster-token.yaml
token-secret.yaml

# Generated Cilium manifests (Helm values should be committed)
cilium-manifests.yaml

# Any file with credentials
*secret*.yaml
*token*.yaml
```

### Step 24: Commit Configuration Files

```bash
cd web-app-infra

# Add only safe files
git add argocd-config.yaml
git add argocd-ingress.yaml
git add application.yaml
git add application-lab-2.yaml
git add .gitignore

# Commit
git commit -m "Add ArgoCD multi-cluster setup with advanced Cilium CNI

- Cilium CNI with Hubble, L7 visibility, bandwidth management, eBPF optimizations
- Hubble UI ingress for network observability
- ArgoCD permanent access via Ingress
- Application for control plane (my-lab)
- Application for remote cluster (my-lab-2)
- IPv6 support and network policy enforcement enabled
- Updated .gitignore to protect secrets"

# Push to GitHub
git push
```

---

## Part 9: Verification and Testing

### Step 25: Complete System Verification

**Check all clusters:**

```bash
# Control plane cluster
kubectl config use-context my-lab
kubectl get pods -A

# Remote cluster
kubectl config use-context my-lab-2
kubectl get pods -A
```

**Check ArgoCD Applications:**

1. Open https://localhost:8080 (via port-forward)
2. You should see two applications:
   - `my-web-app` → ✅ Synced & Healthy (my-lab)
   - `my-web-app-lab-2` → ✅ Synced & Healthy (my-lab-2)

**Test Applications:**

```bash
# Test my-lab application
curl http://localhost:8080

# Test my-lab-2 application
curl http://localhost:9080
```

Both should return your web application!

---

## Architecture Overview

Here's what you've built:

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Network: minikube                  │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  my-lab (Control Plane)                              │    │
│  │  External: localhost:8080                            │    │
│  │  ┌──────────────────────────────────────┐            │    │
│  │  │  ArgoCD (namespace: argocd)           │            │    │
│  │  │  - Manages both clusters              │            │    │
│  │  │  - UI: https://localhost:8080         │            │    │
│  │  └──────────────────────────────────────┘            │    │
│  │  ┌──────────────────────────────────────┐            │    │
│  │  │  Web App (namespace: default)         │            │    │
│  │  │  - Frontend: Nginx                    │            │    │
│  │  │  - Backend: Node.js                   │            │    │
│  │  │  - Access: http://localhost:8080      │            │    │
│  │  └──────────────────────────────────────┘            │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  my-lab-2 (Remote Managed Cluster)                   │    │
│  │  External: localhost:9080                            │    │
│  │  ┌──────────────────────────────────────┐            │    │
│  │  │  ServiceAccount: argocd-manager       │            │    │
│  │  │  - Allows ArgoCD to deploy here       │            │    │
│  │  └──────────────────────────────────────┘            │    │
│  │  ┌──────────────────────────────────────┐            │    │
│  │  │  Web App (namespace: default)         │            │    │
│  │  │  - Frontend: Nginx                    │            │    │
│  │  │  - Backend: Node.js                   │            │    │
│  │  │  - Access: http://localhost:9080      │            │    │
│  │  └──────────────────────────────────────┘            │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Common Commands Reference

### Cluster Management

```bash
# List all clusters
minikube profile list

# Switch between clusters
kubectl config use-context my-lab
kubectl config use-context my-lab-2

# Delete a cluster
minikube delete -p my-lab
```

### ArgoCD Management

```bash
# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Then open: https://localhost:8080

# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

# List applications
kubectl get applications -n argocd

# Force sync an application
kubectl patch app my-web-app -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# List connected clusters
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster
```

### Cilium & Hubble Management

```bash
# Check Cilium status
cilium status

# Run connectivity test
cilium connectivity test

# View Cilium configuration
cilium config view

# Access Hubble UI
kubectl port-forward -n kube-system svc/hubble-ui 12000:80

# Watch network flows in real-time
cilium hubble observe

# Filter flows by namespace
cilium hubble observe --namespace default

# Watch DNS queries
cilium hubble observe --type trace:to-endpoint --protocol DNS

# Watch HTTP traffic (Layer 7)
cilium hubble observe --protocol http

# Check dropped packets
cilium hubble observe --verdict DROPPED

# View network policy decisions
cilium hubble observe --type policy-verdict

# Check bandwidth manager status
kubectl exec -n kube-system ds/cilium -- cilium status | grep BandwidthManager

# View eBPF statistics
kubectl exec -n kube-system ds/cilium -- cilium bpf stats

# List service load balancing
kubectl exec -n kube-system ds/cilium -- cilium bpf lb list

# View Cilium metrics
kubectl exec -n kube-system ds/cilium -- curl localhost:9962/metrics
```

### Debugging

```bash
# Check application status
kubectl describe application my-web-app -n argocd

# Check pods in all namespaces
kubectl get pods -A

# Check cluster IP (if connection fails)
MYLAB2_IP=$(minikube ip -p my-lab-2)
echo "my-lab-2 IP: $MYLAB2_IP"

# Test connectivity from ArgoCD pod (replace <IP> with actual IP)
kubectl exec -n argocd deployment/argocd-application-controller -- wget -O- https://<IP>:6443/version --no-check-certificate
```

---

## Troubleshooting

### Problem: Cluster Connection Failed

**Symptoms:** ArgoCD shows "connection refused" or "no route to host"

**Solution:**

```bash
# 1. Verify cluster is running
minikube profile list

# 2. Get cluster IP address
MYLAB2_IP=$(minikube ip -p my-lab-2)
echo "my-lab-2 IP: $MYLAB2_IP"

# 3. Update cluster secret with new IP
kubectl patch secret my-lab-2-cluster -n argocd -p '{"stringData":{"server":"https://'$MYLAB2_IP':6443"}}'

# 4. Update application-lab-2.yaml with new IP and reapply
kubectl apply -f application-lab-2.yaml
```

### Problem: Application Not Syncing

**Symptoms:** Application shows "OutOfSync" status

**Solution:**

```bash
# Force a hard refresh
kubectl patch app my-web-app -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# Check application details
kubectl describe application my-web-app -n argocd
```

### Problem: Can't Access ArgoCD UI

**Symptoms:** Browser shows "Application not available" or connection refused

**Solution:**

Use port-forward instead of ingress:

```bash
# Port-forward ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access via: https://localhost:8080 (note: https, not http)
```

**If port-forward also fails, check ArgoCD pods:**

```bash
# Check ArgoCD pods are running
kubectl get pods -n argocd

# All pods should be Running and Ready
# If not, check logs:
kubectl logs -n argocd deployment/argocd-server
```

---

## Next Steps

Now that you have a working multi-cluster GitOps setup:

1. **Add More Applications:** Create more Application manifests for different services
2. **Implement App of Apps Pattern:** Use ArgoCD's App of Apps pattern for managing multiple applications
3. **Add RBAC:** Configure role-based access control for team members
4. **Set Up Monitoring:** Add Prometheus and Grafana for observability
5. **Configure Alerts:** Set up ArgoCD notifications for Slack/email
6. **Production Hardening:** Remove `insecure: true` and set up proper TLS

---

## Fresh Start (Reset Environment)

If you want to start over from scratch while keeping all tools installed (Colima, minikube, kubectl, Docker, etc.):

### Quick Reset (Recommended)

```bash
# 1. Delete both minikube clusters
minikube delete -p my-lab
minikube delete -p my-lab-2

# 2. Verify clusters are gone
minikube profile list
# Should show: "No minikube profile found"

# 3. Verify kubectl contexts are cleaned up
kubectl config get-contexts
# my-lab and my-lab-2 contexts should be removed

# Done! Now you can restart from Step 2 (Create Kubernetes Clusters with Cilium)
```

### Deep Clean (If Quick Reset Doesn't Work)

Sometimes Docker containers or networks persist. Use this for a complete reset:

```bash
# 1. Stop all minikube profiles
minikube stop --all

# 2. Delete all profiles
minikube delete --all --purge

# 3. Remove minikube Docker containers (if any remain)
docker ps -a --filter "name=minikube" --format "{{.Names}}" | xargs -r docker stop
docker ps -a --filter "name=minikube" --format "{{.Names}}" | xargs -r docker rm -f

# 4. Remove minikube networks
docker network ls --filter "name=minikube" --format "{{.Name}}" | xargs -r docker network rm

# 5. Verify cleanup
docker ps -a | grep minikube  # Should return nothing
docker network ls | grep minikube  # Should return nothing

# 6. Clean kubectl config (remove stale contexts)
kubectl config delete-context my-lab 2>/dev/null || true
kubectl config delete-context my-lab-2 2>/dev/null || true

# 6. Verify kubectl config is clean
kubectl config get-contexts

# Done! Now restart from Step 2 (Create Kubernetes Clusters with Cilium)
```

### Reset with Colima Restart

If you're experiencing Docker or network issues, restart Colima:

```bash
# 1. Delete clusters first
minikube delete -p my-lab
minikube delete -p my-lab-2

# 2. Stop Colima
colima stop

# 3. Start Colima fresh
colima start --runtime docker --cpu 4 --memory 6

# 4. Verify Docker is working
docker ps

# Done! Now restart from Step 2 (Create Kubernetes Clusters with Cilium)
```

### Remove Docker Images (Optional)

If you want to test the full build process again:

```bash
# List your application images
docker images | grep -E "my-app|my-server"

# Remove specific images
docker rmi <your-dockerhub-username>/my-app:v1.0.0
docker rmi <your-dockerhub-username>/my-server:v1.0.0

# Or remove all unused images (be careful!)
docker image prune -a

# Now restart from Step 6 (Build and Push Application Images)
```

### Clean Up Local Files

Remove temporary files created during setup:

```bash
cd web-app-infra

# Remove temporary secret files (if you created them)
rm -f add-cluster-token.yaml
rm -f token-secret.yaml

# Check git status to ensure no secrets are staged
git status

# Reset any unstaged changes
git checkout .
```

### What Gets Preserved

After a fresh start reset, you keep:

- ✅ Colima installation
- ✅ minikube installation
- ✅ kubectl installation
- ✅ Cilium CLI
- ✅ Docker CLI
- ✅ Git repository and commits
- ✅ Docker Hub images (unless you explicitly remove them)
- ✅ All local code files

### What Gets Removed

After a fresh start reset:

- ❌ minikube clusters (my-lab, my-lab-2)
- ❌ ArgoCD installation
- ❌ Deployed applications
- ❌ Kubernetes contexts
- ❌ Docker networks created by minikube
- ❌ Cluster secrets and tokens

### Verification After Reset

Before starting over, verify everything is clean:

```bash
# Should return no clusters
minikube profile list

# Should only show Docker default networks (bridge, host, none)
docker network ls

# Should show no minikube containers
docker ps -a | grep minikube

# Colima should still be running
colima status

# Docker should be accessible
docker ps
```

If all checks pass, you're ready to start fresh from **Step 2: Create Control Plane Cluster with Cilium**! 🎉

---

## Complete Teardown (Uninstall Everything)

**⚠️ Warning:** This removes clusters AND stops all Docker/Colima services. Use "Fresh Start" above if you just want to reset.

### Remove Clusters and Stop Docker

```bash
# Delete both clusters
minikube delete -p my-lab
minikube delete -p my-lab-2

# Stop Colima (stops Docker runtime)
colima stop
```

### Complete Removal (including Colima VM)

If you want to completely remove Colima and start from scratch:

```bash
# Delete both clusters first
minikube delete -p my-lab
minikube delete -p my-lab-2

# Stop and delete Colima VM completely
colima delete

# This removes:
# - Colima VM
# - All Docker containers, images, volumes, networks
# - Docker daemon configuration
```

**After `colima delete`:**

- All Docker data is lost (images, containers, volumes)
- You need to run `colima start` again to use Docker
- Colima settings reset to defaults

### Uninstall Tools (Complete Removal)

To completely remove all tools from your system:

```bash
# Uninstall minikube (macOS with Homebrew)
brew uninstall minikube

# Uninstall Colima
brew uninstall colima

# Uninstall Cilium CLI
brew uninstall cilium-cli

# Uninstall kubectl (if installed via Homebrew)
brew uninstall kubectl

# Remove Docker config directory (optional)
rm -rf ~/.docker

# Remove kubectl config (optional - removes ALL cluster configs)
rm -rf ~/.kube
```

**⚠️ Be Careful:** Removing `~/.kube` will delete configurations for ALL Kubernetes clusters, not just this lab!

---

## Summary

**What You've Accomplished:**

✅ Set up Colima Docker runtime  
✅ Installed Cilium CNI with advanced features:

- 🔭 **Hubble** - Network observability with UI
- 🔒 **Layer 7 visibility** - HTTP/DNS traffic monitoring
- 🛡️ **Network policies** - Security policy enforcement
- 📊 **Bandwidth management** - BBR congestion control
- ⚡ **eBPF optimizations** - kube-proxy replacement, host routing
- 🌐 **IPv6 support** - Dual-stack networking
- 📈 **Prometheus metrics** - Full observability stack

✅ Created two minikube Kubernetes clusters with Cilium  
✅ Built and pushed Docker images to Docker Hub  
✅ Installed ArgoCD with permanent UI access  
✅ Deployed applications to control plane cluster  
✅ Connected remote cluster to ArgoCD  
✅ Deployed applications to remote cluster  
✅ Configured GitOps workflow with automated sync

**Key Technology Benefits:**

- 📦 **GitOps:** Single source of truth (Git repository)
- 🔄 **ArgoCD:** Automated deployments and self-healing
- 🔍 **Audit Trail:** Full history of all changes
- 🎯 **Declarative:** Infrastructure as Code
- 🌍 **Multi-Cluster:** Centralized management from single control plane
- ⚡ **eBPF:** Kernel-level networking performance
- 🔭 **Observability:** Deep network visibility with Hubble
- 🛡️ **Security:** Advanced network policies and traffic control

**Access Points:**

- 🎨 ArgoCD UI: https://localhost:8080 (via port-forward)
- 🔭 Hubble UI (my-lab): http://localhost:12000 (via port-forward)
- 🔭 Hubble UI (my-lab-2): http://localhost:12001 (via port-forward)
- 🌐 my-lab app: http://localhost:8080
- 🌐 my-lab-2 app: http://localhost:9080
- 🔭 Hubble UI: http://localhost:8080/hubble (or port-forward to :12000)
- 🌐 my-lab app: http://localhost:8080
- 🌐 my-lab-2 app: http://localhost:9080

**Happy GitOps! 🚀**
