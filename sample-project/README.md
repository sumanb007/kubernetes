

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
8. Actual Orchestration




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

- Verify MetalLB assignment. Look for Type: LoadBalancer and External-IP to be your network IP.
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

- You should see:
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
      volumes:
      - name: nginx-cache
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
              command: ["sh", "-c"]
              args: ["npm start -- --cache /tmp"]  # Disabling npm cache
              # OR use a volume:
              # volumeMounts:
              # - name: npm-cache
              #   mountPath: /home/node/.npm
            # volumes:
            # - name: npm-cache
            #   emptyDir: {}
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
