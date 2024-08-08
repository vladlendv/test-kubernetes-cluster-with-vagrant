ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'

Vagrant.configure("2") do |config|
  config.vm.provision "shell", inline: <<-SHELL
  apt-get update -y
  echo "100.0.0.1  master" >> /etc/hosts
  echo "100.0.0.2  worker" >> /etc/hosts
  SHELL
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