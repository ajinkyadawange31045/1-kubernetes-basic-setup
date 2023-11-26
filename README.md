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
<br />
<br />
Create configuration file for containerd:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf overlay br_netfilter 
EOF
```
<br />
<br />
Load modules:
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```
<br />
<br />
Set system configurations for Kubernetes networking: (niche ka pura woh command hai)
```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
<br />
<br />
Apply new settings:
```bash
sudo sysctl --system
```
<br />
<br />
Install containerd:
```bash
sudo apt-get update && sudo apt-get install -y containerd
```
<br />
<br />
Create default configuration file for containerd
```bash
sudo mkdir -p /etc/containerd
```
<br />
<br />
Generate default containerd configuration and save to the newly created default file:
```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```
<br />
<br />
Restart containerd to ensure new configuration file usage:
```bash
sudo systemctl restart containerd
```
<br />
<br />
Verify that containerd is running.
```bash
sudo systemctl status containerd
```
<br />
<br />
Disable swap:
```bash
sudo swapoff -a
```
<br />
<br />
Disable swap on startup in /etc/fstab:
```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
<br />
<br />
Install dependency packages:
```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```
<br />
<br />
Download and add GPG key:
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
<br />
<br />
Add Kubernetes to repository list:
```bash
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
<br />
<br />

Update package listings:
```bash
sudo apt-get update
```
<br />
<br />
Install Kubernetes packages (Note: If you get a dpkg lock message, just wait a minute or two before trying the command again):
```bash
sudo apt-get install -y  kubelet kubeadm kubectl kubernetes-cni nfs-common
```
<br />
<br />
Turn off automatic updates:
```bash
sudo apt-mark hold kubelet kubeadm kubectl kubernetes-cni nfs-common
```
<br />
<br />

## Below commands are to be executed only in Master Node or Control plane.
take a medium instance for master node

**Initialize the Cluster**
Initialize the Kubernetes cluster on the control plane node using kubeadm 
**Note: This is only performed on the Control Plane Node**
```bash
sudo kubeadm init
```
<br />
<br />
-----------------------------------------------no need--------------------------------------------------------
Note: if we will get an error as "[ERROR NumCPU]: the number of available CPUs 1 is less than the required 2" 
Kubeadm runs a series of pre-flight checks to validate the system state before making changes.                
This error means the host don't have minimum requirement of 2 CPU.                                            
You can ignore the error if you still want to go ahead and install kubernetes on this host.                    
```sudo kubeadm init --ignore-preflight-errors=NumCPU```                                                       
--------------------------------------------------------------------------------------------------------------
<br />
<br />
Set kubectl access:
```bash
mkdir -p $HOME/.kube
```
<br />
<br />
```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
<br />
<br />
Test access to cluster:
```bash
kubectl get nodes
```
<br />
<br />
Install the Calico Network Add-On -
On the Control Plane Node, install Calico Networking:
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
<br />
<br />
```bash
kubectl get nodes
```
<br />
<br />
Join the Worker Nodes to the Cluster
In the Control Plane Node, create the token and copy the kubeadm join command 
**NOTE:The join command can also be found in the output from kubeadm init command**
```bash
kubeadm token create --print-join-command
```
<br />
<br />
Note : In both Worker Nodes, **paste the kubeadm join command** to join the cluster. Use sudo to run it as root:
```sudo kubeadm join ....................<rest-of-the-command>.......................```
<br />
<br />
In the Control Plane Node, view cluster status (Note: You may have to wait a few moments to allow all nodes to become ready)

The port number which shows there should be enabled, **6443** for example. **Enable that in security group in EC2 instance.**
<br />
<br />
Validate the setup by executing below commnad in master node
```bash
kubectl get nodes
```
