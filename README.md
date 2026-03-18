# Argo CD + Argo Rollouts Canary Deployment**

## **Lab Objectives**

By the end of this lab you will:

*   Install **Argo CD**
*   Install **Argo Rollouts**
*   Deploy a **sample app** using Rollouts
*   Configure **Canary strategy**
*   Use **Argo Rollouts Dashboard/Kubectl plugin** to promote/abort canary

***

# **Prerequisites**

*   Kubernetes cluster (Kind / Minikube recommended)
*   `kubectl` installed
*   `argocd` CLI (optional but useful)

***

#  **Create a Kubernetes Cluster (Kind Example)**

```bash
kind create cluster --name argolab
```

***

#  **Install Argo CD**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Wait for pods:

```bash
kubectl get pods -n argocd
```

***

# **Expose Argo CD (Port-Forward)**

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

### Get ArgoCD admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Login UI:  
 <https://localhost:8080>

***

#  **Install Argo Rollouts**

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

***

#  **Install Argo Rollouts Kubectl Plugin (Recommended)**

```bash
brew install argoproj/tap/kubectl-argo-rollouts
```

***

#  **Create a Sample Application (Rollout + Service)**

Create folder:

    rollouts-lab/
      rollout.yaml
      service.yaml
      app.yaml (for ArgoCD)

***

#  **rollout.yaml**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: canary-demo
  namespace: default
spec:
  replicas: 4
  strategy:
    canary:
      canaryService: canary-demo-canary
      stableService: canary-demo-stable
      steps:
        - setWeight: 20
        - pause: { duration: 30 }
        - setWeight: 40
        - pause: {}
  selector:
    matchLabels:
      app: canary-demo
  template:
    metadata:
      labels:
        app: canary-demo
    spec:
      containers:
        - name: demo
          image: nginx:1.20
          ports:
            - containerPort: 80
```

***

#  **service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: canary-demo-stable
spec:
  selector:
    app: canary-demo
  ports:
    - port: 80

---
apiVersion: v1
kind: Service
metadata:
  name: canary-demo-canary
spec:
  selector:
    app: canary-demo
  ports:
    - port: 80
```

***

#  **Create ArgoCD Application**

Create: **app.yaml**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: canary-rollout-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_GITHUB/rollouts-lab.git
    targetRevision: HEAD
    path: .
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

Apply it:

```bash
kubectl apply -f app.yaml
```

ArgoCD now auto-syncs your Rollout.

***

#  **Verify Rollout**

```bash
kubectl argo rollouts get rollout canary-demo
```

or use dashboard:

```bash
kubectl argo rollouts dashboard
```

Open:  
 <http://localhost:3100>

***

# **Trigger a Canary Deployment (Update Image)**

Edit image from:

    nginx:1.20

to:

    nginx:1.21

Commit → ArgoCD detects → Rollout starts canary.

***

# **Observe Canary Steps**

Check progress:

```bash
kubectl argo rollouts get rollout canary-demo --watch
```

You will see:

*   Set weight: 20%
*   Pause 30 sec
*   Set weight: 40%
*   Pause (await manual approval)

***

# **Promote Canary**

```bash
kubectl argo rollouts promote canary-demo
```

This moves from 40% → 100% → Stable.

***

# **Abort Canary **

```bash
kubectl argo rollouts abort canary-demo
```

Automatically rolls back to last stable version.



# Want me to package this lab into a **PDF**, **GitHub-ready repository**, or **step-by-step guide with diagrams**?
