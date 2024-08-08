# Step by step guide to setup Multi-node Kubernetes cluster on a local machine using Vagrant and Kubeadm

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
ENV['VAGRANT_SERVER_URL'] ='https://vagrant.elab.pro'

# -*- mode: ruby -*-
# vi:set ft=ruby sw=2 ts=2 sts=2:

# Define the number of master and worker nodes
# If this number is changed, remember to update setup-hosts.sh script with the new hosts IP details in /etc/hosts of each VM.
NUM_MASTER_NODE = 1 
NUM_WORKER_NODE = 2

IP_NW = "192.168.77."
MASTER_IP_START = 1
NODE_IP_START = 2
LB_IP_START = 30


# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  # config.vm.box = "base"
  config.vm.box = "ubuntu/jammy64"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  config.vm.box_check_update = false

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.
  
  # Provision Load Balancer Node
  
  if NUM_MASTER_NODE > 1 
    config.vm.define "lb" do |node|

      node.vm.provider "virtualbox" do |vb|
        vb.name = "lb"
        vb.memory = 2048
        vb.cpus = 2

      end

      node.vm.hostname = "lb"
      node.vm.network :private_network, ip: IP_NW + "#{LB_IP_START}"
      node.vm.network "forwarded_port", guest: 22, host: "2730"

      node.vm.provision "setup-hosts", :type => "shell", :path => "ubuntu/setup-hosts.sh" do |s|
        s.args = ["enp0s8"]
      end

      node.vm.provision "setup-dns", type: "shell", :path => "ubuntu/update-dns.sh"
    end
  end

  # Provision Master Nodes
  (1..NUM_MASTER_NODE).each do |i|
      config.vm.define "controlplane0#{i}" do |node|
        # Name shown in the GUI
        node.vm.provider "virtualbox" do |vb|
            vb.name = "controlplane0#{i}"
            vb.memory = 2048
            vb.cpus = 2
        end
        node.vm.hostname = "controlplane0#{i}"
        node.vm.network :private_network, ip: IP_NW + "#{MASTER_IP_START}" + "#{i}"
        node.vm.network "forwarded_port", guest: 22, host: "271#{i}"

        node.vm.provision "setup-hosts", :type => "shell", :path => "ubuntu/setup-hosts.sh" do |s|
          s.args = ["enp0s8"]
        end

        node.vm.provision "setup-dns", type: "shell", :path => "ubuntu/update-dns.sh"

      end
  end


  # Provision Worker Nodes
  (1..NUM_WORKER_NODE).each do |i|
    config.vm.define "node0#{i}" do |node|
        node.vm.provider "virtualbox" do |vb|
            vb.name = "node0#{i}"
            vb.memory = 2048
            vb.cpus = 2
        end
        node.vm.hostname = "node0#{i}"
        node.vm.network :private_network, ip: IP_NW + "#{NODE_IP_START}" + "#{i}"
                node.vm.network "forwarded_port", guest: 22, host: "272#{i}"

        node.vm.provision "setup-hosts", :type => "shell", :path => "ubuntu/setup-hosts.sh" do |s|
          s.args = ["enp0s8"]
        end

        node.vm.provision "setup-dns", type: "shell", :path => "ubuntu/update-dns.sh"
    end
  end
end
```

Start you virtual boxes by starting up your vagrant box
```
vagrant up 
```
## Step 2 - Update host files(on both nodes)
This stage has been added for automatic execution in the vargant file, for all nodes, so you don't have to worry about it. You can only check that the connection between nodes is available by their name.

Test the worker node by sending from master
```
ping worker
```
Test the master node by sending from worker
```
ping master
```
example of expected output
```
PING master (100.0.0.1) 56(84) bytes of data.
64 bytes from master (100.0.0.1): icmp_seq=1 ttl=64 time=0.238 ms
64 bytes from master (100.0.0.1): icmp_seq=2 ttl=64 time=0.510 ms
```

To connect directly via SSH to nodes use the following commands:
```
vagrant ssh master
```
```
vagrant ssh worker
```
## Step 3 - Install Docker(on both nodes)
You need to install Docker on both the node.
So run the following installation command on both the nodes
```
sudo apt-get update
```
```
sudo apt install docker.io -y
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
## Step 4 - Disable the firewall and turnoff the "swapping"(on both nodes)
We need to disable firewall as well as swapping on master as well as worker node. Because to install kubernetes we need to disable the swapping on both the nodes
```
sudo ufw disable
```
```
sudo swapoff -a
```
## Step 5 - Installing kubeadm, kubelet and kubectl(on both nodes)
As a next step you will install these packages on all of your machines
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

## Step 6 - Let's enable cni pliugn in the configuration file /etc/containerd/config.toml(on both nodes)
if you dont have file /etc/containerd/config.toml
```
sudo mkdir /etc/containerd
```
```
sudo touch /etc/containerd/config.toml
```
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
## Step 7 - Initialize the kubernetes cluster(only run on master)
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


