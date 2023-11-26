# kubernetes basic setup
There are some commands which are to be executed in all 3 nodes, but others are to be executed only in master node or control plane of the k8s.

Create 3 EC2 instances with below things in mind - 
1. Create a security group with type: ALL TCP, source:Anywhere
2. one Master node -> t2.medium  (ubuntu 20.04)
   Two Worker node -> t2.micro  (ubuntu 20.04)
3. K8s required atlest 2gb of ram to execute the control plane ( or master node)
4. For worker nodes it is fine to use the t2.micro free tier instance.


## Commands to be executed in all 3 machines (1 master node and 2 worker nodes)
Upgrade apt packages
```bash
sudo apt-get update
```

Create configuration file for containerd:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf overlay br_netfilter 
EOF
```

Load modules:
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Set system configurations for Kubernetes networking: (niche ka pura woh command hai)
```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

Apply new settings:
```bash
sudo sysctl --system
```

Install containerd:
```bash
sudo apt-get update && sudo apt-get install -y containerd
```

Create default configuration file for containerd
```bash
sudo mkdir -p /etc/containerd
```

Generate default containerd configuration and save to the newly created default file:
```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Restart containerd to ensure new configuration file usage:
```bash
sudo systemctl restart containerd
```

Verify that containerd is running.
```bash
sudo systemctl status containerd
```

Disable swap:
```bash
sudo swapoff -a
```

Disable swap on startup in /etc/fstab:
```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Install dependency packages:
```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```

Download and add GPG key:
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

Add Kubernetes to repository list:
```bash
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```


Update package listings:
```bash
sudo apt-get update
```

Install Kubernetes packages (Note: If you get a dpkg lock message, just wait a minute or two before trying the command again):
```bash
sudo apt-get install -y  kubelet kubeadm kubectl kubernetes-cni nfs-common
```

Turn off automatic updates:
```bash
sudo apt-mark hold kubelet kubeadm kubectl kubernetes-cni nfs-common
```


## Below commands are to be executed only in Master Node or Control plane.
take a medium instance for master node

**Initialize the Cluster**
Initialize the Kubernetes cluster on the control plane node using kubeadm 
**Note: This is only performed on the Control Plane Node**
```bash
sudo kubeadm init
```

-----------------------------------------------no need--------------------------------------------------------|
Note: if we will get an error as "[ERROR NumCPU]: the number of available CPUs 1 is less than the required 2" |
Kubeadm runs a series of pre-flight checks to validate the system state before making changes.                |
This error means the host don't have minimum requirement of 2 CPU.                                            |
You can ignore the error if you still want to go ahead and install kubernetes on this host.                   | 
$ sudo kubeadm init --ignore-preflight-errors=NumCPU                                                          | 
--------------------------------------------------------------------------------------------------------------|

Set kubectl access:
```bash
mkdir -p $HOME/.kube
```

```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Test access to cluster:
```bash
kubectl get nodes
```

Install the Calico Network Add-On -
On the Control Plane Node, install Calico Networking:
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

```bash
kubectl get nodes
```

Join the Worker Nodes to the Cluster
In the Control Plane Node, create the token and copy the kubeadm join command 
**NOTE:The join command can also be found in the output from kubeadm init command**
```bash
kubeadm token create --print-join-command
```
Note : In both Worker Nodes, **paste the kubeadm join command** to join the cluster. Use sudo to run it as root:
$ sudo kubeadm join ....................<rest-of-the-command>.......................

In the Control Plane Node, view cluster status (Note: You may have to wait a few moments to allow all nodes to become ready)

The port number which shows there should be enabled, **6443** for example. **Enable that in security group in EC2 instance.**

Validate the setup by executing below commnad in master node
```bash
kubectl get nodes
```
