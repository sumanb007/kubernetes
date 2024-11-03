<div align="center">
<h1>Kubernetes Cluster Installation and Upgrade</h1>
</div>


### Contents
A. [Cluster Setup](#a-cluster-setup)  
B. [Cluster Upgrade 1.29.x to 1.31.x](#b-cluster-upgrade-129x-to-131x)


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
## B. Cluster Upgrade

First let's note that we have version 1.29.8 as shown below:

<img width="515" alt="currentVersion" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/currentVersion.png">

### Before Moving to Upgrade [below](#procedures)
let's note some important points to remember from the mistakes that attempted during the process.

### Attempt 1
- Right after I canceled holds on kubeadm,kubectl and kubelet components, We directly moved to install 'v1.31.2' version.
  Then got this error seen in screenshot. That means we need to setup package repository for v1.31.x to install components.
  <img width="500" alt="pckM1" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/pckM1.png">

- So, moved to setup packages repository for v1.31.x and continued to install kubeadm and upgrade it. Here, got to know that, upgrading to version 1.31.x only supports when current version is 1.30.x
  <img width="1000" alt="pckUpdate1-31" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/pckUpdate1-31.png">
  <img width="700" alt="adm131update" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/adm131update.png">
  <img width="1252" alt="needUpgrade130" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/needUpgrade130.png">

### Attempt 2
- Next, continued installing v1.30.0 components. Again here, got to know that we need to setup package repository for v1.30.x before we install its components.
  <img width="1000" alt="pckUpdate1-30" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/pckUpdate1-30.png">

- But once you have installed components to higher version you have to 'allow downgrades' to install lower version.
  <img width="1011" alt="allow-downgrades" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/allow-downgrades.png">

- Then upgraded to v1.30.5
  <img width="926" alt="upgradeExec" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/upgradeExec.png">
  <img width="1165" alt="upgradeSuccess" src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/upgradeSuccess.png">

### Attempt 3
- Finally, updated the package repository to v1.31.x and installed components. Then applied kubeadm version 1.31.2.
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
> It's essential to review the release notes for each version for any breaking changes or additional upgrade steps required.

---
## Procedures

### 1. Cancel hold on kubeadm, kubectl and kubelet
   ```bash
   sudo apt-mark unhold kubeadm kubelet kubectl
   ```
	
### 2. Update Repository for v1.29.x and install components

   ```bash
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   ```
   ```bash
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```
   ```bash
   sudo apt-get update
   ```
   
   Check if the version is available:
   ```bash
   sudo apt-cache madison kubeadm kubelet kubectl
   ```
   
   If latest patch version is available, proceed to upgrade:
   ```bash
   sudo apt install -y kubeadm=1.29.10-1.1 kubectl=1.29.10-1.1 kubelet=1.29.10-1.1
   ```

   If you are upgrading the control plane (master) node, run:
   ```bash
   sudo kubeadm upgrade plan
   ```

   If the plan looks good, proceed with the upgrade
   ```bash
   sudo kubeadm upgrade apply v1.29.10
   ```

   After upgrading kubeadm, restart kubelet:
   ```bash
   sudo systemctl restart kubelet
   ```

   Verfify the versions
   ```bash
   kubelet --version
   kubectl version
   kubeadm version
   ```

### 3. Update Repository for v1.30.x and install components

   ```bash
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   ```
   ```bash
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```
   ```bash
   sudo apt-get update
   ```
   
   Check if the version is available:
   ```bash
   sudo apt-cache madison kubeadm kubelet kubectl
   ```
   
   If latest patch version is available, proceed to upgrade:
   ```bash
   sudo apt install -y kubeadm=1.30.6-1.1 kubectl=1.30.6-1.1 kubelet=1.30.6-1.1
   ```

   If you are upgrading the control plane (master) node, run:
   ```bash
   sudo kubeadm upgrade plan
   ```

   If the plan looks good, proceed with the upgrade
   ```bash
   sudo kubeadm upgrade apply v1.30.6
   ```

   After upgrading kubeadm, restart kubelet:
   ```bash
   sudo systemctl restart kubelet
   ```

   Verfify the versions
   ```bash
   kubelet --version
   kubectl version
   kubeadm version
   ```

### 4. Update Repository for v1.31.x and install components

   ```bash
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   ```
   ```bash
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```
   ```bash
   sudo apt-get update
   ```
   
   Check if the version is available:
   ```bash
   sudo apt-cache madison kubeadm kubelet kubectl
   ```
   
   If latest patch version is available, proceed to upgrade:
   ```bash
   sudo apt install -y kubeadm=1.31.2-1.1 kubectl=1.31.2-1.1 kubelet=1.31.2-1.1
   ```

   If you are upgrading the control plane (master) node, run:
   ```bash
   sudo kubeadm upgrade plan
   ```

   If the plan looks good, proceed with the upgrade
   ```bash
   sudo kubeadm upgrade apply v1.31.2
   ```

   After upgrading kubeadm, restart kubelet:
   ```bash
   sudo systemctl restart kubelet
   ```

   Verfify the versions
   ```bash
   kubelet --version
   kubectl version
   kubeadm version
   ```

   <img width="610" alt="finalUpdate" src="https://github.com/user-attachments/assets/65e0024d-a5a7-4200-bbf3-4fba2ac484d6">
