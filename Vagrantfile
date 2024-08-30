# -*- mode: ruby -*-
# vi: set ft=ruby :

# ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'

Vagrant.configure(2) do |config|
    config.vm.box = "almalinux/9"
    config.vm.synced_folder ".", "/vagrant"
    config.vm.provider "virtualbox" do |v|
      v.memory = 1024
      v.cpus = 1
    end
  
    config.vm.define "rpm" do |rpm|
      rpm.vm.network "private_network", ip: "192.168.50.10", virtualbox__intnet: "net1"
      rpm.vm.hostname = "rpm"
      rpm.vm.provision "shell", path: "config.sh"
    end  
  end