# Set up K8s Cluster with kubeadm

To set up a Kubernetes (k8s) cluster with `kubeadm`, follow these steps:

## Prerequisites

### Minimum hardware requirements
- Master Node: 2 CPUs and 2 GB RAM
- Worker Nodes: 1 CPU and 1 GB RAM

### Verify the MAC address and product_uuid are unique for every node

- Get the MAC address of the network interfaces using the command `ip link`
- The product_uuid can be checked by using the command `sudo cat /sys/class/dmi/id/product_uuid` (ex- a202d385-4817-508f-a713-e2074f422121)

### Check required ports 
These [required ports](https://kubernetes.io/docs/reference/networking/ports-and-protocols/) need to be open in order for Kubernetes components to communicate with each other. You can use tools like `netcat` to check if a port is open.

### Swap Config
The default behavior of a kubelet is to fail to start if swap memory is detected on a node. This means that swap should either be disabled or tolerated by kubelet. To disable swap, `sudo swapoff -a` can be used to disable swapping temporarily. To make this change persistent across reboots, make sure swap is disabled in config files like `/etc/fstab` using `sudo sed -i '/swap/d' /etc/fstab`

### Installing a container runtime
Kubernetes requires a container runtime, and Docker is a commonly used option.
```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo sh -c "containerd config default > /etc/containerd/config.toml"
sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

### Installing kubeadm, kubelet and kubectl (need to install in all machines)
- Update the `apt` package index and install packages needed to use the Kubernetes `apt` repository:
```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

- Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

- Add the appropriate Kubernetes apt repository. Please note that this repository have packages only for Kubernetes 1.31; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

- Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

- (Optional) Enable the kubelet service before running kubeadm:
```bash
sudo systemctl enable --now kubelet
```

### Set up Master Node
On the Master node, follow these steps to initialize the cluster and pull the necessary images.

- Pull Kubernetes Images `sudo kubeadm config images pull`
- Initialize the Cluster `sudo kubeadm init --pod-network-cidr=10.244.0.0/16`
- Check the Master Node's Readiness `sudo kubectl get --raw='/readyz?verbose'`

To start using your cluster, you need to run the following as a regular user:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Alternatively, if you are the root user, you can run:
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```
You should now deploy a pod network to the cluster.
Run `kubectl apply -f [podnetwork].yaml` with one of the options listed at:
```
https://kubernetes.io/docs/concepts/cluster-administration/addons/
```
Then you can join any number of worker nodes by running the following on each as root:
```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> \
	--discovery-token-ca-cert-hash sha256:<hash>
```

### Install Pod network addon
Use the following command to install Flannel:
```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
