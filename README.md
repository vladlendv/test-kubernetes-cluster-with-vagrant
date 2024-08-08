# Guide to setup two and more nodes for Kubernetes cluster.

### Prerequisites

1. Vagrant 2.2.15 or latest
2. VM VirtualBox

## Step 1 - Start vagrant box
As a minimum requirement for kubernetes installation we need 
1. Master Node - 2 cpus, 2 GB Memory
2. Worker Node - 1 cpu, 1 GB Memory

Use following Vagrantfile or at least create a Vagrantfile and copy the following configuration into it:

NOTE: to bypass the blocking we use the following link at the beginning of the file
```
ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'
```

```
ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'

Vagrant.configure("2") do |config|
  config.vm.define "master" do |master|
    master.vm.box_download_insecure = true    
    master.vm.box = "ubuntu/jammy64"
    master.vm.network "private_network", ip: "100.0.0.1"
    master.vm.hostname = "master"
    master.vm.provider "virtualbox" do |v|
      v.name = "master"
      v.memory = 2048
      v.cpus = 2
    end
  end

  config.vm.define "worker" do |worker|
    worker.vm.box_download_insecure = true 
    worker.vm.box = "ubuntu/jammy64"
    worker.vm.network "private_network", ip: "100.0.0.2"
    worker.vm.hostname = "worker"
    worker.vm.provider "virtualbox" do |v|
      v.name = "worker"
      v.memory = 1024
      v.cpus = 1
    end
  end

end
```

Start you virtual boxes by starting up your vagrant box
```
vagrant up 
```
## Step 2 - Update host files on both master and worker node

After starting the vagrant box now we need to login into the virtual machine using the command vagrant ssh master

master node - SSH into the master node
```
vagrant ssh master
```
Add host entry for master as well as worker node
```
sudo vi /etc/hosts
```
```
100.0.0.1 master.com master
100.0.0.2 worker.com worker
```
worker node - SSH into the master node
```
vagrant ssh worker
```
Add host entry for master as well as worker node
```
sudo vi /etc/hosts
```
```
100.0.0.1 master.com master
100.0.0.2 worker.com worker
```
Test the worker node by sending from master
```
ping worker
```
example of expected output
```
PING worker.com (100.0.0.2) 56(84) bytes of data.
64 bytes from worker.com (100.0.0.2): icmp_seq=1 ttl=64 time=0.462 ms
64 bytes from worker.com (100.0.0.2): icmp_seq=2 ttl=64 time=0.686 ms
```
Test the master node by sending from worker
```
ping master
```
example of expected output
```
PING master.com (100.0.0.1) 56(84) bytes of data.
64 bytes from master.com (100.0.0.1): icmp_seq=1 ttl=64 time=0.238 ms
64 bytes from master.com (100.0.0.1): icmp_seq=2 ttl=64 time=0.510 ms
```
## Step 3 - Install Docker on both master and worker node
You need to install Docker on both the node.
So run the following installation command on both the nodes
```
sudo apt-get update
```
```
sudo apt install docker.io
```
Enable and start docker
```
sudo systemctl enable docker
```
```
sudo systemctl start  docker
```
Check the docker service status
```
sudo systemctl status docker
```
## Step 4 - Disable the firewall and turnoff the "swapping"
We need to disable firewall as well as swapping on master as well as worker node. Because to install kubernetes we need to disable the swapping on both the nodes
```
sudo ufw disable
```
```
sudo swapoff -a
```
## Step 5 - Installing kubeadm, kubelet and kubectl
As a next step you will install these packages on all of your machines
1. Update the apt package index and install packages needed to use the Kubernetes apt repository:
```
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
2. Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
```
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
NOTE: In releases older than Debian 12 and Ubuntu 22.04, directory /etc/apt/keyrings does not exist by default, and it should be created before the curl command.

3. Add the appropriate Kubernetes apt repository. Please note that this repository have packages only for Kubernetes 1.30; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).
```
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
4. Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
5. (Optional) Enable the kubelet service before running kubeadm:
```
sudo systemctl enable --now kubelet
```
The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.

## Step 6 - Let's enable cni pliugn in the configuration file /etc/containerd/config.toml
```
containerd config default > /etc/containerd/config.toml
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
```
sudo systemctl restart containerd.service
```
## Step 7 - Initialize the kubernetes cluster
Okay now we have reach to point where we have done all the prerequisite for initializing the kubernetes cluster.
**Let's run the kubernetes initialization command on only on master**
```
sudo kubeadm init --apiserver-advertise-address=100.0.0.1
```
Note down kubeadm join command which we are going to use from worker node to join the master node using token. *(Note : - Followig command will be different for you, do not try copy the following command)*
```
sudo kubeadm join 100.0.0.1:6443 --token ixgzbr.effn6upbfszz8xiw \
        --discovery-token-ca-cert-hash sha256:75f77617955d94b788e3f3d9578dd84aff4b27b6036e73134db61da85ea1cf01
```
## Step 8 - Move kube config file to current user (only run on master)
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
## Step 9 - Apply CNI from kube-flannel.yml(only run on master)
```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
## Step 10 - Join worker nodes to master(only run on worker)
In the Step 7 we generated the token and kubeadm join command.
Now we need to use that join command from our worker node
*(Note : - Followig command will be different for you, do not try copy the following command)*
```
sudo kubeadm join 100.0.0.1:6443 --token ixgzbr.effn6upbfszz8xiw \
        --discovery-token-ca-cert-hash sha256:75f77617955d94b788e3f3d9578dd84aff4b27b6036e73134db61da85ea1cf01
```
and wait for the conclusion
```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```
## Step 11 - Check the nodes status(only run on master)
To check the status of the nodes use
```
kubectl get nodes
```
```
NAME     STATUS     ROLES           AGE   VERSION
master   Ready      control-plane   42h   v1.30.3
worker   NotReady   <none>          41h   v1.30.3
```
That's it, have fun :)


