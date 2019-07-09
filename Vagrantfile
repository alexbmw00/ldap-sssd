# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.box_check_update = false
 
    config.vm.define "opensuse" do |sus|
      sus.vm.hostname="opensuse"
      sus.vm.box = "opensuse/openSUSE-15.0-x86_64"
      sus.vm.network "private_network", ip: "27.11.90.10"
    end

    config.vm.define "debian" do |deb|
      deb.vm.hostname="debian"
      deb.vm.box = "debian/stretch64"
      deb.vm.network "private_network", ip: "27.11.90.20"
    end

    config.vm.define "centos" do |centos|
      centos.vm.hostname="centos"
      centos.vm.box = "centos/7"
      centos.vm.network "private_network", ip: "27.11.90.30"
    end
  
end
