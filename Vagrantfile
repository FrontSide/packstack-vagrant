# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  config.vm.hostname = "packstack"

  config.vm.box = "centos/7"

  # The line disabling sync from the current directory 
  # to /vagrant inside the VM has been removed 
  # as we need to make the answers.txt file available. 

  config.vm.provider "virtualbox" do |vb|
    vb.cpus = 2
    vb.memory = "4096"
  end

  # rules to allow access to vagrant ports from local machine
  # - keystone auth service 
  config.vm.network "forwarded_port", guest: 5000, host: 5000, id: "keystone"
  # - nova compute service 
  config.vm.network "forwarded_port", guest: 8774, host: 8774, id: "nova"
  # - glance image service 
  config.vm.network "forwarded_port", guest: 9292, host: 9292, id: "glance"
  # - neutron network service 
  config.vm.network "forwarded_port", guest: 9696, host: 9696, id: "neutron"
  # - cinder volume service 
  config.vm.network "forwarded_port", guest: 8776, host: 8776, id: "cinder"




  config.vm.provision "shell", privileged: false, inline: <<-SHELL

    ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa -q
    sudo bash -c "ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa -q"
    sudo bash -c "umask 077 && cat ~vagrant/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys"

    sudo yum -y install centos-release-openstack-rocky
    sudo yum -y install openstack-packstack
    sudo yum -y install e2fsprogs

    sudo systemctl disable NetworkManager
    sudo systemctl stop NetworkManager

    packstack --answer-file=/vagrant/answers.txt
  SHELL
end
