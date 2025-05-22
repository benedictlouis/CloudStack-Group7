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
- ``` sudo -i ``` Berguna untuk masuk ke root user, hal ini dilakukan karena banyak konfigurasi yang hanya bisa dilakukan oleh root.
- ``` cd /etc/netplan ``` Digunakan untuk berpindah ke direktori konfigurasi Netplan, yang digunakan di Ubuntu untuk mengatur jaringan.
- ``` nano ./*.yaml ``` Untuk membuka file konfigurasi, yaml yang ada di dalam folder Netplan dengan editor teks nano. Tanda * artinya semua file .yaml akan cocok, berguna jika Anda tidak tahu nama persisnya.

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
- ``` netplan generate ``` Memeriksa dan validasi file YAML yang sudah ditulis.
- ``` netplan apply ``` Menerapkan konfigurasi jaringan.
- ``` reboot ```  Mereboot sistem agar konfigurasi benar-benar aktif.
  
#### Menguji konfigurasi jaringan

```
ip add
ping 8.8.8.8
```
- ``` ip add ``` Mengecek apakah IP sudah diterapkan ke interface.
- ``` ping 8.8.8.8 ``` Menguji koneksi jaringan dan DNS ke internet.

#### Mengakifkan SSH login sebagai root user

```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
service ssh restart     
```
- ``` sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config ``` Mengedit file konfigurasi SSH untuk mengaktifkan login root dengan password.
- ``` service ssh restart ``` Restart layanan SSH agar perubahan konfigurasi langsung aktif.

### Instalasi Management Server  

#### Membuat file cloudstack.list di dalam direktori /etc/apt/sources.list.d/

```
nano /etc/apt/sources.list.d/cloudstack.list
```
- Membuat file sumber repository baru agar sistem mengenali paket CloudStack.
  
#### Menambahkan repository cloudstack ke dalam cloudstack.list

```
deb https://download.cloudstack.org/ubuntu noble 4.20
```
- Menambahkan alamat repository CloudStack 4.20 untuk Ubuntu Noble.

#### Menambahkan public key ke trusted keys

```
wget -O - https://download.cloudstack.org/release.asc |sudo tee /etc/apt/trusted.gpg.d/cloudstack.asc
```
- Mendownload dan menyimpan public GPG key agar apt bisa memverifikasi integritas paket dari CloudStack.

### Memperbarui local apt cache

```
apt update
```
- Update database ``` apt ``` agar mengenali paket dari repo baru.

### Install cloudstack-management

```
apt install cloudstack-management
```
- Install CloudStack Management Server.

### Install MySQL

```
apt install mysql-server
```
- Menginstall server database MySQL.

### Konfigurasi file mysqld.cnf

#### Membuka file mysqld.cnf 

```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
- Membuka file konfigurasi utama MySQL.

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
- Restart MySQL agar konfigurasi baru diterapkan.

#### Cek status service MySQL

```
systemctl status mysql
```
- Cek apakah MySQL berjalan normal.

### Database setup 

#### Menggunakan script cloudstack-setup-databases

```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:root -i 192.168.1.107
```
- Buat dan deploy database CloudStack. ``` cloud:cloud ``` adalah username dan password DB. ``` -i  ``` menunjuk ke IP Management Server.

#### Mengubah cluster.node.IP di dalam db.properties

```
cluster.node.IP=192.168.1.107
```

### Konfigurasi Management Server sebagai NFS Server

#### Instalasi nfs-kernel-server

```
apt install nfs-kervel-server
```
- Install layanan NFS.

#### Membuat dua direktori

```
mkdir -p /export/primary /export/secondary
```
- Buat dua folder untuk storage utama dan sekunder.

#### Mengkonfigurasi direktori baru sebagai NFS exports di /etc/exports

```
nano /etc/exports
```
- File ini mengatur folder mana yang bisa diakses melalui NFS.

```
/export *(rw,async,no_root_squash,no_subtree_check)
```
Membuka akses penuh ke folder ``` /export ``` dari semua IP.

#### Export direktori /export

```
exportfs -a
```
- Terapkan ekspor direktori NFS.

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
- Restart layanan NFS.

#### Menyiapkan System VM Template

```
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt -m /mnt/secondary -u http://download.cloudstack.org/systemvm/4.20/systemvmtemplate-4.20.0-x86_64-kvm.qcow2.bz2 -h kvm -F
```
- Unduh dan pasang template VM sistem ke storage sekunder.
- ``` -m ``` : mount point
- ``` -u ``` : URL file qcow2
- ``` -h ``` : hypervisor
- ``` -F ``` : paksa overwrite jika ada

### Konfigurasi Host KVM

#### Instalasi KVM

```
apt-get install qemu-kvm cloudstack-agent
```
- Install hypervisor KVM dan agent CloudStack.

#### Mengaktifkan VNC untuk console proxy

```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf
```
- Aktifkan VNC dari semua IP agar Console Proxy bisa bekerja.

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
- Tambahkan argumen agar ``` libvirtd ``` bisa menerima koneksi TCP.

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
- Nonaktifkan socket lama, lalu restart service.
  
#### Menonaktifkan AppArmor

```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```
- AppArmor kadang mengganggu operasi ``` libvirt ``` . Perintah ini menonaktifkan profil terkait.

### Menjalankan Management Server

```
cloudstack-setup-management
```
- Menyelesaikan setup dan konfigurasi service CloudStack.

#### Melihat status cloudstack-management

```
systemctl status cloudstack-management
```
- Cek status layanan.

### Mengakses dashboard Cloudstack

```
http://<IP_ADDRESS>:8080
http://192.168.1.107:8080
```

Login dengan username **admin** and password **password**

### Konfigurasi Zone

### Menambahkan Zone
![picture 24](https://i.imgur.com/nluyOJ0.png)  
Pada menu Zone, pengguna akan disajikan tampilan awal seperti berikut. Pengguna perlu memilih jenis zone yang sesuai dengan kebutuhan:
- Core: Dirancang untuk data center, tipe ini memiliki jangkauan jaringan yang lebih luas dan mendukung berbagai fitur tambahan di Apache CloudStack.
- Edge: Merupakan tipe zone ringan (lightweight) yang cocok untuk layanan publik atau aplikasi yang diakses dari luar. Fitur yang tersedia lebih terbatas dibandingkan tipe Core.

### Memilih tipe Zone
![picture 25](https://i.imgur.com/R7n20XP.png)  
- Advanced: Menyediakan fleksibilitas lebih tinggi dan memungkinkan pembuatan beberapa jaringan, baik publik maupun privat.
- Basic: Lebih simpel, seluruh VM berada dalam satu jaringan yang sama, sehingga cocok digunakan untuk keperluan pengujian.

### Membuat detail Zone
![picture 26](https://i.imgur.com/LqBQUup.png)  
Di bagian Zone details, pengguna diminta untuk mengonfigurasi informasi dasar terkait zone (data center) yang akan dibuat. Informasi yang perlu diisikan meliputi:
```
Name : GROUP7-ZONE
IPv4 DNS1 : 8.8.8.8
Internal DNS 1 : 192.168.1.1
Hypervisor : KVM
Default Guest CIDR : 10.1.1.0/24
```

### Mengkonfigurasi Network Cloudstack

#### Physical Network
![picture 27](https://i.imgur.com/IJY5Voj.png)  
Pada tahap Network, pengguna mengatur jaringan fisik dalam zone, termasuk metode isolasi (seperti VLAN) dan jenis lalu lintas (seperti Guest, Management, dan Public). Jaringan ini menentukan bagaimana VM dan layanan saling terhubung serta berinteraksi dengan jaringan luar.

#### Public traffic
![picture 28](https://i.imgur.com/PeMarCa.png) 
Pada bagian ini, pengguna harus menentukan rentang alamat IP publik yang akan digunakan untuk menghubungkan instance (VM) ke internet. Informasi yang dikonfigurasi meliputi:
- Gateway: Alamat gateway jaringan publik 
- Netmask: Subnet mask jaringan 
- VLAN/VNI: Biasanya dikosongkan jika tidak menggunakan VLAN khusus
- Start IP – End IP: Rentang alamat IP publik yang disediakan
IP publik ini akan digunakan untuk mengatur NAT antara jaringan Guest dan jaringan publik.

#### Pod
![picture 29](https://i.imgur.com/Hm7S6Ly.png)  
Pada tahap Pod, pengguna diminta untuk membuat pod pertama dalam zone dengan mengisi nama pod serta rentang IP yang dicadangkan untuk lalu lintas manajemen internal CloudStack. Informasi yang dimasukkan meliputi gateway, netmask, serta IP awal dan akhir yang dicadangkan.
```
Pod name: GROUP7_POD
Reserved system gateway: 192.168.1.1
Reserved system netmask: 255.255.255.0
Start reserved system IP: 192.168.1.51
End reserved system IP: 192.168.1.80
```

#### Guest traffic
![picture 30](https://i.imgur.com/Tl5HlTA.png)  
Pada tahap Guest traffic, pengguna menetapkan rentang VLAN/VNI untuk mengatur lalu lintas jaringan antar instance milik pengguna. Rentang ini digunakan untuk memisahkan dan mengelola komunikasi VM dalam jaringan Guest.

### Add resources

#### Cluster
![picture 31](https://i.imgur.com/ZVPzobp.png)  
Langkah ini, setiap kumpulan host menggunakan jenis hypervisor sama. Semua host di dalam cluster ini akan berada di subnet yang sama dan mengakses shared storage yang sama pula.

#### IP address
![picture 32](https://i.imgur.com/GcoGr1a.png)  
Lalu, konfigurasi informasi dari host agar host dapat bekerja di dalam CloudStack.

#### Primary storage
![picture 33](https://i.imgur.com/LCErt8i.png)  
Storage ini, tempat menyimpan disk utama VM yang berjalan pada semua host yang ada di dalam cluster.

#### Secondary storage
![picture 34](https://i.imgur.com/Fn7PiO2.png)  
Sedangkan storage ini, tempat untuk menyimpan template, ISO, dan Snapshot.

### Launch Zone
![picture 35](https://i.imgur.com/JZfYH82.png)  
Terakhir, bentuk launch zone agar zone akan aktif dan dapat digunakan untuk menjalankan VM.

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
