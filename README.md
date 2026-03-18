# **Canary Rollout (v1 → v2)**

### **rollout-canary.yaml**

This starts with **v1**.  
When you apply v2, Argo Rollouts will serve canary traffic **on a dedicated canary service**.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: demo-rollout
spec:
  replicas: 4
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo
        image: hashicorp/http-echo:0.2.3
        args:
        - "-text=Hello from v1"
        - "-listen=:80"
        ports:
        - containerPort: 80

  strategy:
    canary:
      canaryService: demo-canary
      stableService: demo-stable
      steps:
      - setWeight: 20      # 20% traffic to v2
      - pause: {}          # wait for manual check
      - setWeight: 50      # 50% traffic to v2
      - pause: {}          # wait again
```

# **Services**

### **Stable Service (v1 traffic)**

`demo-stable.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-stable
spec:
  selector:
    app: demo
  ports:
  - port: 80
    targetPort: 80
```

### **Canary Service (direct access to v2)**

`demo-canary.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-canary
spec:
  selector:
    app: demo
  ports:
  - port: 80
    targetPort: 80
```


# Ingress for both Stable & Canary**

`demo-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  rules:
  - host: stable.demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-stable
            port:
              number: 80

  - host: canary.demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-canary
            port:
              number: 80
```


# Deploy v1**

```bash
kubectl apply -f rollout-canary.yaml
kubectl apply -f demo-stable.yaml
kubectl apply -f demo-canary.yaml
kubectl apply -f demo-ingress.yaml
```


# Add Hosts Entries**

```bash
echo "10.10.0.2 stable.demo.local" | sudo tee -a /etc/hosts
echo "10.10.0.2 canary.demo.local" | sudo tee -a /etc/hosts
```


# Test v1**

Stable (v1 only):

    curl http://stable.demo.local:30080
    Hello from v1

Canary (points to same pods until update):

    curl http://canary.demo.local:30080
    Hello from v1


# Update to v2 (Start Canary)**

Modify rollout:

```yaml
args:
- "-text=Hello from v2"
```

Apply:

```bash
kubectl apply -f rollout-canary.yaml
```

Watch rollout:

```bash
kubectl argo rollouts get rollout demo-rollout --watch
```


# Curl Stable vs Canary vs Weighted Traffic**

### **Stable = 100% v1**

    curl http://stable.demo.local:30080
    Hello from v1

***

### **Canary (direct) = 100% v2**

    curl http://canary.demo.local:30080
    Hello from v2


### **During rollout – weighted traffic test**

Argo Rollouts does split on *stable service*, so test this:

    curl http://stable.demo.local:30080

You will see mixture:

*   At 20% weight → mostly v1, sometimes v2
*   At 50% weight → 50/50 ratio



# **Promote v2**

```bash
kubectl argo rollouts promote demo-rollout
```

Now:

    curl http://stable.demo.local:30080
    Hello from v2

Canary also becomes v2:

    curl http://canary.demo.local:30080
    Hello from v2



# **Rollback to v1**

```bash
kubectl argo rollouts undo demo-rollout
```

    curl http://stable.demo.local:30080
    Hello from v1


