# Apache CloudStack Single-Node Installation Guide

This guide provides a detailed, step-by-step process for installing and configuring Apache CloudStack on a single machine. In this setup, the same host will function as both the Management Server and the KVM Hypervisor.


## Group 7

- Christopher Satya - 2206059755
- Louis Benedict Archie - 2206025224
- Michael Winston Tjahaja - 2106731270
- Rifqi Ramadhan - 2206062964
- Tjokorde Gde Agung Abel - 2206059736

## Prerequisites and Preliminary System Setup

Before proceeding with the installation of CloudStack, it is crucial to ensure your server meets the necessary hardware and software requirements and has been properly configured. These preliminary steps will prevent common installation issues and provide a stable foundation for your cloud environment.

### 1. Hardware Requirements

- CPU: The processor must support hardware virtualization.
    - For Intel CPUs, this is Intel VT-x.
    - For AMD CPUs, this is AMD-V.
    - Action: You must enable virtualization technology in your server's BIOS or UEFI settings. This setting is often found under "CPU Configuration" or "Advanced" tabs.
- RAM: A minimum of 8 GB of RAM is recommended for a single-node test environment. This provides enough memory for the host operating system, the CloudStack Management Server, the KVM hypervisor, and a few small virtual machines. For better performance, 16 GB or more is highly recommended.
- Storage:
    - Disk Space: A minimum of 100 GB of storage is recommended to accommodate the operating system, CloudStack components, and storage for templates, ISOs, and virtual machine disks.
    - Disk Type: Using an SSD (Solid State Drive) is highly recommended for both primary and secondary storage, as it will dramatically improve the performance of VM provisioning and general I/O operations.
- Network: A stable network connection with a static IP address available for the server. This setup will use one network interface card (NIC).

### 2. Software Requirements

- This installation guide is done with  Ubuntu 24.04 (Noble Numbat). It is recommended to start with a fresh, minimal installation of this OS to avoid potential software conflicts.

### 3. Initial System Configuration

Before installing any CloudStack components, perform the following configuration steps.

#### Update the System

First, ensure all system packages are up-to-date. This applies the latest security patches and software updates.

```
sudo apt update
sudo apt full-upgrade -y
```
- **sudo apt update**: Refreshes the local package index.
- **sudo apt full-upgrade** -y: Upgrades all installed packages to their newest versions.

#### Configure the Firewall

It is critical to configure a firewall to secure your server before exposing services. The following steps use UFW (Uncomplicated Firewall) on Ubuntu.

1. Allow Essential Services: Add rules to permit traffic for SSH (so you don't get locked out), the CloudStack UI, the libvirt management port, and NFS.

    ```
    # Allow SSH for remote access
    sudo ufw allow ssh
    ```

    ```
    # Allow CloudStack UI
    sudo ufw allow 8080/tcp
    ```

    ```
    # Allow Libvirt for hypervisor management
    sudo ufw allow 16509/tcp
    ```

    ```
    # Allow NFS for storage
    sudo ufw allow nfs
    ```

2. Enable the Firewall: Activate the firewall to apply the rules.

    ```
    sudo ufw enable
    ```

3. Verify Status: Check that the firewall is active and the rules have been added correctly.

    ```
    sudo ufw status
    ```

## Network Configuration

First, we'll configure the server's network to use a static IP address on a bridge interface, which is required for CloudStack.

### Edit the Netplan Configuration File


### Define the Bridge Interface

Replace the entire content of the YAML file with the following configuration. This will disable DHCP and create a network bridge named cloudbr0. This bridge is essential for the guest VMs to communicate with the network.
    
> Note: Replace enp1s0 with your actual network interface name, which you can find by running the ip a command. Adjust the IP address, gateway (via), and DNS servers (addresses) to match your local network settings.

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
      addresses: [192.168.1.107/24]  
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      interfaces: [enp1s0]
      dhcp4: false
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```

### Apply the Network Configuration

Generate and apply the new network settings, then reboot the system to ensure they take effect properly.

```
netplan generate
netplan apply
reboot
```

- **netplan generate**: Checks the syntax of your .yaml file for any errors.
- **netplan apply**: Applies the new network configuration to the system.
- **reboot**: Restarts the machine to ensure all network services are re-initialized correctly.

### Verify the Network Configuration

After rebooting, log back in and verify that the new IP address has been assigned and that you have internet connectivity.

```
ip add
ping 8.8.8.8
```
- **ip add**: Displays all network interfaces and their assigned IP addresses. You should see cloudbr0 with the IP address.
- **ping 8.8.8.8**: Tests your server's connection to the internet by sending packets to Google's public DNS server.


### (Optional) Enable Root SSH Login

For easier management, you can enable direct SSH login for the root user.

```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
service ssh restart
```

- **sed ...**: This command edits the SSH daemon's configuration file (sshd_config) to add the line PermitRootLogin yes, allowing root login.
- **service ssh restart**: Restarts the SSH service to apply the changes.

## Management Server Installation

Now, we will install the CloudStack Management Server and its database backend, MySQL.

### Add the CloudStack Repository

Create a new repository file so your system can locate the CloudStack packages.

```
sudo nano /etc/apt/sources.list.d/cloudstack.list
```

Add the official CloudStack 4.20 repository for Ubuntu Noble (24.04) to this file.

```
deb https://download.cloudstack.org/ubuntu noble 4.20
```

### Add the GPG Key

Download and add the CloudStack public key to your system's trusted keys. This allows apt to verify that the packages are authentic.

```
wget -O - https://download.cloudstack.org/release.asc | sudo tee /etc/apt/trusted.gpg.d/cloudstack.asc
```

### Install Management Server and MySQL

Update your local package cache and install the cloudstack-management and mysql-server packages.

```
sudo apt update
sudo apt install -y cloudstack-management mysql-server
```

### Configure MySQL

Edit the MySQL configuration file to include settings required by CloudStack.

```
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Add the following lines under the [mysqld] section. These settings are crucial for database replication and connection handling.

```
server-id=1
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=350
log-bin=mysql-bin
binlog-format = 'ROW'
```

Restart and check the status of the MySQL service to apply the changes.

```
sudo systemctl restart mysql
sudo systemctl status mysql
```

### Setup the CloudStack Database

Run the database setup script. This command initializes the CloudStack database schema.

```
sudo cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:root -i 192.168.1.107
```

- **cloud:cloud@localhost**: Sets the database user and password to cloud.
- **--deploy-as=root:root**: Specifies the root user and password for your MySQL server (assuming a default setup).
- **-i 192.168.1.107**: This is important. It specifies the IP address of the Management Server.

## NFS and System VM Template Setup

CloudStack requires NFS (Network File System) for its primary and secondary storage. We will configure the host to be its own NFS server.

### Install NFS Server

Install the necessary NFS packages.

```
sudo apt install -y nfs-kernel-server
```

### Create Storage Directories

Create the directories that will be used for primary and secondary storage.

```
sudo mkdir -p /export/primary /export/secondary
```

- **/export/primary**: Will store the running virtual machine disk files.
- **/export/secondary**: Will store templates, ISO images, and snapshots.

### Configure NFS Exports

Edit the /etc/exports file to share these directories via NFS.

```
sudo nano /etc/exports
```

Add the following line to the file. This makes the /export directory and its subdirectories accessible to any client with read-write permissions.

```
/export *(rw,async,no_root_squash,no_subtree_check)
```

Apply the new export rules.

```
sudo exportfs -a
```

### Download System VM Template

This command downloads the default System VM template (which includes virtual routers, console proxies, etc.) and installs it into the secondary storage directory.

```
sudo /usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt -m /export/secondary -u http://download.cloudstack.org/systemvm/4.20/systemvmtemplate-4.20.0-x86_64-kvm.qcow2.bz2 -h kvm -F
```

- **-m /export/secondary**: Specifies the mount point for secondary storage.
- **-u ...**: The URL of the KVM System VM template.
- **-h kvm**: Specifies the hypervisor type.
- **-F**: Overwrites any existing template.

## KVM Host Configuration

Now, configure the same machine to act as a KVM hypervisor that the CloudStack Management Server will manage.

### Install KVM and CloudStack Agent

```
sudo apt install -y qemu-kvm cloudstack-agent
```

- **qemu-kvm**: The core KVM hypervisor package.
- **cloudstack-agent**: The agent that allows the Management Server to communicate with and manage the hypervisor.

### Configure Libvirt

Modify the libvirt configuration to allow unsecured TCP connections, which CloudStack uses for management.

First, edit /etc/libvirt/qemu.conf to enable VNC listening on all interfaces.

```
sudo sed -i -e 's/#vnc_listen = "0.0.0.0"/vnc_listen = "0.0.0.0"/' /etc/libvirt/qemu.conf
```

Next, edit /etc/libvirt/libvirtd.conf.

```
sudo nano /etc/libvirt/libvirtd.conf
```

Ensure the following lines are present and uncommented:

```
listen_tls = 0
listen_tcp = 1
tcp_port = "16509"
auth_tcp = "none"
mdns_adv = 0
```

Now, modify the libvirtd service defaults file.

```
sudo nano /etc/default/libvirtd
```

Ensure the libvirtd daemon starts with the --listen flag.

```
LIBVIRTD_ARGS="--listen"
```

Finally, restart the libvirtd service to apply all changes.

```
sudo systemctl restart libvirtd
```

### Disable AppArmor

AppArmor can sometimes interfere with libvirt's permissions. To avoid issues, disable the default libvirt profiles.

```
sudo ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
sudo ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
sudo apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

## Finalize and Launch CloudStack

With all components configured, you can now set up and start the Management Server.

### Setup and Start Management Server

Run the final setup script. This will configure the web server and start the cloudstack-management service.

```
sudo cloudstack-setup-management
```

After the script finishes, you can check the status to ensure it's running correctly. It may take several minutes for the web UI to become fully available.

```
sudo systemctl status cloudstack-management
```

### Access the CloudStack Dashboard

Open a web browser and navigate to the following URL, using your server's IP address: http://192.168.1.107:8080/client

The default login credentials are:
- **Username**: admin
- **Password**: password

You will be prompted to change the password on your first login.

## CloudStack UI Configuration

The final step is to configure the CloudStack environment through the UI by creating a Zone, Pod, Cluster, and adding the Host and Storage.

### Launch Guided Tour

After your first login to a fresh CloudStack installation, the UI prompts you to start a "Guided Tour."

- This is a step-by-step wizard designed to ensure you create all the necessary components of a basic cloud in the correct order. While you can add Zones, Pods, and other resources manually from different sections of the UI, this tour streamlines the initial setup, preventing you from missing a critical step.
- It simplifies the complex process of initial configuration. For a first-time user, it's the recommended path to get a functional environment up and running quickly.

### Zone

![picture 24](https://i.imgur.com/nluyOJ0.png) 
![picture 25](https://i.imgur.com/R7n20XP.png) 

A Zone is the highest-level organizational unit within a CloudStack deployment. Architecturally, a Zone corresponds to a single physical datacenter. It represents a distinct geographical location containing a collection of pods, clusters, hosts, and primary and secondary storage resources. 

A Zone is the boundary for certain services and resources, such as secondary storage and the availability of guest VM templates and ISOs.

CloudStack offers two distinct types of Zones, each with a different networking model: Advanced and Core. The choice of Zone type is fundamental as it dictates the networking capabilities and topology of the cloud environment.

**Core**

A Core Zone is the standard, full-featured datacenter deployment model in CloudStack.

- It is designed to be a primary, self-sufficient datacenter that provides the complete range of CloudStack's networking and infrastructure services.

- Core zones are intended to host the full cloud environment, including the management plane (in the first Core Zone), secondary storage, and system VMs, alongside the guest instances.

**Edge**

An Edge Zone is a specialized, lightweight deployment model designed for edge computing scenarios.

- The primary goal of an Edge Zone is to extend compute capabilities to remote geographical locations, closer to end-users, in order to reduce latency.

- They are intentionally limited in functionality. An Edge Zone focuses on providing compute resources (running instances) and relies on a central Core Zone for management, orchestration, and templates/ISOs (secondary storage).

**Name (GROUP7-ZONE)**

A human-readable name for your datacenter. Naming conventions are important for organization, especially in larger deployments with multiple zones.

![picture 26](https://i.imgur.com/LqBQUup.png)  
![picture 27](https://i.imgur.com/IJY5Voj.png)  
![picture 28](https://i.imgur.com/PeMarCa.png) 

**IPv4 DNS 1 (8.8.8.8)**

This is the public DNS resolver that will be provided to your guest Virtual Machines (VMs). When a VM needs to look up an internet address like www.google.com, it will use this server. 8.8.8.8 is Google's public DNS.

**Internal DNS 1 (192.168.1.1)**

This is the DNS resolver used by CloudStack's own System VMs (like the Virtual Router and Console Proxy). These system VMs need to resolve internal addresses as well as potentially public ones. Often, this is set to your local network's gateway or a dedicated internal DNS server.

**Hypervisor (KVM)**

This specifies the underlying virtualization technology that the hosts in this Zone will use. KVM (Kernel-based Virtual Machine) is a popular, open-source hypervisor for Linux. All hosts within a single Cluster must use the same hypervisor.

**Public Traffic**

This defines the public IP address range for the Zone. These are the IPs that can be assigned to VMs to make them accessible from the internet or your wider corporate network.

- **Gateway (192.168.1.1) & Netmask (255.255.255.0):** These are the fundamental network settings for your public IP range. The Gateway is the router that connects this network to the outside world. The Netmask defines the size of this network segment.

- **Start IP / End IP (192.168.1.200 - 192.168.1.220):** This is the specific pool of public IP addresses that you are handing over to CloudStack to manage. CloudStack will automatically assign IPs from this range to VMs when requested. Crucially, this range must be free and not used by any other device on your physical network.

### Pod

![picture 29](https://i.imgur.com/Hm7S6Ly.png)  
![picture 30](https://i.imgur.com/Tl5HlTA.png)  

A Pod is a logical grouping of resources within a Zone, typically representing a single rack or a row of racks. Resources within a pod are on the same private subnet.

- **Pod Name (GROUP7_POD):** A human-readable name for the rack/group of hosts.

- **Reserved System Gateway/Netmask/IP Range:** This configures a private IP network exclusively for CloudStack's System VMs.

  - CloudStack uses its own lightweight VMs to provide services. The most important is the Virtual Router, which handles networking (DHCP, DNS, Firewall, NAT) for guest VMs. Others include the Console Proxy VM (for web console access) and the Secondary Storage VM.

  - This range (192.168.1.51 - 192.168.1.80) is carved out for these critical system components to communicate with each other and the management server, keeping their traffic separate from your guest VM traffic. This is a management network.

- **Guest Traffic:** This defines the network for the actual user VMs you will create. In a simple setup, leaving this as the default is fine. In a more complex environment, you would define VLAN ranges here to isolate traffic between different tenants or applications (e.g., VLANs 100-200 for Tenant A, VLANs 201-300 for Tenant B).

### Cluster

![picture 31](https://i.imgur.com/ZVPzobp.png)  

A Cluster is a group of one or more homogenous hypervisor hosts. They are typically connected to the same high-speed switch and share the same primary storage.

- **Cluster Name (GROUP7_CLUSTER):** A human-readable name for the cluster of hosts.

- Grouping hosts into a cluster enables advanced features like High Availability (HA) for VMs. If one host fails, CloudStack can automatically restart its VMs on another host within the same cluster. It is also the boundary for VM live migration.

### Host

![picture 32](https://i.imgur.com/GcoGr1a.png)  

This is where you add your actual physical compute server to the cluster.

- **Hostname (192.168.1.107):** The IP address of the physical server running the KVM hypervisor. CloudStack will use this address to communicate with and manage the host.
- **Username (root) & Password:** The credentials CloudStack will use to SSH into the host. It needs this access to perform actions like starting/stopping VMs, configuring networking, and managing storage.

### Primary Storage

![picture 33](https://i.imgur.com/LCErt8i.png)  

Primary Storage is where the virtual disks of running VMs are stored. It needs to be fast and have low latency because it's actively being used by the VMs.

- **Name (GROUP7_PRIMARY):** A human-readable name for this storage device.
- **Protocol (NFS):** Specifies the type of storage. NFS (Network File System) is a common choice where you share a directory from a server over the network. CloudStack tells all hosts in the cluster to "mount" this shared directory.
- **Server (192.168.1.107):** The IP address of the server providing the NFS share. In this single-node setup, the host itself is also the storage server.
- **Path (/export/primary):** The specific directory path on the server that is being shared for this purpose. This is where CloudStack will create the VDI (Virtual Disk Image) files for your VMs.

### Secondary Storage

![picture 34](https://i.imgur.com/Fn7PiO2.png)  

Secondary Storage acts as a library for your Zone. It is used to store templates, ISO images, and disk snapshots. It does not require the same high performance as primary storage.

- **Provider (NFS):** The storage protocol, same as the primary storage in this example.
Name (GROUP7_SECONDARY): A friendly name.
- **Server (192.168.1.107):** The IP of the NFS server.
- **Path (/export/secondary):** The shared directory path for storing templates and ISOs. When you create a new VM from a template, CloudStack copies the template from this secondary storage to the primary storage and then starts the VM.

### Launch Zone

![picture 35](https://i.imgur.com/JZfYH82.png)  

This is the final confirmation step.

- When you click "Launch Zone," CloudStack takes all the information you provided and begins executing a workflow to provision the Zone. It will:
  1. Add the Zone, Pod, and Cluster records to its database.
  2. Attempt to connect to the Host (192.168.1.107) using the provided credentials.
  3. Configure networking on the host.
  4. Instruct the host to mount the Primary Storage (/export/primary).
  5. Set up the Secondary Storage for the Zone.
  6. Deploy the necessary System VMs (like the Secondary Storage VM).

- This process takes several minutes. The CloudStack UI will show the progress as it works through these steps. If any step fails (e.g., it can't connect to the host or mount the storage), it will report an error, and you will need to troubleshoot the issue.

## Creating a Compute Offering

![picture 37](https://i.imgur.com/dzZt0RZ.png)  

1. Navigate to the Service Offerings section in the left-hand navigation menu.
2. Ensure the Compute Offerings tab is selected.
3. Click the Add Compute Offering button.
4. Complete the configuration form with the following parameters:
    - Name: A unique, human-readable identifier for the offering (e.g., Small-Instance, c1.medium).
    - Description: A clear description of the offering's intended use or specifications.
    - Storage Type: Specifies the type of primary storage for the instance's root disk.
      - shared: The root disk will be provisioned on shared primary storage (e.g., NFS, Ceph RBD). This is the standard choice and is required for features like live migration.
      - local: The root disk will be provisioned on the local disk of the physical host. This can offer higher performance but does not support live migration.
5. CPU Number: The quantity of virtual CPU cores to allocate to the instance.
6. CPU Speed (in MHz): The guaranteed clock speed for each virtual CPU core.
7. Memory (in MB): The amount of RAM to allocate to the instance.
8. Offer HA: If checked, instances created with this offering will be highly available. If the host they are running on fails, CloudStack will attempt to restart the instance on another host within the same cluster. This requires the use of shared storage.
9. Storage Tags (Optional): An advanced feature that allows binding an offering to a specific primary storage pool identified by a tag (e.g., SSD, HDD_Tier1). This ensures instances are created on the appropriate storage backend.

## Registering an ISO

![picture 36](https://i.imgur.com/cqY8Lec.png)  

1. Navigate to the Images section in the left-hand navigation menu.
2. Select the ISOs tab.
3. Click the Register ISO button.
4. Complete the registration form:
    - Name: A descriptive name for the ISO (e.g., Ubuntu-24.04-Server)
    - Description: Additional details about the ISO file.
    - URL: The direct, publicly accessible URL from which the ISO file can be downloaded. The CloudStack Secondary Storage VM will use this URL to fetch the file.
    - Zone: Select the Zone in which this ISO will be made available. It will be downloaded to this Zone's secondary storage.
    - Bootable: Check this box to indicate that the ISO contains a bootable operating system installer.
    - OS Type: Select the guest operating system family and version from the dropdown list (e.g., Ubuntu Linux (64-bit)). This helps CloudStack optimize virtual hardware settings for the instance.
5. Click OK to begin the registration process.

## Creating an Instance

![picture 41](https://i.imgur.com/Xf07Qjh.png)  
![picture 42](https://i.imgur.com/g5nfbzo.png)  
![picture 43](https://i.imgur.com/ofeWqBl.png)  

1. Navigate to the Instances section in the left-hand navigation menu.
2. Click the Add Instance button. This will launch a multi-step wizard.
3. Define Instance Ownership
    - For Domain: Select the domain that the owner account belongs to from the dropdown list.
    - For Account: Choose the specific user account that will own this instance.
4. Select the Deployment Infrastructure
    - Click on the Zone field and select the datacenter (Zone) where you want your instance to be deployed.
    - Optionally, you can specify a Pod, Cluster, or even a specific Host for the deployment. If you only select the Zone, CloudStack will automatically place the instance on the most suitable hardware to balance resources.
5. Choose a Template or ISO
    - Click on either the Templates or ISOs tab.
    - Select the desired template or ISO from the list. If you registered an ISO earlier, it will appear here.
6. Choose a Compute Offering
    - Review the list of available compute offerings. Each offering represents a different size with a specific vCPU count, CPU speed, and amount of memory.
    - Select the compute offering that best fits the performance needs of your application.
7. Choose a Disk Offering
    - Select a disk offering from the provided list. These are predefined storage sizes (e.g., Small, Medium, Large).
    - Alternatively, if available, select the Custom option and enter the desired disk size in Gigabytes (GB).
8. Configure Networks
    - Check the box next to one or more networks from the list to attach them to your instance.
9. Launch the Instance
    - Review all your selections on the final confirmation screen.
    - Click the Launch Instance button to create the VM.

## Allowing Your VM to Access the Internet (Egress Rules)

![picture 38](https://i.imgur.com/AyfuVe8.png)  

To allow your virtual machine to reach outside resources like websites or software repositories (e.g., to run apt update), you need to configure Egress Rules. Egress rules control all outgoing traffic originating from within your network.

1. Navigate to the Network section in the left-hand navigation menu.
2. Click on the Egress Rules tab.
3. Understand the Default Policy: Notice the message: "The default egress policy of this Network is Deny." This is a security-first approach, meaning that by default, no traffic is allowed to leave your network. You must explicitly create rules to allow it.
4. Create an "Allow All" Rule: To grant your VM general internet access, you need to create a rule that allows traffic from your network to any destination.
    - **Source CIDR:** Enter the internal network range of your VMs (e.g., 10.1.1.0/24 as shown in your existing rule). This specifies that the rule applies to traffic coming from your VMs.
    - **Destination CIDR:** Enter 0.0.0.0/0. This is a special address that means "any destination" or "the entire internet."
    - **Protocol:** Select All. This allows all types of traffic (TCP, UDP, ICMP, etc.) to leave your network.
    - Click Add.

## Allowing SSH Access to Your VM (Firewall & Port Forwarding)

To connect to your VM from your computer using an SSH client, you need to configure rules for incoming traffic. This is a two-step process involving the Firewall and Port Forwarding.

### Open the Public Port (Firewall)

![picture 39](https://i.imgur.com/QyWglff.png)  

The Firewall controls which ports are open to the world on your network's public IP address.

1. Navigate to the Network section in the left-hand navigation menu.
2. Click on the Firewall tab.
3. Click Add to create a new rule to allow incoming SSH connections.
    - **Source CIDR:** Enter 0.0.0.0/0. This will allow you to connect from any computer on the internet.
    - **Protocol:** Select TCP.
    - **Start port / End port:** Enter the public port you want to open. For standard SSH, this is port 22. However, you can use a different port for added security (e.g., 2224 as seen in the screenshot's port forwarding rule). 
    - Click Add.

### Port Forwarding

![picture 40](https://i.imgur.com/rgE3Sse.png) 

The firewall rule opens the door on your public IP, but it doesn't know which specific VM to send the traffic to. Port Forwarding creates that connection.

1. Navigate to the Network section in the left-hand navigation menu.
2. Click on the Port Forwarding tab.
3. Click Add Instance. A dialog will appear for you to select your VM.
4. Configure the port mapping:
- **Private port:** Enter the port the service is listening on inside your VM. For SSH, this is almost always 22.
- **Public port:** Enter the public port you opened in the firewall step. To match our firewall example, this would be 2224.
- **Protocol:** Select TCP.
- Click Add.
