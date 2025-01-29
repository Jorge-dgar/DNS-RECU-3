# Configuración de Servidores DNS en Olimpo.test

Este documento describe los pasos realizados para configurar los servidores DNS en la red `olimpo.test`, utilizando Vagrant para crear dos máquinas virtuales:

- **atlas** (servidor primario)
- **ceo** (servidor secundario)

## 1. Configuración de los Servidores

### 1.1 Vagrantfile

Se ha utilizado el siguiente `Vagrantfile` para definir y provisionar las máquinas virtuales:

```ruby
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

  # Configuración de Atlas
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
    SHELL
  end

  # Configuración de CEO
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
    SHELL
  end

end
```

### 1.2 Configuración de los Archivos DNS

#### `resolv.conf`
```plaintext
search olimpo.test
nameserver 127.0.0.1
nameserver 1.1.1.1
```

#### `named.conf.options`
```plaintext
options {
    directory "/var/cache/bind";
    forwarders {
        1.1.1.1;
    };
    dnssec-validation no;
    listen-on-v6 { any; };
};
```

#### `named.conf.local`
```plaintext
zone "olimpo.test" {
    type master;
    file "/etc/bind/olimpo.test.dns";
    allow-transfer { 192.168.57.11; };
    notify yes;
};

zone "57.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/192.168.57.dns";
    allow-transfer { 192.168.57.11; };
    notify yes;
};
```

#### `olimpo.test.dns`
```plaintext
$TTL 86400
@   IN  SOA atlas. hefestos.olimpo.test. (
        2024012901 ; Serial
        1200       ; Refresh (20 min)
        900        ; Retry (15 min)
        2419200    ; Expire
        86400      ; Negative Cache TTL
)

@       IN  NS  atlas.
atlas   IN  A   192.168.57.10
ceo     IN  A   192.168.57.11
atenea  IN  A   192.168.57.20
mercurio IN A   192.168.57.30
ares    IN  A   192.168.57.40
ftp     IN  CNAME atenea.
smtp    IN  CNAME mercurio.
pop     IN  CNAME mercurio.
www     IN  CNAME ares.

@       IN  MX  10 mercurio.
@       IN  MX  20 dionisio.
```

### `192.168.57.dns` (Zona Inversa)
```plaintext
$TTL 86400
@   IN  SOA atlas. hefestos.olimpo.test. (
        2024012901 ; Serial
        1200       ; Refresh (20 min)
        900        ; Retry (15 min)
        2419200    ; Expire
        86400      ; Negative Cache TTL
)

@       IN  NS  atlas.
10      IN  PTR atlas.olimpo.test.
11      IN  PTR ceo.olimpo.test.
20      IN  PTR atenea.olimpo.test.
30      IN  PTR mercurio.olimpo.test.
40      IN  PTR ares.olimpo.test.
```

## 2. Pruebas Realizadas

### 2.1 Desde Atlas
```sh
sudo systemctl status bind9
```
![Revisamos el estado de bind9 en atlas](https://github.com/user-attachments/assets/f20b209c-ce20-4233-a959-ea8cbeb601ff)

```sh
dig @localhost atlas.olimpo.test
```
![Test en atlas olimpo test](https://github.com/user-attachments/assets/6ff8c894-c1fe-4105-9dbc-9163282c57e2)

```sh
dig @localhost -x 192.168.57.10
```
![Test en atlas con la IP 192.168.57.10](https://github.com/user-attachments/assets/b65b73a0-fd5e-4a8e-b23c-c4d58bf1ffc5)

### 2.2 Desde CEO
```sh
sudo systemctl status bind9
```
![Estatus de bind9 desde máquina ceo](https://github.com/user-attachments/assets/6b1f09af-8ea7-446a-8aed-5d46d38962a0)

### 2.3 Verificación de Transferencia de Zona en CEO
```sh
ls -l /var/cache/bind/
```
![Verificación transferencia de zona desde CEO](https://github.com/user-attachments/assets/199d800c-391c-4a95-9ef6-1ec67f28dc2d)

### 2.4 Captura de Transferencia de Zona en Atlas
```sh
sudo journalctl -u bind9 --since "10 minutes ago"
```
![Captura de transferencia de zona](https://github.com/user-attachments/assets/ae5cea5e-2c8c-46d2-afa1-31cf2635f3b1)
