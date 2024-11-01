<div align="center">
<h1>Kubernetes Cluster Installation</h1>
</div>

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

## 1. Set Hostname On Each Node
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

## 2. Add the following lines in /etc/hosts file on each nodes

  ```bash
  # /etc/hosts

  192.168.0.11   masternode.k8s.com
  192.168.0.12   workernode1.k8s.com
  192.168.0.13   workernode2.k8s.com		
  ```

## 3. Disable Swap & Add Kernel Parameters on each nodes
	
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
    
## 4. Install Containerd Runtime.
	
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

## 5.	Add Apt Repository For Kubernetes
  ```bash
	curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
	echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
  ```

## 6.	Install Kubectl, Kubeadm And Kubelet
  ```bash
	sudo apt update
	sudo apt install -y kubelet kubeadm kubectl
	sudo apt-mark hold kubelet kubeadm kubectl
  ```

## 7.	Install Kubernetes Cluster to setup master node

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

## 8. 	Join Worker Nodes To The Cluster
	
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
	
## 9.	Install Network Plugin

   Run following kubectl command to install Calico network plugin from the master node,
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml	
   ```
   <img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/calicoCNI.png" alt="Calico" width="900" />

## 10.	Name and check the nodes.
   ```bash
	 #kubectl label node workernode1.k8s.com node-role.kubernetes.io/<role>=

	 kubectl label node workernode1.k8s.com node-role.kubernetes.io/worker=
	 kubectl label node workernode2.k8s.com node-role.kubernetes.io/worker=
   ```
   <img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/nodesDetail.png" alt="nodeDetail" width="600" />

   <img src="https://raw.githubusercontent.com/sumanb007/kubernetes/master/img/getPods-A.png" alt="getPods" width="600" />
   
