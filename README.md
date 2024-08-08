Guide to setup two and more nodes for Kubernetes cluster.

Prerequisites

1. Vagrant 2.2.15 or latest
2. VM VirtualBox

Step 1 - Start vagrant box
As a minimum requirement for kubernetes installation we need 
1. Master Node - 2 cpus, 2 GB Memory
2. Worker Node - 1 cpu, 1 GB Memory

Use following Vagrantfile or at least create a Vagrantfile and copy the following configuration into it:

NOTE: to bypass the blocking we use the following link at the beginning of the file - ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'

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
Step 2 - Update host files on both master and worker node

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
100.0.0.1 master.jhooq.com master
100.0.0.2 worker.jhooq.com worker
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
100.0.0.1 master.jhooq.com master
100.0.0.2 worker.jhooq.com worker
```
Test the worker node by sending from master
```
ping worker
```
example of expected output
```
PING worker.jhooq.com (100.0.0.2) 56(84) bytes of data.
64 bytes from worker.jhooq.com (100.0.0.2): icmp_seq=1 ttl=64 time=0.462 ms
64 bytes from worker.jhooq.com (100.0.0.2): icmp_seq=2 ttl=64 time=0.686 ms
```
Test the master node by sending from worker
```
ping master
```
example of expected output
```
PING master.jhooq.com (100.0.0.1) 56(84) bytes of data.
64 bytes from master.jhooq.com (100.0.0.1): icmp_seq=1 ttl=64 time=0.238 ms
64 bytes from master.jhooq.com (100.0.0.1): icmp_seq=2 ttl=64 time=0.510 ms
```