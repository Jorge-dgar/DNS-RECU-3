# coding: utf-8
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.box = "debian/bullseye64"
  
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "256"
    vb.linked_clone = true
  end 

  # Actualizar SO
  config.vm.provision "update", type:"shell", inline: <<-SHELL
    apt-get update 
  SHELL

  ######################################################################
  #
  # Servidor 'atlas' (Primario)
  #
  ######################################################################
  
  config.vm.define "atlas" do |atlas|
    atlas.vm.hostname = "atlas"
    atlas.vm.network "private_network", ip: "192.168.57.10"
    
    atlas.vm.provision "bind9-install", type:"shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL

    atlas.vm.provision "config", type:"shell", inline: <<-SHELL
      cp -v /vagrant/resolv.conf /etc/resolv.conf
      cp -v /vagrant/named.conf.options /etc/bind/
      cp -v /vagrant/named.conf.local /etc/bind/
      cp -v /vagrant/olimpo.test.dns /etc/bind/
      cp -v /vagrant/192.168.57.dns /etc/bind/
      systemctl restart bind9
      systemctl status bind9
    SHELL
  end 

  ######################################################################
  #
  # Servidor 'ceo' (Secundario)
  #
  ######################################################################
  
  config.vm.define "ceo" do |ceo|
    ceo.vm.hostname = "ceo"
    ceo.vm.network "private_network", ip: "192.168.57.11"
    
    ceo.vm.provision "bind9-install", type:"shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL

    ceo.vm.provision "config", type:"shell", inline: <<-SHELL
      cp -v /vagrant/resolv.conf /etc/resolv.conf
      cp -v /vagrant/named.conf.options /etc/bind/
      cp -v /vagrant/named.conf.local /etc/bind/
      systemctl restart bind9
      systemctl status bind9
    SHELL
  end 

end