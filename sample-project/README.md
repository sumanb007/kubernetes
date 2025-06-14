# Orchestrating Application Container

Now let's continue further to orchestrate [Application](https://github.com/sumanb007/crud-webapplication/blob/main/README.md) containers using Kubernetes.

As planned in project, let's first:
1. Design Cluster. (as shown in the [link](https://github.com/sumanb007/kubernetes?tab=readme-ov-file#a-cluster-setup)
2. Ensure NFS is setup in every cluster host. (like [here](https://github.com/sumanb007/Labs/blob/main/NFS%20setup.md))
3. Cluster set up to Trust Private Registry with TLS. (like [here](https://github.com/sumanb007/kubernetes/blob/master/README.md#d-setting-up-cluster-to-trust-private-registry-with-tls))

### Table of Conents
1. [Creating Deployment and Service YAMLs](#2-creating-deployment-and-service-yamls)
2. [Setting Up Persistent Volumes with NFS](#3-setting-up-persistent-volumes-with-NFS)
3. [Ingress and TLS](#4-ingress-and-tls)
4. [Testing the Cluster](#5-testing-the-cluster)
5. [Scaling and Resource Limits](#6-scaling-and-resource)  

---

1. Creating Deployment and Service YAMLs
   
   
3. 

#Raw ideas

5.2 Creating Deployment & Service YAMLs

Deployments: replicas, rollingUpdate strategy, revisionHistoryLimit.
Services: ClusterIP vs NodePort vs LoadBalancer; targetPort vs port.
ConfigMaps & Secrets: envFrom/volumes patterns; secret mounts for TLS certs or DB creds.
Health probes: liveness vs readiness vs startup probes.
Labels & selectors: logical grouping, version labels for canary deploys.
Resource requests/limits (prep for 5.6).
Snippet examples for each.
5.3 Setting Up Persistent Volumes with NFS

Static vs dynamic provisioning; highlight StorageClass.
YAML for PersistentVolume (NFS server IP, path, reclaimPolicy).
PersistentVolumeClaim with ReadWriteMany.
Mount into pods: volumeMounts + subPath best practices.
Security: fsGroup, runAsUser, NFS‑related SELinux considerations.
5.4 Ingress and TLS

Ingress controller choice: NGINX, Traefik, or cloud‑native.
Ingress manifest: host‑based routing for /api vs /.
TLS termination:
Self‑signed for dev,
Let’s Encrypt via cert‑manager for prod.
Redirect HTTP→HTTPS.
Path rewrites or upstream timeouts.
(Optional) IngressClassParams in new NGINX‑v1.
Helm chart/values for easier reuse.
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
