# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.box_check_update = false
 
    config.vm.define "ldap" do |sus|
      sus.vm.hostname="ldap"
      sus.vm.box = "opensuse/openSUSE-15.0-x86_64"
      sus.vm.network "private_network", ip: "192.168.99.10"
    end

    config.vm.define "debian" do |deb|
      deb.vm.hostname="debian"
      deb.vm.box = "debian/stretch64"
      deb.vm.network "private_network", ip: "192.168.99.20"
      # apt-get install sssd libpam-sss libnss-sss
    end

    config.vm.define "centos" do |centos|
      centos.vm.hostname="centos"
      centos.vm.box = "centos/7"
      centos.vm.network "private_network", ip: "192.168.99.30"
    end
  
end
