<div align="center">
<h1>Kubernetes Cluster Setup</h1>
</div>


### Contents  
A. [Cluster Setup](#a-cluster-setup)  
B. [Cluster Upgrade 1.28.x to 1.31.x](#b-cluster-upgrade-128x-to-131x)  
C. [Etcd Setup](#c-etcd-setup)  
D. [Setting Up Cluster to Trust Private Registry with TLS](#d-setting-up-cluster-to-trust-private-registry-with-tls)

---
## A. Cluster Setup

<img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/cluster.png" alt="Cluster Image" width="500" />

## Prerequisites
Setup with 1 master nodes and 2 or more worker nodes. Each should have at least of following requirements
- Ubuntu 20.04 or higher
- 2 CPUs or 2vCPU
- 2 GB RAM or more per node
- 20 GB free disk space on /var or more
- Stable internet connection

## Lab Setup
- Masternode: 192.168.0.11   masternode.k8s.com
- First Workernode: 192.168.0.12   workernode1.k8s.com
- Second Workernode: 192.168.0.13   workernode2.k8s.com

### 1. Set Hostname On Each Node
  Login to each nodes and set hostname.
  
  - On masternode: 
      ```bash
      sudo hostnamectl set-hostname masternode.k8s.com
      exec bash
      ```
  
  - On workernode1
      ```bash
      sudo hostnamectl set-hostname workernode1.k8s.com
      exec bash
      ```
  
  - On workernode1
      ```bash
      sudo hostnamectl set-hostname workernode1.k8s.com
      exec bash
      ```

### 2. Add the following lines in /etc/hosts file on each nodes

  ```bash
  # /etc/hosts

  192.168.0.11   masternode.k8s.com
  192.168.0.12   workernode1.k8s.com
  192.168.0.13   workernode2.k8s.com		
  ```

### 3. Disable Swap & Add Kernel Parameters on each nodes
	
  - Disable swap: 
    ```bash
    sudo swapoff -a;
    sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab
    ```

  - Check swap by executing command 'free -h' and output to be as below:
    ```bash
    @masternode:~$ free -h
                   total        used        free      shared  buff/cache   available
    Mem:           1.9Gi       371Mi       509Mi       1.0Mi       1.1Gi       1.4Gi
    Swap:             0B          0B          0B
    @masternode:~$ 
    ```
   
  - Load the following kernel modules on all the nodes.
    ```bash
    sudo tee /etc/modules-load.d/containerd.conf <<EOF
    overlay
    br_netfilter
    EOF
    ```
    ```bash	
    sudo modprobe overlay;
    sudo modprobe br_netfilter
    ```
  
  - Set the following Kernel parameters for Kubernetes, run beneath tee command.
    ```bash
    sudo tee /etc/sysctl.d/kubernetes.conf <<EOT
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOT
    ```
  
  - Reload the above changes, run
    ```bash
    sudo sysctl --system
    ```
    
### 4. Install Containerd Runtime.
	
  - First install its dependencies
    ```bash
  	sudo apt-get update;
  	sudo apt-get install -y ca-certificates curl gnupg lsb-release apt-transport-https
    ```

  - Add Docker's official GPG key:
    ```bash
  	sudo install -m 0755 -d /etc/apt/keyrings ;
  	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  	sudo chmod a+r /etc/apt/keyrings/docker.gpg
    ```

  - Add the repository to Apt resources:
    ```bash
  	echo \
    	"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  	$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

  - Install containerd
  	```bash
  	sudo apt-get update;
  	sudo apt install -y containerd.io
  	```

  - Configure containerd so that it starts using systemd as cgroup.
  	```bash
  	containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1 ;
  	sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
  	```

  - Restart and enable containerd service
  	```bash
  	sudo systemctl enable containerd ;
  	sudo systemctl restart containerd
  	```

### 5.	Add Apt Repository For Kubernetes
  ```bash
	curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
	echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
  ```

### 6.	Install Kubectl, Kubeadm And Kubelet
  ```bash
	sudo apt update
	sudo apt install -y kubelet kubeadm kubectl
	sudo apt-mark hold kubelet kubeadm kubectl
  ```

### 7.	Install Kubernetes Cluster to setup master node

  - On the control plane node only, initialize the kubeadm and set up kubectl access
    ```bash
    sudo kubeadm init --control-plane-endpoint=masternode.k8s.com
    ```
	<img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/kubeadmInit.png" alt="kubeadm" width="600" />
 
  - Setting Up kubectl to interact with cluster
    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

### 8. 	Join Worker Nodes To The Cluster
	
  - On each worker node, use the kubeadm join command you noted down earlier after initializing the master node on step 6. It should look something like this:
    ```bash
  	kubeadm join masternode.k8s.com:6443 --token jyzam4.hyu4fag0bk3t6dqk \
  	--discovery-token-ca-cert-hash sha256:6f03fe7ad1875080d3b2b9592670175466254ba6bcedf87ba0d16fa6b7a0f23a 
    ```
    <img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/workerJoint.png" alt="Join" width="900" />
    

  - Alternatively generate token in masternode, and then execute the generated token in each worker node.
    ```bash
  	sudo kubeadm token create --print-join-command
    ```
	
### 9.	Install Network Plugin

   Run following kubectl command to install Calico network plugin from the master node,
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml	
   ```
   <img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/calicoCNI.png" alt="Calico" width="900" />

### 10.	Name and check the nodes.
   ```bash
	 #kubectl label node workernode1.k8s.com node-role.kubernetes.io/<role>=

	 kubectl label node workernode1.k8s.com node-role.kubernetes.io/worker=
	 kubectl label node workernode2.k8s.com node-role.kubernetes.io/worker=
   ```
   <img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/nodesDetail.png" alt="nodeDetail" width="600" />

   <img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/getPods-A.png" alt="getPods" width="600" />

---
## B. Cluster Upgrade 1.28.x to 1.31.x

First let's note that we have version 1.28.15 as shown below:

<img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/cluster.png" alt="Cluster Image" width="500" />

### Before Moving to Upgrade [below](#procedures)
let's note some important points to remember from the mistakes that attempted during the process.

### Attempt 1
- Right after canceling holds on kubeadm, kubectl, and kubelet components, the direct move to install 'v1.31.2' resulted in an error seen in the screenshot.
  This indicates a need to set up a package repository for v1.31.x to install components.
  <img width="500" alt="pckM1" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/pckM1.png">

- So, moved to setup packages repository for v1.31.x and continued to install kubeadm and upgrade it. Here, got to know that, upgrading to version 1.31.x only supports when current version is 1.30.x
  <img width="1000" alt="pckUpdate1-31" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/pckUpdate1-31.png">
  <img width="700" alt="adm131update" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/adm131update.png">
  <img width="1252" alt="needUpgrade130" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/needUpgrade130.png">

### Attempt 2
- Next, continued installing v1.30.0 components. Again here, got to know that it's necessary to setup package repository for v1.30.x before we install its components.
  <img width="1000" alt="pckUpdate1-30" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/pckUpdate1-30.png">

- But once installed components to higher version, we should perform 'allow downgrades' to install lower version.
  <img width="1011" alt="allow-downgrades" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/allow-downgrades.png">

- Then upgraded to v1.30.5
  <img width="926" alt="upgradeExec" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/upgradeExec.png">
  <img width="1165" alt="upgradeSuccess" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/upgradeSuccess.png">

### Attempt 3
- Finally, the package repository was updated to v1.31.x, components were installed and kubeadm version 1.31.2 was then applied.
  <img width="934" alt="upgrade131exec" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/upgrade131exec.png">
  <img width="1165" alt="upgrade131success" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/upgrade131success.png">

### Summary  
When upgrading Kubernetes, the general recommendation is to perform a stepwise upgrade, moving through each minor version sequentially. Here are the key things you should follow when upgrading from 1.29.8 to 1.31.2


> [!IMPORTANT]
> **Upgrade to the latest patch version in the 1.29 series:** This means you should update from 1.29.8 to the latest available patch version, which may be 1.29.x where x is the highest patch number (e.g., 1.29.10).  
>  
> **Upgrade to the latest patch of 1.30.x:** After successfully upgrading to the latest 1.29.x, the next step is to upgrade to the latest patch in the 1.30 series (e.g., 1.30.x).  
>  
> **Finally, upgrade to 1.31.2:** After completing the upgrade to the latest 1.30.x, you can then upgrade to 1.31.2.

> [!NOTE]
> Always upgrade to the latest patch version within a minor version before moving to the next minor version.
>
> You can skip intermediate patch versions within the same minor version (for example, if you are currently on 1.30.0, you can upgrade directly to 1.30.6 if that is the latest).
>
> The upgrade procedure on worker nodes should be executed one node at a time or few nodes at a time, without compromising the minimum required capacity for running your workloads.
>
> Ideally, upgrade both the control plane and worker nodes together when moving to a new major version.
> 
> It's essential to review the release notes for each version for any breaking changes or additional upgrade steps required.

> [!CAUTION]
> While the direct upgrade on worker nodes may complete successfully and pods may continue to run, there is a risk of instability or unexpected behavior in the application if any features or APIs have changed between versions. It is recommended to thoroughly test all functionality post-upgrade to ensure that everything operates as intended.
> 
> Still, skipping versions and direct upgrade on worker nodes is not recommended.
---
## Procedures

### 1. Cancel hold on kubeadm, kubectl and kubelet
   ```bash
   sudo apt-mark unhold kubeadm kubelet kubectl
   ```
	
### 2. Update Repository for v1.29.x and install components

### 2.1. Control Plane

- Setup the repository.

    ```bash
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```

    ```bash
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```

    ```bash
    sudo apt-get update
    ```

- Check if the version is available:
    ```bash
    sudo apt-cache madison kubeadm kubelet kubectl
    ```

- If the latest patch version is available, proceed to upgrade:
    ```bash
    sudo apt install -y kubeadm=1.29.10-1.1 kubectl=1.29.10-1.1 kubelet=1.29.10-1.1
    ```

- If you are upgrading the control plane (master) node, run:
    ```bash
    sudo kubeadm upgrade plan
    ```

- If the plan looks good, proceed with the upgrade:
    ```bash
    sudo kubeadm upgrade apply v1.29.10
    ```

- After upgrading kubeadm, restart kubelet:
    ```bash
    sudo systemctl restart kubelet
    ```

- Verify the versions:
    ```bash
    kubelet --version
    kubectl version
    kubeadm version
    ```


### 2.2. Worker Nodes

Let's note the running pods and nodes first.

<img width="500" alt="runningPods" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/runningPods.png">

- Execute below command in control plane.
  ```bash
  sudo kubeadm upgrade node
  ```

- Prepare the nodes for maintenance by marking it unschedulable and evicting the workloads:

  For workernode1:
  ```bash
  kubectl drain workernode1.k8s.com --ignore-daemonsets --delete-emptydir-data --force
  ```
  For workernode2:
  ```bash
    kubectl drain workernode2.k8s.com --ignore-daemonsets --delete-emptydir-data --force
  ```

  Use the --force option:
  	This will allow you to delete the pods even if they don't have a controller. However, be cautious when using this option, as it will forcefully terminate the pods, which may lead to data loss if they are running stateful applications.

  <img width="1397" alt="forceDrain" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/forceDrain.png">


  This is how pods and nodes run when drained successfully. All the running pods will be scheduled to another working node.

  <img width="800" alt="worker1drain" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/worker1drain.png">

  
- Cancel hold on kubeadm, kubectl and kubelet
   ```bash
   sudo apt-mark unhold kubeadm kubelet kubectl
   ```
   
- Setup the repository.

    ```bash
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```

    ```bash
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```

    ```bash
    sudo apt-get update
    ```

- Check if the version is available:
    ```bash
    sudo apt-cache madison kubeadm kubelet kubectl
    ```

- If the latest patch version is available, proceed to upgrade:
    ```bash
    sudo apt install -y kubeadm=1.29.10-1.1 kubectl=1.29.10-1.1 kubelet=1.29.10-1.1
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet
    ```
    
- (Optional) Bring the node back online by marking it schedulable:
    For workernode1
    ```bash
    kubectl uncordon workernode1.k8s.com
    ```
    For workernode2
    ```bash
    kubectl uncordon workernode2.k8s.com
    ```
    
### 3. Update Repository for v1.30.x and install components

### 3.1. Control Plane

- Setup the repository.

    ```bash
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```

    ```bash
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```

    ```bash
    sudo apt-get update
    ```

- Check if the version is available:
    ```bash
    sudo apt-cache madison kubeadm kubelet kubectl
    ```

- If the latest patch version is available, proceed to upgrade:
    ```bash
    sudo apt install -y kubeadm=1.30.6-1.1 kubectl=1.30.6-1.1 kubelet=1.30.6-1.1
    ```

- If you are upgrading the control plane (master) node, run:
    ```bash
    sudo kubeadm upgrade plan
    ```

- If the plan looks good, proceed with the upgrade:
    ```bash
    sudo kubeadm upgrade apply v1.30.6
    ```

- After upgrading kubeadm, restart kubelet:
    ```bash
    sudo systemctl restart kubelet
    ```

- Verify the versions:
    ```bash
    kubelet --version
    kubectl version
    kubeadm version
    ```

### 3.2. Worker Nodes

- Setup the repository.

    ```bash
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```

    ```bash
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```

    ```bash
    sudo apt-get update
    ```

- Check if the version is available:
    ```bash
    sudo apt-cache madison kubeadm kubelet kubectl
    ```

- If the latest patch version is available, proceed to upgrade:
    ```bash
    sudo apt install -y kubeadm=1.30.6-1.1 kubectl=1.30.6-1.1 kubelet=1.30.6-1.1
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet
    ```

- (Optional) Bring the node back online by marking it schedulable:
    For workernode1
    ```bash
    kubectl uncordon workernode1.k8s.com
    ```
    For workernode2
    ```bash
    kubectl uncordon workernode2.k8s.com
    ```

### 4. Update Repository for v1.31.x and install components

### 4.1. Control plane

 - Setup the repository.

    ```bash
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```

    ```bash
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```

    ```bash
    sudo apt-get update
    ```

- Check if the version is available:
    ```bash
    sudo apt-cache madison kubeadm kubelet kubectl
    ```

- If the latest patch version is available, proceed to upgrade:
    ```bash
    sudo apt install -y kubeadm=1.31.2-1.1 kubectl=1.31.2-1.1 kubelet=1.31.2-1.1
    ```

- If you are upgrading the control plane (master) node, run:
    ```bash
    sudo kubeadm upgrade plan
    ```

- If the plan looks good, proceed with the upgrade:
    ```bash
    sudo kubeadm upgrade apply v1.31.2
    ```

- After upgrading kubeadm, restart kubelet:
    ```bash
    sudo systemctl restart kubelet
    ```

- Verify the versions:
    ```bash
    kubelet --version
    kubectl version
    kubeadm version
    ```
    
### 4.2. Worker Nodes

- Setup the repository.

    ```bash
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```

    ```bash
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```

    ```bash
    sudo apt-get update
    ```

- Check if the version is available:
    ```bash
    sudo apt-cache madison kubeadm kubelet kubectl
    ```

- If the latest patch version is available, proceed to upgrade:
    ```bash
    sudo apt install -y kubeadm=1.31.2-1.1 kubectl=1.31.2-1.1 kubelet=1.31.2-1.1
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet
    ```

- (Necessary) As this is the final version we are updating, we need to execute below commands to bring the node back online by marking it schedulable:
    For workernode1
    ```bash
    kubectl uncordon workernode1.k8s.com
    ```
    For workernode2
    ```bash
    kubectl uncordon workernode2.k8s.com
    ```
### Finally
### Control Plane after upgrade:
   
   <img width="800" alt="finalUpdate" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/finalUpdate.png">

### Workernode1 after upgrade
   
   <img width="800" alt="podsafteruncordon1" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/podsafteruncordon1.png">

### Workernode2 after upgrade

   <img width="800" alt="podsafteruncordon2" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/podsafteruncordon2.png">

---

## C. Etcd Setup

Refer to the official [link](https://etcd.io/docs/v3.5/install/) for installation and new releases.

### 1. Install Prerequisites
   Ensure you have Go installed. etcd requires Go version 1.20 or higher, so update if necessary.

   ```bash
   wget https://go.dev/dl/go1.21.0.linux-amd64.tar.gz
   sudo rm -rf /usr/local/go
   sudo tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
   export PATH=/usr/local/go/bin:$PATH
   ```

   Confirm that the correct Go version is now in use.
   ```bash
   go version
   ```


### 2. Download the etcd Source Code 
   Clone the etcd repository from GitHub or download the source code.
   
   ```bash
   git clone https://github.com/etcd-io/etcd.git
   cd etcd
   ```

### 3. Build etcd and verify the version

   ```bash
   ./build.sh
   etcdctl version
   ```

### 4. (Optional) To run etcd, etcdctl, and etcdutl commands from anywhere, you can add the bin directory to your PATH:

   ```bash
   export PATH=$PATH:$(pwd)/bin
   ```

### 5. Testing

   ```bash
   etcdctl put key1 "Hello etcd"
   etcdctl get key1
   ```
---
## D. Setting Up Cluster to Trust Private Registry with TLS

### 1. Add Registry Certificate to All Nodes

On EVERY node (master + workers):
```bash
# 1. Copy the registry certificate to the node
#    (Replace with your actual certificate path)
sudo scp bsuman@192.168.1.110:/mnt/sdb2-partition/certs/domain.crt /usr/local/share/ca-certificates/192.168.1.110.crt

# 2. Update CA certificates
sudo update-ca-certificates

# 3. Restart container runtime and kubelet
sudo systemctl restart containerd  # or docker if using Docker
sudo systemctl restart kubelet
```

First check which runtime is cluster using
```bash
kubectl get node <node-name> -o json | jq '.status.nodeInfo.containerRuntimeVersion'
```
If you don't have jq, you can do:
```
kubectl get node <node-name> -o jsonpath='{.status.nodeInfo.containerRuntimeVersion}'
```
### Why don't we need to modify `containerd/config.toml`?

- By default, containerd uses the system's root CA set (located in `/etc/ssl/certs`). When we run `update-ca-certificates`, it updates the system's CA bundle, and containerd will trust any certificate signed by these CAs.

### What if we had a registry that uses a self-signed certificate and we don't want to add it to the system's trust store?

- Then we would have to configure containerd specifically for that registry by setting the `ca_file` in the `config.toml` as shown earlier. But in this case, we did the system-wide trust which is simpler and worked.

### Note

If Kubernetes nodes did not trust the self-signed certificate of the private Docker registry. The solution would be to add the registry's certificate to the system trust store on every node and restart containerd and kubelet.

### 2. (Optional) Configure Container Runtime to Trust Registry

Edit the container runtime config on each node:

For containerd (/etc/containerd/config.toml):

```toml
[plugins."io.containerd.grpc.v1.cri".registry.configs."192.168.1.110:5050".tls]
  insecure_skip_verify = false  # Keep false since we added the cert
```

For Docker

```json
{
  "insecure-registries": [],
  "registry-mirrors": [],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  }
}
```
Restart the runtime

```bash
sudo systemctl restart containerd && sudo systemctl restart kubelet
```

### 3. Verify Access on Nodes

Test image pulling manually on each node:
```bash
sudo ctr image pull 192.168.1.110:5050/frontend-crud-webapp:minimal
# Should show: "successfully pulled"
```

Since our cluster uses containerd runtime lets verify if image exists.
```bash
sudo ctr images ls
```

### 4. Create Image Pull Secret (Optional)

If the registry requires authentication:
```bash
kubectl create secret docker-registry regcred \
  --docker-server=192.168.1.110:5050 \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --namespace=default
```

### 5. Update Deployment YAML

Modify frontend-deploy.yml to use the secret:
```bash
spec:
  containers:
  - image: 192.168.1.110:5050/frontend-crud-webapp:minimal
    name: frontend-crud-webapp
  imagePullSecrets:  # Add this section
  - name: regcred
```

Finally redeploy application and verify.

### Key Notes:

### 1. Certificate Distribution
The registry's certificate (domain.crt) must be installed on all nodes in:
```
/usr/local/share/ca-certificates/
```
### 2. Firewall Rules
Ensure all nodes can access 192.168.1.110:5050:
```bash
sudo ufw allow from 192.168.1.0/24 to any port 5050
```

### 3. Registry Health Check
Verify registry accessibility:
```bash
curl -k https://192.168.1.110:5050/v2/_catalog
```

### 4. Image Naming Convention
Always use full registry path in deployments:
```yaml
image: 192.168.1.110:5050/image-name:tag
```

