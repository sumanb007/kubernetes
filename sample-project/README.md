# Orchestrating Containerized Application

Now let's continue further to orchestrate [Application](https://github.com/sumanb007/crud-webapplication/blob/main/README.md) containers using Kubernetes.

## Project Walkthrough
As planned in project, let's first:
1. Design Cluster. (as done in this [link](https://github.com/sumanb007/kubernetes/blob/master/README.md#a-cluster-setup))
2. Ensure NFS is setup in every cluster host. (like [here](https://github.com/sumanb007/Labs/blob/main/NFS%20setup.md))
3. Setup Cluster to Trust Private Registry with TLS. (like [here](https://github.com/sumanb007/kubernetes/blob/master/README.md#d-setting-up-cluster-to-trust-private-registry-with-tls))

### Table of Conents
1. [Setting Up Persistent Volumes with NFS](#1-setting-up-persistent-volumes-with-nfs)
2. [Creating Deployment and Service YAMLs](#2-creating-deployment-and-service-yamls)
3. [Ingress and TLS](#3-ingress-and-tls)
4. [Testing the Cluster](#4-testing-the-cluster)
5. [Scaling and Resource Limits](#5-scaling-and-resource)  

---
## 1. Setting Up Persistent Volumes with NFS

Though hardcoding NFS in pod is simple and easy to use, it has some key limitations:
   - No resusability
   - Dynamic provisioning not supported
   - No Centralized management
   - Bypasses Kubernetes’ storage abstraction model.
   - If the nfs server changes, all pod specs must be updated manually.

### So, we are creating Persistent Volume. Let's continue..

### Persistent Volume yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongo-nfs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.1.110
    path: /mnt/sdb2-partition/mongo-NFS-server
  persistentVolumeReclaimPolicy: Retain
```

### Persistent Volume Claim yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

Let's apply and verify them now.
```bash
kubectl create -f mongo-nfs-pv.yml
kubectl create -f mongo-pvc.yml
kubectl get pv
kubectl get pvc
```
---
## 2. Creating Deployment and Service YAMLs

### Plan:
   - Both frontend and backend exposed over a single domain (app.example.com) using Kubernetes Ingress and NGINX Ingress Controller.
   - Mongodb exposed inside cluster.

### Frontend yaml

Let's generate yaml using imperative command and modify it.

```bash
kubectl create deployment frontend --image=192.168.1.110:5050/frontend-crud-webapp:minimal --dry-run=client -o yaml > frontend-deploy.yaml
```


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: frontend
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: frontend
    spec:
      containers:
      - image: 192.168.1.110:5050/frontend-crud-webapp:minimal
        name: frontend-crud-webapp-container
        resources: {}
status: {}
```
Apply the deployment and verify
```bash
kubectl apply -f frontend-deploy.yaml
kubectl get deployment frontend
kubectl describe deployment frontend
kubectl get pods -l app=frontend
```


Expose the pod using ClusterIP service.
```bash
kubectl expose deployment frontend--port=80 --target-port=3000 --type=ClusterIP --dry-run=client -o yaml > frontend-service.yaml
```

Verify service
```bash
kubectl get svc frontend
kubectl describe svc frontend
```

### Backend yaml
Similarly for backend, lets continue from imperative command and modify.

```bash
kubectl create deployment backend--image=192.168.1.110:5050/backend-crud-webapp:minimal --dry-run=client -o yaml > backend-deployment.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: backend
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: backend
    spec:
      containers:
      - image: 192.168.1.110:5050/backend-crud-webapp:minimal
        name: backend-crud-webapp
        resources: {}
status: {}
```

Apply the deployment and verify.
```bash
kubectl apply -f backend-deployment.yaml
kubectl get deployments
kubectl describe deployment backend
kubectl get pods -l app=backend
```

Expose the pod using ClusterIP service.
```bash
kubectl expose deployment backend --name=backend --port=5000 --target-port=4000  --type=ClusterIP  --dry-run=client -o yaml > backend-service.yaml
```


### Mongodb yaml

For mongodb, we will configure it to use nfs volume as PVC. 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-mongodb
  template:
    metadata:
      labels:
        app: web-mongodb
    spec:
      containers:
      - name: web-mongodb
        image: mongo:4.0-rc-xenial
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-storage
          mountPath: /data/db  # MongoDB stores its data here
      volumes:
      - name: mongo-storage
        persistentVolumeClaim:
          claimName: mongo-pvc
---
# MONGODB Service (remains the same)
apiVersion: v1
kind: Service
metadata:
  name: web-mongodb
spec:
  selector:
    app: web-mongodb
  ports:
  - port: 27017
    targetPort: 27017
  type: ClusterIP  # Accessible only within the cluster
```
---

## 3. Ingress and TLS

We will expose both frontend and backend pods over a single domain (app.example.com) using Kubernetes Ingress and NGINX Ingress Controller, enabling clean URLs and HTTPS support.

### Steps in detail:

We will to deploy an Ingress controller (nginx-ingress) and then expose it via a LoadBalancer service so that MetalLB assigns an IP to it.

### 3.1. Install MetalLB

We can use the manifest from the official project [https://metallb.io/installation/](https://metallb.io/installation/):
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

Wait for the pods to be ready.

### 3.2. Configure the IP address pool for MetalLB

We need to provide a range of IP addresses that are available in the local network and not in use by DHCP. That is, 192.168.1.200-192.168.1.220.

Let's create a file named `metallb-config.yml`
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.240-192.168.0.250

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```

Apply it: `kubectl apply -f metallb-config.yml`

### 3.3. Install the nginx-ingress controller and apply ingress resource.

We can use the manifest from the official project [https://metallb.io/installation/](https://metallb.io/installation/):
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

But note: this manifest creates a service of type LoadBalancer. After applying, the service for the nginx-ingress controller will get an external IP from MetalLB.

Alternatively,
we can use the manifest that is for bare metal and then change the service type to LoadBalancer? Actually, the manifest above already uses LoadBalancer.

However, let me note: the manifest above might not be the best for bare metal?
Actually, it is the same because we are using MetalLB to provide the load balancer.

After applying, check the service in namespace `ingress-nginx`:

```bash
kubectl get svc -n ingress-nginx
```
It should get an external IP.

### Ingress resource
Let's create the ingress resource now. 
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      - path: /students
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 5000
```

5.5 Testing the Cluster

Smoke tests: curl ingress URL, call API endpoints.
kubectl port‑forward for local debugging.
Log checks: kubectl logs -f per component.
kubectl get events / describe` for troubleshooting.
Readiness gates: make CI fail fast if pods not ready.
Chaos testing: delete pod, cordon node, simulate NFS outage.
5.6 Scaling and Resource Limits

CPU/memory requests & limits rationale.
Horizontal Pod Autoscaler (HPA) tied to CPU or custom metrics.
Vertical Pod Autoscaler (VPA) for long‑running services.
Cluster Autoscaler / node groups (if on cloud).
PodDisruptionBudgets so rolling upgrades don’t drop traffic.
Quota & LimitRange per namespace.
Quick Grafana screenshot for visual proof.
5.7 CI/CD Integration (Optional)

Build pipeline: GitHub Actions → Docker build → push to registry.
Lint & helm‑template stage to catch YAML errors.
Deploy pipeline:
kubectl apply,
Helm upgrade, or
Argo CD / Flux (GitOps).
Automatic HPA tests in pipeline (k6 or locust).
Promotion workflow: dev → staging → prod via tags.
Secret management: OIDC‑based push to kubeseal/sops.
