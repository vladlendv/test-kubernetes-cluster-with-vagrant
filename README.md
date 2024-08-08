# Step by step guide to setup Multi-node Kubernetes cluster on a local machine using Vagrant and Kubeadm

In this example we will create a cluster with one master node and two workers.

### Prerequisites
1. Vagrant
2. VM VirtualBox

As a minimum requirement for kubernetes installation we need 
1. Master Node - 2 cpus, 2 GB Memory
2. Worker Node - 2 cpu, 2 GB Memory

## Step 1 - Copy repo and start vagrant box
To get started, copy this repository to your computer and go to its directory to execute commands, because we use helper scripts in ubuntu folder to build vagrant.
```
git clone https://github.com/vladlendv/test-kubernetes-cluster-with-vagrant.git
```
You can also set the desired number of nodes by editing these values ​​in the wagent file. If this number is changed, remember to update setup-hosts.sh script with the new hosts IP details in /etc/hosts of each VM.

NOTE: to bypass the blocking we use the following link at the beginning of the file
```
ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'
```
Start you virtual boxes by starting up your vagrant box
```
vagrant up
```
## Step 2 - Update host files(on all nodes)
This stage has been added for automatic execution in the vargant file, for all nodes, so you don't have to worry about it. You can only check that the connection between nodes is available by their name.

Test the worker node by sending from master
```
ping node01
ping node02
```
Test the master node by sending from worker
```
ping controlplane01
```
example of expected output
```
PING controlplane01 (100.0.0.1) 56(84) bytes of data.
64 bytes from controlplane01 (100.0.0.1): icmp_seq=1 ttl=64 time=0.238 ms
64 bytes from controlplane01 (100.0.0.1): icmp_seq=2 ttl=64 time=0.510 ms
```

log in to all nodes via SSH as shown below, for ease of further configuration you can open them in separate terminals:
```
vagrant ssh controlplane01
```
```
vagrant ssh node01
vagrant ssh node02
```

## Step 3 - Enable IPv4 packet forwarding(on all nodes)
```
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
```
```
# Apply sysctl params without reboot
sudo sysctl --system
```
Verify that net.ipv4.ip_forward is set to 1 with:
```
sysctl net.ipv4.ip_forward
```

## Step 4 - Install Container Runtime - in this example we use containerd(on all nodes)
You need to install containerd on all nodes.
So run the following installation command:
```
sudo apt-get update
```
```
sudo apt-get install ca-certificates curl
```
```
sudo install -m 0755 -d /etc/apt/keyrings
```
```
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```
```
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
```
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
sudo apt-get update
```
```
sudo apt install -y containerd.io
```

## Step 5 - Disable the firewall and turnoff the "swapping"(on all nodes)
We need to disable firewall as well as swapping on master as well as worker node. Because to install kubernetes we need to disable the swapping on both the nodes
```
sudo ufw disable
```
```
sudo swapoff -a
```

## Step 6 - Configuring the systemd cgroup driver
```
sudo -i
```
```
containerd config default > /etc/containerd/config.toml
```
```
exit
```
```
sudo systemctl restart containerd.service
```
```
sudo vi /etc/containerd/config.toml
```
replace the following line
```
SystemdCgroup = false
```
to the true
```
SystemdCgroup = true
```
save file and exit
```
sudo systemctl restart containerd.service
```

## Step 7 - Installing kubeadm, kubelet and kubectl(on all nodes)
1. Update the apt package index and install packages needed to use the Kubernetes apt repository:
apt-transport-https may be a dummy package; if so, you can skip that package
```
sudo apt-get update
```
```
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
2. Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:

If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
sudo mkdir -p -m 755 /etc/apt/keyrings
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
NOTE: In releases older than Debian 12 and Ubuntu 22.04, directory /etc/apt/keyrings does not exist by default, and it should be created before the curl command.

3. Add the appropriate Kubernetes apt repository. Please note that this repository have packages only for Kubernetes 1.30; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).
This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
4. Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
```
```
sudo apt-get install -y kubelet kubeadm kubectl
```
```
sudo apt-mark hold kubelet kubeadm kubectl
```
5. (Optional) Enable the kubelet service before running kubeadm:
```
sudo systemctl enable --now kubelet
```
The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.

## Step 8 - Initializing your control-plane node(only run on master)
Okay now we have reach to point where we have done all the prerequisite for initializing the kubernetes cluster.
**Let's run the kubernetes initialization command on only on master**
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/24 --apiserver-advertise-address=192.168.77.11
```
Note down kubeadm join command which we are going to use from worker node to join the master node using token. *(Note : - Followig command will be different for you, do not try copy the following command)*
```
sudo kubeadm join 192.168.77.11:6443 --token 71zj3b.4zff1ufs669wpbdl         --discovery-token-ca-cert-hash sha256:77edffec2e21206191dbc3f996d86b2f040c1793ea581c2148707cc1be9ee4d2
```

## Step 9 - Move kube config file to current user(only run on master)
To interact with the kubernetes cluster and to user kubectl command, we need to have the kube config file with us.
Use the following command to get the kube config file and put it under working directory.
```
mkdir -p $HOME/.kube
```
```
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
```
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Step 10 - Installing a Pod network addon(only run on master)

```
kubectl apply -f https://reweave.azurewebsites.net/k8s/v1.30/net.yaml
```
edit configuration file for network plugin weave-net
```
kubectl -n kube-system edit ds weave-net
```
in the weave container add an additional env variable
```
- name: IPALLOC_RANGE
  value: 10.244.0.0/24
```

## Step 11 - Join worker nodes to master(only run on worker nodes)
In the Step 8 we generated the token and kubeadm join command.
Now we need to use that join command from our worker node
*(Note : - Followig command will be different for you, do not try copy the following command)*
```
sudo kubeadm join 192.168.77.11:6443 --token 71zj3b.4zff1ufs669wpbdl         --discovery-token-ca-cert-hash sha256:77edffec2e21206191dbc3f996d86b2f040c1793ea581c2148707cc1be9ee4d2
```
and wait for the conclusion
```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

## Step 12 - Check the nodes status(only run on master)
To check the status of the nodes use
```
kubectl get nodes
```
```
NAME             STATUS   ROLES           AGE   VERSION
controlplane01   Ready    control-plane   83m   v1.30.3
node01           Ready    <none>          66m   v1.30.3
node02           Ready    <none>          66m   v1.30.3
```
That's it, have fun :)


