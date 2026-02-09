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

---

## Part 2: Create Kubernetes Clusters

### Step 2: Create Control Plane Cluster (my-lab)

This cluster will host ArgoCD and manage deployments.

```bash
# Create the control plane cluster with load balancer port mapping
k3d cluster create my-lab -p "8080:80@loadbalancer"

# Verify cluster creation
k3d cluster list
kubectl config get-contexts

# Switch to the cluster
kubectl config use-context k3d-my-lab

# Verify cluster is working
kubectl get nodes
```

### Step 3: Create Remote Managed Cluster (my-lab-2)

This cluster will be managed by ArgoCD from my-lab.

```bash
# Create the second cluster on a different port
k3d cluster create my-lab-2 -p "9080:80@loadbalancer"

# List all clusters
k3d cluster list

# Verify you can switch contexts
kubectl config use-context k3d-my-lab-2
kubectl get nodes

# Switch back to control plane
kubectl config use-context k3d-my-lab
```

### Step 4: Connect Clusters on Same Network

**⚠️ CRITICAL:** For ArgoCD to communicate with my-lab-2, both clusters must share a Docker network.

```bash
# Connect my-lab-2 to my-lab's network
docker network connect k3d-my-lab k3d-my-lab-2-server-0

# Verify network connectivity
docker network inspect k3d-my-lab | grep my-lab-2-server-0

# Get my-lab-2's internal IP (note this for later)
docker inspect k3d-my-lab-2-server-0 | grep -A 10 '"k3d-my-lab"'
# Look for "IPAddress": "172.18.0.X" - save this IP!
```

**📝 Note:** Save the IP address (e.g., `172.18.0.4`) - you'll need it when adding the cluster to ArgoCD.

---

## Part 3: Build and Push Application Images

### Step 5: Build Frontend Image

```bash
# Navigate to frontend code directory
cd web-app-code

# Build the frontend image
docker build -t <your-dockerhub-username>/my-app:v1.0.0 .

# Example: docker build -t skimpjr/my-app:v1.0.0 .

# Verify image was built
docker images | grep my-app
```

### Step 6: Build Backend Image

```bash
# Navigate to backend code directory
cd ../web-app-server

# Build the backend image
docker build -t <your-dockerhub-username>/my-server:v1.0.0 .

# Example: docker build -t skimpjr/my-server:v1.0.0 .

# Verify image was built
docker images | grep my-server
```

### Step 7: Push Images to Docker Hub

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

### Step 8: Update Helm Values with Your Images

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

### Step 9: Install ArgoCD on Control Plane

```bash
# Make sure you're on the control plane cluster
kubectl config use-context k3d-my-lab

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

### Step 10: Configure Permanent ArgoCD Access

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

### Step 11: Get ArgoCD Admin Password

```bash
# Retrieve the admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
```

**Save this password!** You'll need it to login.

### Step 12: Access ArgoCD UI

Open your browser and navigate to:

```
http://localhost:8080/argocd
```

**Login credentials:**

- Username: `admin`
- Password: (the password from Step 11)

**📝 Note:** If you see "Application not available", wait 1-2 minutes for the Ingress to be fully configured.

---

## Part 5: Deploy Applications to Control Plane

### Step 13: Create ArgoCD Application for Control Plane

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

- Navigate to http://localhost:8080/argocd
- You should see `my-web-app` application
- Status should be: **Synced** and **Healthy**

### Step 14: Verify Application Deployment

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

### Step 15: Create ServiceAccount on Remote Cluster

Switch to the remote cluster and create the necessary credentials:

```bash
# Switch to my-lab-2
kubectl config use-context k3d-my-lab-2

# Create namespace for ArgoCD service account
kubectl create namespace argocd-manager

# Create service account
kubectl create serviceaccount argocd-manager -n argocd-manager

# Grant cluster-admin permissions
kubectl create clusterrolebinding argocd-manager-role \
  --clusterrole=cluster-admin \
  --serviceaccount=argocd-manager:argocd-manager
```

### Step 16: Generate Long-lived Token

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

### Step 17: Get Remote Cluster IP Address

You noted this earlier in Step 4, but let's verify it:

```bash
# Get the internal IP address of my-lab-2
docker inspect k3d-my-lab-2-server-0 | grep -A 10 '"k3d-my-lab"'

# Look for "IPAddress": "172.18.0.X"
```

**Example output:**

```json
"IPAddress": "172.18.0.4"
```

### Step 18: Create Cluster Secret in ArgoCD

Switch back to the control plane and create the cluster secret:

```bash
# Switch back to control plane
kubectl config use-context k3d-my-lab
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
  server: https://172.18.0.4:6443 # Use the IP from Step 17
  config: |
    {
      "bearerToken": "<PASTE_YOUR_TOKEN_HERE>",
      "tlsClientConfig": {
        "insecure": true
      }
    }
```

**Replace `<PASTE_YOUR_TOKEN_HERE>` with the token from Step 16, then apply:**

```bash
kubectl apply -f add-cluster-token.yaml

# Wait 10 seconds for ArgoCD to discover the cluster
sleep 10

# List clusters
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster
```

### Step 19: Verify Cluster Connection

**In ArgoCD UI:**

1. Navigate to http://localhost:8080/argocd
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

### Step 20: Create Application for Remote Cluster

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
    server: https://172.18.0.4:6443 # Use the IP from Step 17
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

### Step 21: Verify Remote Deployment

```bash
# Check resources on my-lab-2 cluster
kubectl get pods --context k3d-my-lab-2
kubectl get svc --context k3d-my-lab-2
kubectl get ingress --context k3d-my-lab-2

# Test the application on my-lab-2
curl http://localhost:9080
```

You should see the application running on the remote cluster!

---

## Part 8: Update .gitignore and Commit

### Step 22: Protect Sensitive Files

**Update `web-app-infra/.gitignore`:**

```gitignore
# Cluster secrets (contain bearer tokens)
add-cluster.yaml
add-cluster-token.yaml
token-secret.yaml

# Any file with credentials
*secret*.yaml
*token*.yaml
```

### Step 23: Commit Configuration Files

```bash
cd web-app-infra

# Add only safe files
git add argocd-config.yaml
git add argocd-ingress.yaml
git add application.yaml
git add application-lab-2.yaml
git add .gitignore

# Commit
git commit -m "Add ArgoCD multi-cluster setup

- ArgoCD permanent access via Ingress
- Application for control plane (my-lab)
- Application for remote cluster (my-lab-2)
- Updated .gitignore to protect secrets"

# Push to GitHub
git push
```

---

## Part 9: Verification and Testing

### Step 24: Complete System Verification

**Check all clusters:**

```bash
# Control plane cluster
kubectl config use-context k3d-my-lab
kubectl get pods -A

# Remote cluster
kubectl config use-context k3d-my-lab-2
kubectl get pods -A
```

**Check ArgoCD Applications:**

1. Open http://localhost:8080/argocd
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
│                    Docker Network: k3d-my-lab                │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  my-lab (Control Plane)                              │    │
│  │  External: localhost:8080                            │    │
│  │  ┌──────────────────────────────────────┐            │    │
│  │  │  ArgoCD (namespace: argocd)           │            │    │
│  │  │  - Manages both clusters              │            │    │
│  │  │  - UI: http://localhost:8080/argocd   │            │    │
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
k3d cluster list

# Switch between clusters
kubectl config use-context k3d-my-lab
kubectl config use-context k3d-my-lab-2

# Delete a cluster
k3d cluster delete my-lab
```

### ArgoCD Management

```bash
# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

# List applications
kubectl get applications -n argocd

# Force sync an application
kubectl patch app my-web-app -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# List connected clusters
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster
```

### Debugging

```bash
# Check application status
kubectl describe application my-web-app -n argocd

# Check pods in all namespaces
kubectl get pods -A

# Check cluster IP (if connection fails)
docker inspect k3d-my-lab-2-server-0 | grep -A 10 '"k3d-my-lab"'

# Test connectivity from ArgoCD pod
kubectl exec -n argocd deployment/argocd-application-controller -- wget -O- https://172.18.0.4:6443/version --no-check-certificate
```

---

## Troubleshooting

### Problem: Cluster Connection Failed

**Symptoms:** ArgoCD shows "connection refused" or "no route to host"

**Solution:**

```bash
# 1. Verify cluster is running
k3d cluster list

# 2. Check Docker network connection
docker network inspect k3d-my-lab | grep my-lab-2-server-0

# 3. If not connected, reconnect
docker network connect k3d-my-lab k3d-my-lab-2-server-0

# 4. Get current IP address
docker inspect k3d-my-lab-2-server-0 | grep -A 10 '"k3d-my-lab"'

# 5. Update cluster secret with new IP
kubectl patch secret my-lab-2-cluster -n argocd -p '{"stringData":{"server":"https://NEW_IP:6443"}}'

# 6. Update application-lab-2.yaml with new IP and reapply
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

**Symptoms:** Browser shows "Application not available"

**Solution:**

```bash
# Check ArgoCD pods are running
kubectl get pods -n argocd

# Check ingress is created
kubectl get ingress -n argocd

# Verify port 8080 is mapped
k3d cluster list

# Check Traefik is running
kubectl get pods -n kube-system | grep traefik
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

If you want to start over from scratch while keeping all tools installed (Colima, k3d, kubectl, Docker, etc.):

### Quick Reset (Recommended)

```bash
# 1. Delete both k3d clusters
k3d cluster delete my-lab
k3d cluster delete my-lab-2

# 2. Verify clusters are gone
k3d cluster list
# Should show: "No clusters found"

# 3. Verify kubectl contexts are cleaned up
kubectl config get-contexts
# my-lab and my-lab-2 contexts should be removed

# Done! Now you can restart from Step 2 (Create Kubernetes Clusters)
```

### Deep Clean (If Quick Reset Doesn't Work)

Sometimes Docker containers or networks persist. Use this for a complete reset:

```bash
# 1. Stop all k3d containers
docker ps -a --filter "name=k3d" --format "{{.Names}}" | xargs -r docker stop

# 2. Remove all k3d containers
docker ps -a --filter "name=k3d" --format "{{.Names}}" | xargs -r docker rm -f

# 3. Remove k3d networks
docker network ls --filter "name=k3d" --format "{{.Name}}" | xargs -r docker network rm

# 4. Verify cleanup
docker ps -a | grep k3d  # Should return nothing
docker network ls | grep k3d  # Should return nothing

# 5. Clean kubectl config (remove stale contexts)
kubectl config delete-context k3d-my-lab 2>/dev/null || true
kubectl config delete-context k3d-my-lab-2 2>/dev/null || true

# 6. Verify kubectl config is clean
kubectl config get-contexts

# Done! Now restart from Step 2 (Create Kubernetes Clusters)
```

### Reset with Colima Restart

If you're experiencing Docker or network issues, restart Colima:

```bash
# 1. Delete clusters first
k3d cluster delete my-lab
k3d cluster delete my-lab-2

# 2. Stop Colima
colima stop

# 3. Start Colima fresh
colima start --runtime docker --cpu 4 --memory 6

# 4. Verify Docker is working
docker ps

# Done! Now restart from Step 2 (Create Kubernetes Clusters)
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

# Now restart from Step 5 (Build and Push Application Images)
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
- ✅ k3d installation
- ✅ kubectl installation
- ✅ Docker CLI
- ✅ Git repository and commits
- ✅ Docker Hub images (unless you explicitly remove them)
- ✅ All local code files

### What Gets Removed

After a fresh start reset:

- ❌ k3d clusters (my-lab, my-lab-2)
- ❌ ArgoCD installation
- ❌ Deployed applications
- ❌ Kubernetes contexts
- ❌ Docker networks created by k3d
- ❌ Cluster secrets and tokens

### Verification After Reset

Before starting over, verify everything is clean:

```bash
# Should return no clusters
k3d cluster list

# Should only show Docker default networks (bridge, host, none)
docker network ls

# Should show no k3d containers
docker ps -a | grep k3d

# Colima should still be running
colima status

# Docker should be accessible
docker ps
```

If all checks pass, you're ready to start fresh from **Step 2: Create Control Plane Cluster**! 🎉

---

## Complete Teardown (Uninstall Everything)

**⚠️ Warning:** This removes clusters AND stops all Docker/Colima services. Use "Fresh Start" above if you just want to reset.

### Remove Clusters and Stop Docker

```bash
# Delete both clusters
k3d cluster delete my-lab
k3d cluster delete my-lab-2

# Stop Colima (stops Docker runtime)
colima stop
```

### Complete Removal (including Colima VM)

If you want to completely remove Colima and start from scratch:

```bash
# Delete both clusters first
k3d cluster delete my-lab
k3d cluster delete my-lab-2

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
# Uninstall k3d (macOS with Homebrew)
brew uninstall k3d

# Uninstall Colima
brew uninstall colima

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
✅ Created two k3d Kubernetes clusters  
✅ Built and pushed Docker images to Docker Hub  
✅ Installed ArgoCD with permanent UI access  
✅ Deployed applications to control plane cluster  
✅ Connected remote cluster to ArgoCD  
✅ Deployed applications to remote cluster  
✅ Configured GitOps workflow with automated sync

**Key GitOps Benefits:**

- 📦 Single source of truth (Git repository)
- 🔄 Automated deployments and self-healing
- 🔍 Full audit trail of all changes
- 🎯 Declarative infrastructure configuration
- 🌍 Multi-cluster management from single control plane

**Happy GitOps! 🚀**
