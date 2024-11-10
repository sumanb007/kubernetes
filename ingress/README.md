# Demonstration: Setting Up an Ingress in a Kubernetes Cluster (Bare Metal)

This guide demonstrates how to set up an Ingress in a Kubernetes cluster. The process covers deploying services, configuring an Ingress resource, and setting up an Ingress Controller using the NGINX Ingress Controller as an example.

## Prerequisites

- A running Kubernetes cluster.
- `kubectl` configured to interact with your cluster.
- Install the NGINX Ingress Controller to manage the Ingress resources.

## 1. Install NGINX Ingress Controller

First, let’s install the NGINX Ingress Controller in your cluster. You can use the following command to deploy the NGINX Ingress Controller with a simple setup.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
#kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0-beta.0/deploy/static/provider/baremetal/deploy.yaml
```

<img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/nginx-controller.png" alt="kubeadm" width="800" />

It may take a few moments for all the components (e.g., the ingress-nginx-controller pod) to start up. You can check the status of the pods with:

```bash
kubectl get pods -n ingress-nginx
```

## 2. Create Example Services

Let's deploy two simple services with deployments: app1 and app2. These services will run in the default namespace, and we’ll use Ingress to route traffic to them.


Deploy1: app1

```yaml
# app1-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: hashicorp/http-echo
        args:
        - "-text=Hello from app1"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app1
spec:
  selector:
    app: app1
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

Apply this configuration:
```bash
kubectl apply -f app1-deployment.yaml
```

Deploy2: app2

```yaml
# app2-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: hashicorp/http-echo
        args:
        - "-text=Hello from app2"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app2
spec:
  selector:
    app: app2
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

Apply this configuration

```bash
kubectl apply -f app2-deployment.yaml
```

## 3. Path based Ingress Resource

Now that app1 and app2 services are running, we’ll create an Ingress resource to expose them. In this example, we’ll route requests based on the path:

	- /app1 will route to app1 service.
	- /app2 will route to app2 service.

```yaml
#ingress-resource-path-based.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app1
                port:
                  number: 80
          - path: /app2
            pathType: Prefix
            backend:
              service:
                name: app2
                port:
                  number: 80
```

Run the ingress manifest
```bash
kubectl apply -f ingress.yaml
```

And verify
```bash
kubectl describe ingress example-ingress
```

## 4. Fixing service type.

Some Kubernetes setups on-premises or on unsupported cloud providers can’t assign an external IP automatically for LoadBalancer services.

If you're running Kubernetes on a bare-metal setup, it won’t automatically assign an external IP.

Install MetalLB for Bare-Metal Load Balancer Support.
If you need LoadBalancer support on a bare-metal cluster, you can use MetalLB, which provides a load-balancing IP range for on-premise setups.
```bawh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```

Verify all the resources:
```bash
kubectl get all -n metallb-system
```

After applying MetalLB, MetalLB will assign an external IP from the specified range to ingress-nginx-controller.

<img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/metallb.png" alt="kubeadm" width="900" />

## 5. Commands to Try

For  HTTP:

	curl http://<node-port-ip>:30686/app1
	curl http://<node-port-ip>:30686/app2
	These should work for HTTP without SSL errors.

For HTTPS:

	curl -k https://<node-port-ip>:30133/app1
	curl -k https://<node-port-ip>:30133/app2

<img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/path-based.png" alt="kubeadm" width="900" />

## 6. (Optional) Add Host-Based Routing

```yaml
#ingress-resource-host-based.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 80

```


Update /etc/hosts

	192.168.0.11 app1.example.com
	192.168.0.11 app2.example.com


And then apply config map.
This ConfigMap is configuring MetalLB to provide LoadBalancer functionality for your Kubernetes cluster on a local network.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.0.240-192.168.0.250  # Replace with an appropriate IP range for your network
```

Finally you can access:

- app1 at `curl http://app1.example.com`
- app2 at `curl http://app2.example.com`
