# Instalasi Apache Cloudstack

## Group 7
- Christopher Satya - 2206059755
- Louis Benedict Archie - 2206025224
- Michael Winston Tjahaja - 2106731270
- Rifqi Ramadhan - 2206062964
- Tjokorde Gde Agung Abel - 2206059736

### Konfigurasi Jaringan

#### Mengubah isi file konfigurasi network

```
sudo -i
cd /etc/netplan
nano ./*.yaml
```

```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    cloudbr0:
      addresses: [192.168.1.0/24]  
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1,8.8.8.8]
      interfaces: [enp1s0]
      dhcp4: false
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```

#### Menerapkan konfigurasi jaringan

```
netplan generate        
netplan apply           
reboot                  
```

#### Menguji konfigurasi jaringan

```
ip add
ping 8.8.8.8
```

#### Mengakifkan SSH login sebagai root user

```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
service ssh restart     
```

### Instalasi Management Server  

#### Membuat file cloudstack.list di dalam direktori /etc/apt/sources.list.d/

```
nano /etc/apt/sources.list.d/cloudstack.list
```

#### Menambahkan repository cloudstack ke dalam cloudstack.list

```
deb https://download.cloudstack.org/ubuntu noble 4.20
```

#### Menambahkan public key ke trusted keys

```
wget -O - https://download.cloudstack.org/release.asc |sudo tee /etc/apt/trusted.gpg.d/cloudstack.asc
```

### Memperbarui local apt cache

```
apt update
```

### Install cloudstack-management

```
apt install cloudstack-management
```

### Install MySQL

```
apt install mysql-server
```

### Konfigurasi file mysqld.cnf

#### Membuka file mysqld.cnf 

```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

#### Menambahkan baris berikut di bawah bagian [mysqld]

```
server-id=1
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=350
log-bin=mysql-bin
binlog-format = 'ROW'
```

#### Restart MySQL service

```
systemctl restart mysql
```

#### Cek status service MySQL

```
systemctl status mysql
```

### Database setup 

#### Menggunakan script cloudstack-setup-databases

```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:root -i 192.168.1.107
```

#### Mengubah cluster.node.IP di dalam db.properties

```
cluster.node.IP=192.168.1.107
```

### Konfigurasi Management Server sebagai NFS Server

#### Instalasi nfs-kernel-server

```
apt install nfs-kervel-server
```

#### Membuat dua direktori

```
mkdir -p /export/primary /export/secondary
```
#### Mengkonfigurasi direktori baru sebagai NFS exports di /etc/exports

```
nano /etc/exports
```

```
/export *(rw,async,no_root_squash,no_subtree_check)
```

#### Export direktori /export

```
exportfs -a
```

#### Mengkonfigurasi NFS Server

```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
```

#### Restart layanan NFS

```
service nfs-kernel-server restart
```

#### Menyiapkan System VM Template

```
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt -m /mnt/secondary -u http://download.cloudstack.org/systemvm/4.20/systemvmtemplate-4.20.0-x86_64-kvm.qcow2.bz2 -h kvm -F
```

### Konfigurasi Host KVM

#### Instalasi KVM

```
apt-get install qemu-kvm cloudstack-agent
```

#### Mengaktifkan VNC untuk console proxy

```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf
```

#### Konfigurasi file libvirtd.conf

```
nano /etc/libvirt/libvirtd.conf
```

```
listen_tls=0  
echo 'listen_tcp=1' 
echo 'tcp_port = "16509"' 
echo 'mdns_adv = 0'
echo 'auth_tcp = "none"
```

#### Konfigurasi file libvirtd

```
nano /etc/default/libvirtd
```

```
LIBVIRTD_ARGS="--listen"
```

#### Konfigurasi file libvirt.conf

```
nano /etc/libvirt/libvirt.conf
```

```
remote_mode="legacy"
```

#### Restart libvirt

```
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

#### Menonaktifkan AppArmor

```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

### Menjalankan Management Server

```
cloudstack-setup-management
```

#### Melihat status cloudstack-management

```
systemctl status cloudstack-management
```

### Mengakses dashboard Cloudstack

```
http://<IP_ADDRESS>:8080
http://192.168.1.107:8080
```

Login dengan username **admin** and password **password**

### Konfigurasi Zone

### Menambahkan Zone
![picture 24](https://i.imgur.com/nluyOJ0.png)  

### Memilih tipe Zone
![picture 25](https://i.imgur.com/R7n20XP.png)  

### Membuat detail Zone
![picture 26](https://i.imgur.com/LqBQUup.png)  

### Mengkonfigurasi Network Cloudstack

#### Physical Network
![picture 27](https://i.imgur.com/IJY5Voj.png)  

#### Public traffic
![picture 28](https://i.imgur.com/PeMarCa.png)  

#### Pod
![picture 29](https://i.imgur.com/Hm7S6Ly.png)  

#### Guest traffic
![picture 30](https://i.imgur.com/Tl5HlTA.png)  


### Add resources

#### Cluster
![picture 31](https://i.imgur.com/ZVPzobp.png)  

#### IP address
![picture 32](https://i.imgur.com/GcoGr1a.png)  

#### Primary storage
![picture 33](https://i.imgur.com/LCErt8i.png)  

#### Secondary storage
![picture 34](https://i.imgur.com/Fn7PiO2.png)  

### Launch Zone
![picture 35](https://i.imgur.com/JZfYH82.png)  

### Register ISO
![picture 36](https://i.imgur.com/cqY8Lec.png)  

### Membuat compute offering
![picture 37](https://i.imgur.com/dzZt0RZ.png)  

### Egress rules
![picture 38](https://i.imgur.com/AyfuVe8.png)  

### Firewall
![picture 39](https://i.imgur.com/QyWglff.png)  

### Port forwarding
![picture 40](https://i.imgur.com/rgE3Sse.png) 

### Membuat instance
![picture 41](https://i.imgur.com/Xf07Qjh.png)  
![picture 42](https://i.imgur.com/g5nfbzo.png)  
![picture 43](https://i.imgur.com/ofeWqBl.png)  

### Menjalankan instance
![picture 44](https://i.imgur.com/ofM6DLv.png)  

### Akses internet 
![picture 45](https://i.imgur.com/eCrq9ND.png)  

### SSH ke VM 
![picture 46](https://i.imgur.com/aCmr4K5.png)  
