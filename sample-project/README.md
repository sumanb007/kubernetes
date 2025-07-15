

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
3. [Ingress and TLS Setup](#3-ingress-and-tls-setup)  
    3.1. [MetalLB Setup](#31-metallb-setup)  
    3.2. [Nginx-ingress controller and Ingress resource](#32-nginx-ingress-controller-and-ingress-resource)  
    3.3. [TLS for Secure HTTPS Access](#33-tls-for-secure-https-access)
4. [Testing the Cluster](#4-testing-the-cluster)
5. [Scaling and Resource Limits](#5-scaling-and-resource-limits)  
    5.1. [Resources Limits](#51-resources-limits)  
    5.2. [Implementing Horizontal Pod Autoscaler (HPA)](#52-implementing-horizontal-pod-autoscaler-hpa)
6. [Pod Security and Network Policy](#6-pod-security-and-network-policy)  
    6.1. [Using securityContext](#61-using-securitycontext)  
    6.2. [Using Network Policy](#62-using-network-policy)
7. [Actual Orchestration](#7-actual-orchestration)




---
## 1. Setting Up Persistent Volumes with NFS

Though hardcoding NFS in pod is simple and easy to use, it has some key limitations:
   - No resusability
   - Dynamic provisioning not supported
   - No Centralized management
   - Bypasses Kubernetes’ storage abstraction model.
   - If the NFS server changes, all pod specs must be updated manually.

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
  replicas: 2
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
        name: frontend-crud-webapp
        ports:
        - containerPort: 8080
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
kubectl expose deployment frontend --port=80 --target-port=8080 --type=ClusterIP --dry-run=client -o yaml > frontend-service.yaml
```

Verify service
```bash
kubectl get svc frontend
kubectl describe svc frontend
```

### Backend yaml
Similarly for backend, lets continue from imperative command and modify.

```bash
kubectl create deployment backend --image=192.168.1.110:5050/backend-crud-webapp:minimal --dry-run=client -o yaml > backend-deployment.yaml
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
  replicas: 2
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
        ports:
        - containerPort: 4000
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

For mongodb, we will configure and create it to use nfs volume as PVC. 
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

## 3. Ingress and TLS Setup

We will expose both frontend and backend pods over a single domain (app.example.com) using Kubernetes Ingress and NGINX Ingress Controller, enabling clean URLs and HTTPS support.

### Steps in detail:

We will to deploy an Ingress controller (nginx-ingress) and then expose it via a LoadBalancer service so that MetalLB assigns an IP to it.

### 3.1. MetalLB Setup

We can use the manifest from the official project [https://metallb.io/installation/](https://metallb.io/installation/):
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

Wait for the pods to be ready.

### Configure the IP address pool for MetalLB
Kubernetes on-premises setups or on unsupported cloud providers can’t assign an external IP automatically for LoadBalancer services.
As we running Kubernetes on a bare-metal setup and it won’t automatically assign an external IP so we install MetalLB.

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
  - 192.168.1.240-192.168.1.250

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```

Apply it: `kubectl apply -f metallb-config.yml`

### 3.2. Nginx-ingress controller and Ingress resource.

We can use the manifest from the official project:
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
Let's create the basic ingress resource now, `ingress-resource.yml`:
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

### 3.3. TLS for Secure HTTPS Access

For real-world scenario, let's replace self-signed certificate with below options:

### Option 1: Permanent Solution for Kubernetes Nodes
Install the CA certificate on all cluster nodes:

```bash
#On each node (masternode, workernode1, workernode2)
sudo scp bsuman@masternode:~/kube-yaml/certs/ca.crt /usr/local/share/ca-certificates/app-ca.crt
sudo update-ca-certificates
```

### Option 2: Using Self-Signed Certificate Setup for Private/Local Environment
As we are deploying this in private environment this is recommended as best solution to use self-signed certificates.

Then later `curl` with '-k' option as encryption does not allow anyother host to access the application.
   Or we can copy the ca-certs to all hosts, that way is the trusted one and application can be accessed.

1. Installing cert-manager from link [here](https://cert-manager.io/docs/installation/))

   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.0/cert-manager.yaml
   ```

   Wait until cert-manager pods to become fully operational
   ```bash
   kubectl get pods -n cert-manager
   ```

   Verify webhook service
   ```bash
   kubectl get svc -n cert-manager
   kubectl describe svc cert-manager-webhook -n cert-manager
   ```
   
3. Creating Self-Signed Issuer
   ```bash
   cat <<EOF | kubectl apply -f -
   apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
     name: selfsigned-issuer
     namespace: default
   spec:
     selfSigned: {}
   EOF
   ```

   Verify it
   ```bash
   kubectl get issuer selfsigned-issuer -n default
   kubectl describe issuer selfsigned-issuer -n default
   ```


4. Creating Certificate Resources
   ```bash
   cat <<EOF | kubectl apply -f -
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: app-tls
     namespace: default
   spec:
     secretName: app-tls-secret
     duration: 8760h # 1 year
     renewBefore: 720h # 30 days
     issuerRef:
       name: selfsigned-issuer
       kind: Issuer
     commonName: app.example.com
     dnsNames:
     - app.example.com
     - "*.app.example.com"
     ipAddresses:
     - 192.168.1.240
     - 127.0.0.1
   EOF
   ```
   
5. Let's update created Ingress to use cert-manager
   `kubectl edit ingress app-ingress`
   and add these annotations:
   ```yaml
   tls:
   - hosts:
     - app.example.com
     secretName: app-tls-secret  # matching secretName in Certificate
   ```
   
6. Now, verify:

   - Check certificate status: 
     ```bash
     kubectl get certificate -w
     ```
   
   - Describe certificate (watch for events): 
     ```bash
     kubectl describe certificate app-tls
     ```
     
   - Check secret: 
     ```bash
     kubectl describe secret app-tls-secret
     ```
   
   - Test HTTPS access: 
     ```bash
     curl -k https://app.example.com  # Works immediately
     ```

7. Full Trust Setup
   - Extract CA Certificate: 
     ```bash
     kubectl get secret app-tls-secret -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
     ```

   - Install CA on Local Machine: 
     ```bash
     sudo cp ca.crt /usr/local/share/ca-certificates/app-ca.crt
     sudo update-ca-certificates
     ```
     
8. Now let's verify TLS
   - Check cert-manager logs: 
     ```bash
     kubectl logs -n cert-manager -l app.kubernetes.io/instance=cert-manager
     ```

   - Force certificate renewal: 
     ```bash
     kubectl annotate certificate app-tls cert-manager.io/force-renewal="true"
     ```

   - Verify certificate chain: 
     ```bash
     kubectl get secret app-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
     ```
---     
## 4. Testing the Cluster 

- Verify service endpoints
  `kubectl get endpoints`

- Verify MetalLB assignment. Look for Type: LoadBalancer and External-IP to be wer network IP.
  ```bash
  kubectl get svc -n ingress-nginx
  ```

- Check Ingress Resource Status
  ```bash
  kubectl get ingress
  ```

- Verify Network Path
  Trace request through services:
  ```kubectl describe ingress app-ingress```


- Test Direct LoadBalancer Access
  Bypass DNS to verify direct LoadBalancer IP access:
  ```bash
  curl -H "Host: app.example.com" http://192.168.1.240
  curl -H "Host: app.example.com" http://192.168.1.240/students
  ```
  Should return same responses as domain access


- Update DNS or /etc/hosts to point app.example.com to the external IP of the ingress-nginx service.
  ```bash
  echo '192.168.1.240 app.example.com' | sudo tee -a /etc/hosts
  ```

- Verify DNS resolution:
  ```bash
  dig app.example.com +short
  # Should return: 192.168.1.240
  ```

- Check Ingress Controller Logs
  ```bash
  kubectl logs -n ingress-nginx \
  -l app.kubernetes.io/component=controller \
  --tail=50
  ```

- Verify browsing application
  ```bash
  curl -I http://app.example.com
  curl -v http://app.example.com
  curl http://app.example.com/students
  ```

- Connect to MongoDB using the legacy mongo shell:
  ```bash
  kubectl exec -it web-mongodb-76c57d566d-sh4lk -- mongo studentDB
  ```

- we should see:
  ```
  mongo
  MongoDB shell version v5.0.x
  connecting to: mongodb://127.0.0.1:27017/studentDB?compressors=disabled&gssapiServiceName=mongodb
  Implicit session: session { "id" : UUID(...) }
  MongoDB server version: 5.0.x
  ```

  ### Testing Data Persistency in mongodb pods
  First lets simplify the process my using environment variable.
  ```bash
  MONGODB_POD=$(kubectl get pod -l app=web-mongodb -o jsonpath='{.items[0].metadata.name}')
  ```
  Now let's delete pod and verify the data persistence.
  ```bash
  curl https://app.example.com/students #should show data entry
  kubectl delete pod $MONGODB_POD
  MONGODB_POD=$(kubectl get pod -l app=web-mongodb -o jsonpath='{.items[0].metadata.name}')
  ```
  verify if Pod is UP and recheck data by curl
  ```bash
  kubectl get pod $MONGODB_POD
  curl https://app.example.com/students
  ```
---
## 5. Scaling and Resource Limits
To ensure reliable performance under varying load conditions, lets implement:
- High availability, if one pod fails, another continues serving traffic. `replicas:2`
- Manage resource requests and limits to prevent any one pod from consuming excessive resources and impacting neighbors.
- Ensure the service only receives traffic when the app is fully initialized and setup to restart the pod if it becomes unresponsive, improving resilience.

### 5.1. Resources Limits

Before proceeding, let's deploy metrics then we find out how much of the minimun resources is used when just a pod is up.

1. The most reliable and up-to-date method for component is:
  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  ```
  Let's patch with this below
   ```bash
   kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
   {
   "op": "add",
   "path": "/spec/template/spec/hostNetwork",
   "value": true
   },
   {
   "op": "replace",
   "path": "/spec/template/spec/containers/0/args",
   "value": [
   "--cert-dir=/tmp",
   "--secure-port=4443",
   "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname",
   "--kubelet-use-node-status-port",
   "--metric-resolution=15s",
   "--kubelet-insecure-tls"
   ]
   },
   {
   "op": "replace",
   "path": "/spec/template/spec/containers/0/ports/0/containerPort",
   "value": 4443
   }
   ]'
   ```
   If high-availability is needed in production, there's also a dedicated manifest:
   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability.yaml
   ```

2. Let's add resources limits to frontend, backend and mongo pods as below.
    ```yaml
    resources:
      requests
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
    ```
    
    This configuration ensures:
    - The pod always gets at least 100m CPU and 128Mi memory (requests).
    - It can burst up to 500m CPU and 256Mi memory if available (limits).
    - Helps Kubernetes schedule effectively and avoid pod eviction during high load.
   
   And for web-mongodb pod as below.   
    ```yaml
    resources:
      requests:
        cpu: "200m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"
    ```
    
3. Now apply, restart and verify the deployments
   ```bash
   kubectl apply -f mongodb-deploy.yml
   kubectl rollout restart deploy/web-mongodb
   kubectl get pod web-mongodb-76c57d566d-mtp6l -o yaml | grep -A6 "resources:"
   ```

### 5.2. Implementing Horizontal Pod Autoscaler (HPA)

- Let's use HPA for scaling efficiency. We will scale frontend and backned pods when the usage is 70% of the requested resources.
  
> [!IMPORTANT]
> - For mongo pod, being a stateful, it's generally not appropriate to scale MongoDB with HPA in this context. Because HPA is designed for stateless.
> - Instead we have used durable PVC (as already did via NFS). Added resource limits/requests to MongoDB pod to prevent it from exhausting node resources.

So let's continue with stateless frontend and backend:

```bash
kubectl autoscale deployment frontend cpu-percent=70 min=2 max=10
kubectl autoscale deployment backend --cpu-percent=70 --min=2 --max=10
```

### Or equivalent manifest for HPA we have :
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Let's verify now 

```bash
kubectl get hpa
```
---
## 6. Pod Security and Network Policy

Here, we will enforce below for Security:
- Enforce non-root execution using `runAsNonRoot`, `runAsUser`
- Prevent file system tampering using `readOnlyRootFilesystem`
- Remove Linux privileges using `drop capabilities`
- Limit syscalls using `seccompProfile`
- Limit pod communication using `Network Policy`

### 6.1. Using securityContext

We first structure into two levels:
- Pod-level: Identity and volume permissions (same across containers)
- Container-level: Process-level privileges and filesystem access

In both frontend and backend, Pod-level securityContext (applies to spec.securityContext):
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
```
For mongo pod, all files and directories in NFS server are owned by UID 999 and GID 999.
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 999
    runAsGroup: 999
    fsGroup: 999
    seccompProfile:
      type: RuntimeDefault
```

At Container-level in all frontend, backend and mongo pods, securityContext inside each container(spec.containers.securityContext):
```yaml
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
```

What these do ?
- runAsNonRoot: true → prevents running as root
- allowPrivilegeEscalation: false → blocks sudo-like privilege escalation
- capabilities.drop → removes Linux capabilities like NET_ADMIN, SYS_ADMIN, etc.
- seccompProfile → filters syscalls to reduce kernel attack surface

### Now lets apply changes verify securityContext
```bash
kubectl apply -f frontend-deploy.yml
kubectl apply -f backend-deploy.yml
kubectl apply -f mongodb-deploy.yml
```
# screenshots (logs of failed pods)
### Findings:

Being `readOnlyRootFilesystem: true`, the common theme is that all these containers are trying to write to their filesystems but are blocked.
- In frontend pod, the nginx container is failing because it tries to write to `/var/cache/nginx/client_temp` but the root filesystem is read-only.
- In backend pod, the node.js application is failing because npm tries to create a directory `/home/node/.npm` but the filesystem is read-only.
- MongoDB fails because it cannot unlink the socket file `/tmp/mongodb-27017.sock` due to read-only file system.

We have a few options:
 - Option 1: Disable `readOnlyRootFilesystem` for the containers that need to write to the root filesystem.
    - This is less secure but might be necessary for applications that are not designed to run with a read-only root filesystem.
     
 - Option 2: For each container, identify the directories that need to be writable and mount emptyDir volumes or write to the mounted volumes (like the one we have for MongoDB at `/data/db`).

Let's address each service:
 - Frontend (nginx): 
   The nginx image typically writes to `/var/cache/nginx` and `/var/run` (for pid file). We can try to make these directories writable by mounting emptyDir volumes or by disabling the read-only mode for the root filesystem.

   ```yaml
   # frontend-deploy.yml
   spec:
   template:
    spec:
      containers:
      - name: frontend-crud-webapp
        securityContext:
          readOnlyRootFilesystem: true  # Keeping security enabled
        volumeMounts:
        - name: nginx-cache
          mountPath: /var/cache/nginx
        - name: nginx-run
          mountPath: /var/run
      volumes:
      - name: nginx-cache
        emptyDir: {}
      - name: nginx-run
        emptyDir: {}
   ```
   
 - Backend (node.js):
   The npm cache and node_modules might require writing to the home directory. We can either disable the read-only root filesystem or change the npm cache directory to a location that is mounted as a volume (like an emptyDir).
   
   Here,
   Either:
      Disable the npm cache with npm start -- --cache /tmp
      OR
      Mount a writeable volume for npm cache:

   ```yaml
   # backend-deploy.yml
      spec:
        template:
          spec:
            containers:
   
            - name: backend-crud-webapp
              securityContext:
                readOnlyRootFilesystem: true
                volumeMounts:
                - name: npm-cache
                  mountPath: /home/node/.npm
              volumes:
              - name: npm-cache
                emptyDir: {}
              # OR disable npm cache:
              # command: ["sh", "-c"]
              # args: ["npm start -- --cache /tmp"]  # Disabling npm cache
              
   ```
   
 - MongoDB:
   We already have a volume at `/data/db`, but MongoDB also uses `/tmp`. We can try to mount an emptyDir volume at `/tmp` to allow writing.

   ```yaml
   # mongodb-deploy.yml
      spec:
        template:
          spec:
            containers:
            - name: web-mongodb
              volumeMounts:
              - name: mongo-storage
                mountPath: /data/db
              - name: tmp-volume
                mountPath: /tmp  # Adding this mount
            volumes:
            - name: mongo-storage
              persistentVolumeClaim:
                claimName: mongo-pvc
            - name: tmp-volume
              emptyDir: {}  # Adding this volume
   ```

   

### 6.2. Using Network Policy

By default, if no policies are defined, all pods can communicate with each other. This is a security risk because if one pod is compromised, an attacker can easily access other pods.

Now, lets block all traffic by default (ingress & egress) and explicitly grant ingress → frontend and ingress → backend → MongoDB traffic.
- This matters because it limits blast radius if a pod is compromised, enforces zero-trust within the cluster.

- This default-deny policy blocks all incoming (ingress) and outgoing (egress) traffic for all pods in the namespace.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

- Let's allow all pods to egress to the DNS server (CoreDNS) in the `kube-system` namespace on UDP port 53 for DNS resolution.
  Without DNS, service discovery fails. This policy is essential for allowing pods to resolve service names (like `backend`, `web-mongodb`). It's a necessary exception to the default-deny.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

- Ensure that only the Ingress Controller can talk to the frontend pods, and only on the required port. It prevents direct access to the frontend from other pods or outside the cluster (unless through the Ingress).
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
```

- Next, Ingress Controller routes some paths (e.g., `/students`) to the backend service. Without this, the Ingress Controller would be blocked by the default-deny when trying to reach the backend.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: 4000  # Matches backend containerPort
```

- This policy enables the frontend to call the backend APIs. It is a typical microservice communication pattern. Without this, the frontend would be blocked by the default-deny when trying to reach the backend.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 4000
```

- Finally, we secures the database so that only the backend can access it. It prevents other pods (like the frontend or any others) from connecting to the database.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-mongo
spec:
  podSelector:
    matchLabels:
      app: web-mongodb
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 27017
```

## 5.3. Scaling Mongo As Statefulset 

**Our goal:** Scale MongoDB seamlessly without manually creating PersistentVolumes (PVs) for each replica, while ensuring data consistency and reliable storage.

The steps for MongoDB scaling:
 1. We define the StorageClass that points to our NFS provisioner.
 2. The StatefulSet's `volumeClaimTemplate` uses that StorageClass.
 3. When we scale up the StatefulSet, for each new replica:
    - The StatefulSet controller creates a new PVC (with a unique name) based on the template.
    - The provisioner (which is our NFS provisioner deployment) sees the new PVC and dynamically creates a PV that is bound to that PVC.
    - The PV is backed by a new directory on the NFS server (the provisioner creates this directory).
    - The pod for the new replica is started and mounts the PV (i.e., the NFS directory) to its data path.
 This allows each MongoDB replica to have its own persistent storage, and the process is fully automated.

### Core Understanding

We are using a Persistent Volume (PV) backed by NFS for MongoDB. Since NFS supports ReadWriteMany access mode, multiple pods can mount the same volume simultaneously. However, MongoDB is not designed to run multiple replicas of the same database instance (i.e., the same data directory) in a shared storage setup.

But, the requirement here fails if to use a single NFS volume for the MongoDB data and to scale up instances(pods). If we are using a single NFS volume and plan to mount it to multiple MongoDB pods, all pods would be writing to the same data files, which would lead to data corruption and conflicts because MongoDB doesn't support this.

### HOW ?

MongoDB expects:
- Each replica to have its own isolated data folder
- Full control of the files in that folder (locking, journaling, flushing, etc.)
- No shared access from other MongoDB instances

Even though it's technically possible to create one RWX NFS PV, bind it to multiple PVCs, and then mount them into multiple MongoDB pods...it will lead to data corruption.

Because of File-level interference:
- MongoDB isn't designed for multiple processes to simultaneously access the same underlying data files (even if mounted via different paths).
- Even though the PVCs are separate objects, they still point to the same directory (the same files) on the NFS server. So, we must have separate volumes for each MongoDB instance( or pods).
- RWX allows multiple mounts, but it doesn't prevent overlapping writes or corrupted journals. Databases like MongoDB need exclusive file access to operate reliably.


### SOLUTION:
So, how we maintain Data consistency while still scaling replicas ? What Maintains Data Consistency in MongoDB Replica Set?
- Answer: MongoDB itself — not the storage or Kubernetes — is responsible for maintaining data consistency.
- Yes. So, when we plan to scale mongo pods, only one pod (the Primary) in replicas handles all write operations (insert/update/delete). The other pods are Secondaries, and they replicate data from the Primary. MongoDB itself (via its Replica Set election mechanism) is responsible for choosing which pod is Primary.

We can run this command inside a pod to see the replica set status:
```bash
mongo --eval 'rs.status()'
```

**So our goal is to dynamically scale mongo pod. How we do that ?**
We will design our goal in Steps:
 1. First deploy an NFS provisioner as a Pod to enable dynamic provisioning for NFS.
    - This pod acts as a dynamic storage controller inside Kubernetes cluster.
 3. Create a StorageClass for dynamic provisioning.
 4. Convert the mongo Deployment to a StatefulSet and use volumeClaimTemplates.
 5. Update the MongoDB image to a stable version (avoiding release candidates) and configure the replica set.
 6. Use a headless service for StatefulSet.


### 5.3.1 Install NFS Provisioner

- First, we allows egress from all pods in the default namespace to the master node's IP (192.168.1.11) on port 6443. Why ? Because we have blocked the all default traffic in above network policies. 

  So let's configure and a policy to our `networkPolicy.yml`.
  The network policy `allow-apiserver-access` as
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-apiserver-access
    namespace: default
  spec:
    podSelector: {}
    policyTypes:
    - Egress
    egress:
    - to:
      - ipBlock:
          cidr: 192.168.1.11/32  # Master node IP
      ports:
      - protocol: TCP
        port: 6443
    - to:  # Keep DNS access
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
      ports:
      - protocol: UDP
        port: 53
  ```
  We didn't have a network policy allowing egress to the API server. That's the reason we added `allow-apiserver-access` to allow egress to the master node's IP on port 6443.

  Kubernetes NetworkPolicy best supports IP-based configuration to ensure consistent behavior across all components rather than hostname-based rules. Also, direct IP access reduces latency and potential DNS lookup failures.

  This bypasses all DNS resolution issues; allows traffic explicitly to the API server IP; and consistent Approach that matches our kube-proxy configuration
  
- Next, we need to create RBAC permissions for the NFS provisioner because by default it doesn't have the necessary permissions to manage PersistentVolumes and PersistentVolumeClaims.
  So, before running the NFS provisioner manifest, we should:
  1. Create the RBAC resources (ServiceAccount, ClusterRole, ClusterRoleBinding).
  2. Modify the deployment to use that ServiceAccount.

  Why This is Necessary ?

    - Kubernetes has a zero-trust security model
    - Pods get minimal permissions by default
    - Storage provisioners need elevated privileges to manage cluster resources
    - RBAC ensures least-privilege access

  To avoid leader election errors about endpoints, then we need to add the endpoints resource to the RBAC role.
  We might see errors in the logs about not being able to get endpoints so we should update the ClusterRole to include endpoints too.

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: nfs-provisioner
      namespace: default
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: nfs-provisioner-role
    rules:
    - apiGroups: [""]
      resources: ["endpoints", "persistentvolumes", "persistentvolumeclaims", "events"]
      verbs: ["*"]
    - apiGroups: ["storage.k8s.io"]
      resources: ["storageclasses"]
      verbs: ["*"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: nfs-provisioner-binding
    subjects:
    - kind: ServiceAccount
      name: nfs-provisioner
      namespace: default
    roleRef:
      kind: ClusterRole
      name: nfs-provisioner-role
      apiGroup: rbac.authorization.k8s.io
  ```

- Now we write manifest to install nfs-provisioner in kubernetes network. This NFS provisioner acts as a PersistentVolume provisioner in Kubernetes. It automatically creates PVs backed by NFS storage when a PersistentVolumeClaim (PVC) is created.
   - Then we create a StorageClass that references the provisioner (`k8s-sigs.io/nfs-subdir-external-provisioner`). So, when a PVC requests storage from this StorageClass, the provisioner creates a new PV.
   - For each PVC, the provisioner creates a new directory on the NFS server at the path: `<NFS_PATH>/<namespace>-<pvcname>-<pvname>`.
   - The PV is then mounted to that directory.
   - When we scale a StatefulSet, each replica (pod) gets its own unique PVC.
     For example, if we have 3 replicas, we'll have 3 PVCs (and thus 3 PVs).
   - When we scale up (e.g., from 3 to 5 replicas), the StatefulSet controller creates new PVCs for the new replicas.
   - The NFS provisioner automatically provisions a new PV for each new PVC.
   - Each new MongoDB pod will then mount its own PV (NFS directory).
   - The PVCs are named as: `<volumeClaimTemplates-name>-<statefulset-name>-<index>`.

The StatefulSet ensures that each pod gets a stable network identity and stable storage (via the PVC template).
`nfs-provisioner.yml`
   ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nfs-provisioner
      namespace: default
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nfs-provisioner
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: nfs-provisioner
        spec:
          serviceAccountName: nfs-provisioner
          containers:
          - name: nfs-provisioner
            image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
            volumeMounts:
            - name: nfs-root
              mountPath: /persistentvolumes
            env:
            - name: KUBERNETES_SERVICE_HOST
              value: "192.168.1.11"
            - name: KUBERNETES_SERVICE_PORT
              value: "6443"
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.1.110
            - name: NFS_PATH
              value: /mnt/sdb2-partition/mongo-NFS-server
          volumes:
          - name: nfs-root
            nfs:
              server: 192.168.1.110
              path: /mnt/sdb2-partition/mongo-NFS-server
   ```
  
- Let's apply manifests now.
  ```bash
  kubectl apply -f networkPolicy.yml
  kubectl apply -f nfs-rbac.yml
  kubectl apply -f nfs-provisioner.yml
  ```
  
- Finally, let's verify API server connectivity from a test pod:
  ```bash
  kubectl run test --image=nicolaka/netshoot --rm -it --restart=Never -- curl -vk https://192.168.1.11:6443
  ```
  The 403 Forbidden response is expected and positive - which means:
  - Network connectivity to 192.168.1.11:6443 is working.
  - TLS handshake is successful.
  - API server is responding.


### 5.3.2 StorageClass and PVC Provisioning

**StorageClass**

This acts as the storage "blueprint" for dynamic provisioning. Abstracts the underlying NFS storage from applications.
- Without a StorageClass, we would have to manually create PVs for each PVC, which is not scalable.
- When the MongoDB StatefulSet creates a new pod (replica), it will create a PVC based on the template, and the NFS provisioner will automatically provision a PV and bind to that PVC.
- When a PVC is created and specifies the StorageClass (by name), the provisioner associated with that StorageClass will dynamically create a PV for that PVC.

`nfs-storageclass.yml`
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-sc
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner  # Critical link
parameters:
  archiveOnDelete: "false"
reclaimPolicy: Retain
volumeBindingMode: Immediate
```
This binds storage requests to our NFS provisioner. The provisioner field matches the PROVISIONER_NAME in the NFS deployment

**PVC Provisioning**

Next, when MongoDB requests storage via PVC, the StorageClass intercepts PVC creation, identifies the provisioner (k8s-sigs.io/nfs-subdir-external-provisioner) and triggers the provisioner to create a PersistentVolume. 

Let's write PVC:
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mongodb-data
spec:
  storageClassName: nfs-sc  # References our StorageClass
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Here's how to identify which PVC a PV is bound to:

```bash
kubectl get pv,pvc
kubectl describe pv <pv-name>
```
Key Field to Look For:
`Claim:`
This field shows the PVC that the PV is bound to in the format: namespace/pvc-name

### 5.3.3 Mongo StatefulSet

We are replacing the Deployment with a StatefulSet (sts). 

 Important changes for StatefulSet:
 1. We use `serviceName` under sts.spec. to specify the headless service that controls the network domain.
    `serviceName: web-mongodb-headless`
    
 2. Each pod in a StatefulSet gets a stable hostname based on the StatefulSet name and the ordinal index (e.g., web-mongodb-0).
 3. We use `volumeClaimTemplates` under sts.spec. for persistent storage. This will dynamically create a PVC for each pod.
      ```yaml
      volumeClaimTemplates:
      - metadata:
          name: mongo-storage
        spec:
          accessModes: [ "ReadWriteOnce" ]
          storageClassName: nfs-sc
          resources:
            requests:
              storage: 1Gi
      ```
 4. Remove the static PVC volumes
      ```yaml
      - name: mongo-storage
        persistentVolumeClaim:
          claimName: mongodb-data
      ```
      
 4. Configure MongoDB to start as a replica set by passing the `--replSet` flag under .containers.command
     ```yaml
      command:  
        - mongod
        - --bind_ip_all
        - --replSet
        - rs0
      ```
     
 5. And we add headless service in the manifest.
    ```yaml  
    apiVersion: v1
    kind: Service
    metadata:
      name: web-mongodb-headless
    spec:
      clusterIP: None  # headless required for sts
      selector:
        app: web-mongodb
      ports:
      - port: 27017
        targetPort: 27017
    ```
    **Why?** Pods need to talk to each other by name. Without a headless service, pods only get a shared name (like a common receptionist) and can’t call each other directly.

    MongoDB sts pods needs to:
    - Know who the other members are
    - Elect a primary (the leader pod)
    - Keep all data in sync between members

Now, We save the changes to `mongodb-sts.yml` and apply it

```bash
kubectl create -f mongodb-sts.yml
kubectl logs web-mongodb-4
kubectl exec -it web-mongodb-0 -- mongo --eval 'rs.status()'
```

Verifying these pods log we find that 'Replication has not yet been configured'
`"errmsg" : "no replset config has been received",`
`"codeName" : "NotYetInitialized"`

Even though we’ve deployed MongoDB with the --replSet rs0 flag, MongoDB does not automatically initialize a replica set. We have to do it manually from one pod, usually the first one (e.g. web-mongodb-0).

But, before to move further, let's recall that network policy we applied.
- Because the pods on different nodes, the traffic between nodes is being blocked at the node level (not by Kubernetes NetworkPolicy but by the underlying node network configuration).
- Issue is MongoDB pods on different nodes cannot communicate due to the default network policies blocking cross-node traffic.


MongoDB replica sets require bidirectional communication:
- Pod A → Pod B: Connection initiation
- Pod B → Pod A: Acknowledgment/response

So we try a workaround by creating a new NetworkPolicy that allow traffic based on the node IP ranges.
 The new NetworkPolicy `allow-mongo-cross-node`:
   - Allows ingress from any IP in the node CIDR (172.16.0.0/16) to MongoDB pods (on port 27017)
   - Allows egress to any IP in the node CIDR (172.16.0.0/16) on port 27017
        
   ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: allow-mongo-replica-communication
    spec:
      podSelector:
        matchLabels:
          app: web-mongodb
      policyTypes:
      - Ingress
      - Egress
      ingress:
      - from:
        - podSelector:
            matchLabels:
              app: web-mongodb
        ports:
        - protocol: TCP
          port: 27017
      egress:
      - to:
        - podSelector:
            matchLabels:
              app: web-mongodb
        ports:
        - protocol: TCP
          port: 27017
   ```

### Key Fixes in This Policy:
- **Added Egress Permission** allows MongoDB pods to initiate connections to other pods. Required for replica set formation.
- **Used IP Blocks Instead of Pod Selectors**
   - ipBlock: 172.16.0.0/16 covers all pods regardless of node
   - Pod selectors alone don't handle cross-node traffic well
- **Port-Specific Egress** Only allows egress on MongoDB port (27017). Maintains security while enabling required connectivity


**Now**, let's connect to web-mongodb-0: Run the replica set initiation command:
```sh
kubectl exec web-mongodb-0 -- mongo --eval "rs.initiate({
  _id: 'rs0',
  members: [
    {_id: 0, host: 'web-mongodb-0.web-mongodb-headless:27017'},
    {_id: 1, host: 'web-mongodb-1.web-mongodb-headless:27017'},
    {_id: 2, host: 'web-mongodb-2.web-mongodb-headless:27017'}
  ]
})"
```
The replica set initialization command should return `{ "ok" : 1 }`, indicating success.


Testing connectivity between pods using MongoDB's built-in client:
**From web-mongodb-0 to web-mongodb-1**
```bash
kubectl exec web-mongodb-0 -- mongo --host web-mongodb-1.web-mongodb-headless --eval "db.adminCommand('ping')"
```

Next let's verify if a pod in cluster is able to both access sts and able to resolve dns.
```bash
kubectl run debug-pod --rm -it --image nicolaka/netshoot --restart=Never -- bash -c "nc -zv web-mongodb 27017"
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup web-mongodb-1.web-mongodb-headless.default.svc.cluster.local
```

**Which Pod is Primary ?** check for "stateStr" : "PRIMARY"
```bash
kubectl exec web-mongodb-0 -- mongo --eval "rs.status()"
```

### So then, do we need to add members to mongo replica each time we scale up pod ?
Yes.  Note that MongoDB does not automatically add new members. We have to do it manually or with an automation tool. Here's the typical workflow:
 1. Initialize the replica set once (with the initial set of members).
 2. When scaling up, the StatefulSet creates new pods and new PVCs (if using volumeClaimTemplates) for the new members.
 3. Then, we must add the new members to the replica set configuration using `rs.add()`.

However, note that when we scale down a StatefulSet, it removes the pod but leaves the PVC (because of the retain policy). So if we scale up again and the same pod (with the same index) comes back, it will reuse the PVC and the data

### While we scale down ?

If we scale down, we should also remove the members from the replica set configuration to avoid having unreachable members.
However, scaling down a StatefulSet does not automatically remove the member from the replica set. we must do it manually.

```bash
rs.remove("web-mongodb-3.web-mongodb-headless:27017")  # Remove members
```

Scaling down the StatefulSet without removing the member from the replica set can cause issues because the replica set will keep trying to contact the removed pod.

Therefore, the best practice is:
   - To scale down: 
        1. Remove the member from the replica set (using `rs.remove()` on the primary).
        2. Then scale down the StatefulSet.
   - To scale up:
        1. Scale up the StatefulSet.
        2. Then add the new member to the replica set.


### Let's verify the application reachability now.

```bash
curl https://app.example.com/
```

Push data and verify
```bash
curl -X POST https://app.example.com/students/create-student -H "Content-Type: application/json" -d '{"name":"Suman Bhandari","email":"suman@example.com","rollNo":"111"}'
curl https://app.example.com/students
```

We have a StatefulSet for MongoDB with 3 replicas, each having its own PersistentVolumeClaim (PVC) for storage.
 To check the storage architecture, we need to:
 1. Verify the StatefulSet and its pods.
 2. Check the PVCs and PersistentVolumes (PVs) associated with each pod.
 3. Verify the storage class and its configuration.
 Let's break it down step by step.

Check Persistent Volume Claims (PVCs)
```bash
kubectl get pvc --selector app=web-mongodb
kubectl get pv
```

Verify StorageClass configuration
```bash
kubectl describe storageclass nfs-sc
```
Check Actual Storage on NFS Server
```bash
ls -l /mnt/sdb2-partition/mongo-NFS-server
```
Test data persistence by deleting a pod and checking data retention:
```bash

kubectl exec web-mongodb-0 -- mongosh --eval "
  use testdb;
  db.testcoll.insertOne({message: 'Storage Test'});"

kubectl delete pod web-mongodb-0

kubectl exec web-mongodb-0 -- mongosh --eval "
  use testdb;
  db.testcoll.find().pretty()"
```

Check Storage in Pods.
Inspect mounted volumes in a MongoDB pod:
```bash
kubectl exec web-mongodb-0 -- df -h /data/db
```
Verify Replica Set Storage Status. Check if all replicas are using their own storage:
```bash
kubectl exec web-mongodb-0 -- mongosh --quiet --eval "
  rs.status().members.forEach(m => {
    print(m.name + ': ' + m.stateStr + ' | Size: ' + m.optime.ts);
  })"
```

 


Required Files Summary
nfs-rbac.yml - RBAC permissions
nfs-provisioner.yml - Deployment with service account reference
networkPolicy.yml - Contains allow-apiserver-access policy
nfs-storageclass.yml - StorageClass definition
test-pvc.yml - Test PVC for verification
Key Lessons Learned

RBAC is Critical: Provisioners require explicit permissions to manage storage resources
Network Isolation Matters: Network policies can block API server access
Service Accounts: Must be explicitly referenced in deployments
Cluster Networking: Service network routes must be properly configured
Order of Operations:
RBAC before deployment
Network policies before pod creation
StorageClass before PVC creation


### Setup above Application in custom namespace
### Automating , Penetration Testing in Kubernetes ###
### Automating pod scale up if resources is high ###
