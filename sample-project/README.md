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
  - 192.168.1.240-192.168.1.250

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```

Apply it: `kubectl apply -f metallb-config.yml`

### 3.3. Install the nginx-ingress controller and apply ingress resource.

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

### 3.4. Implementing TLS for Secure HTTPS Access

For real-world scenario, let's replace self-signed certificate with below options:

### Option 1: Permanent Solution for Kubernetes Nodes
Install the CA certificate on all cluster nodes:

```bash
#On each node (masternode, workernode1, workernode2)
sudo scp bsuman@masternode:~/kube-yaml/certs/ca.crt /usr/local/share/ca-certificates/app-ca.crt
sudo update-ca-certificates
```

### Option 2: Using Self-Signed Certificate Setup for Private/Local Environment
As we are deploying this in private environment lets use self-signed certificates.

1. Installing cert-manager from link [here](https://cert-manager.io/docs/installation/))

   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.0/cert-manager.yaml
   ```
2. Creating Self-Signed Issuer
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
   
3. Creating Certificate Resources
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
   
4. Let's update created Ingress to use cert-manager
   `kubectl edit ingress app-ingress`
   and add these annotations:
   ```yaml
   tls:
   - hosts:
     - app.example.com
     secretName: app-tls-secret  # matching secretName in Certificate
   ```
   
5. Now, verify:

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

6. Full Trust Setup
   - Extract CA Certificate: 
     ```bash
     kubectl get secret app-tls-secret -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
     ```

   - Install CA on Local Machine: 
     ```bash
     sudo cp ca.crt /usr/local/share/ca-certificates/app-ca.crt
     sudo update-ca-certificates
     ```
     
7. Now let's verify TLS
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
     
### Finally 

Verify service endpoints
`kubectl get endpoints`

Verify MetalLB assignment. Look for Type: LoadBalancer and External-IP to be your network IP.
```bash
kubectl get svc -n ingress-nginx
```

Check Ingress Resource Status
```bash
kubectl get ingress
```

Verify Network Path
Trace request through services:
```kubectl describe ingress app-ingress```


Test Direct LoadBalancer Access
Bypass DNS to verify direct LoadBalancer IP access:
```bash
curl -H "Host: app.example.com" http://192.168.1.240
curl -H "Host: app.example.com" http://192.168.1.240/students
```
Should return same responses as domain access


Update DNS or /etc/hosts to point app.example.com to the external IP of the ingress-nginx service.
```bash
echo '192.168.1.240 app.example.com' | sudo tee -a /etc/hosts
```

Verify DNS resolution:
```bash
dig app.example.com +short
# Should return: 192.168.1.240
```

Check Ingress Controller Logs
```bash
kubectl logs -n ingress-nginx \
  -l app.kubernetes.io/component=controller \
  --tail=50
```

Verify browsing application
```bash
curl -I http://app.example.com
curl -v http://app.example.com
curl http://app.example.com/students
```

Connect to MongoDB using the legacy mongo shell:
```bash
kubectl exec -it web-mongodb-76c57d566d-sh4lk -- mongo studentDB
```

You should see:
```
mongo
MongoDB shell version v5.0.x
connecting to: mongodb://127.0.0.1:27017/studentDB?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID(...) }
MongoDB server version: 5.0.x
```


### Verify Data Persistence by deleting mongodb pods and then check in newly created pod

5.5 Testing the Cluster


5.6 Scaling and Resource Limits


5.7 CI/CD Integration (Optional)


