---
cover:
  image: /media/images/openstack/openstack-ansible/openstack-ansible-logo.png
  alt: "Openstack-Ansible"

date: '2025-02-18T06:33:46Z'
title: 'Deploying OpenStack with Openstack-Ansible'
tags: ["OpenStack", "Open Source"]
category: ["OpenStack"]
ShowToc: true
---

Saat ini, pengelolaan banyak server secara manual sangat tidak efisien dan memerlukan waktu yang lama. Jika salah satu server mengalami gangguan, sistem dapat terganggu, dan proses penambahan server baru juga cukup rumit. Solusi yang dapat diterapkan adalah menggunakan **OpenStack**, yang menggabungkan seluruh server menjadi satu sistem cloud agar lebih fleksibel dan mudah dikelola. Untuk memastikan proses ini berjalan otomatis dan efisien, digunakan **OpenStack-Ansible**, yang memungkinkan konfigurasi sistem dilakukan tanpa perlu pengaturan manual satu per satu. Dengan solusi ini, infrastruktur menjadi lebih stabil, mudah dikembangkan, dan lebih andal dalam menghadapi gangguan.

# Teori
## Apa itu Openstack?
Openstack adalah platform komputasi awan (cloud computing) open-source yang memungkinkan kita untuk membangun dan mengelola infrastruktur cloud publik maupun private. Bayangkan OpenStack sebagai sistem operasi untuk seluruh pusat data. Dengan OpenStack, kita dapat mengontrol dan mengelola sumber daya komputasi, penyimpanan, dan jaringan melalui antarmuka yang terstandarisasi. OpenStack sendiri memiliki  banyak service, yang memiliki fungsi berbeda-beda. Untuk Informasi lebih lanjut mengenai service OpenStack, bisa baca [disini](https://docs.redhat.com/en/documentation/red_hat_openstack_platform/12/html/architecture_guide/components).

## Apa itu Ansible?
Ansible merupakan alat otomatisasi yang digunakan untuk mengelola konfigurasi server dan aplikasi. Dengan Ansible, kita bisa menjalankan perintah pada banyak server sekaligus, tanpa perlu mengaturnya satu per satu. Ansible menggunakan file YAML untuk mendefinisikan tugas-tugas yang harus dilakukan yang disebut dengan playbook.

## Apa itu Openstack-Ansible?
Openstack-Ansible adalah sebuah proyek yang menggabungkan OpenStack dan Ansible untuk mempermudah instalasi serta manajemen OpenStack secara otomatis. Pengguna dapat menginstal dan mengelola berbagai komponen OpenStack menggunakan Playbook Ansible, yang membuat deployment lebih konsisten dan dapat diskalakan. Dalam implementasinya, OpenStack dapat dijalankan dalam container untuk meningkatkan isolasi dan manajemen sumber daya.

# Tools
* Ubuntu – Jammy – v22.04
*	OpenStack – Caracal – v2024.1
*	OpenStack-Ansible – v2024.1
*	Python – v3.10.12
*	Ansible - v2.15.9

# Topologi
## Jaringan
![Network Topology](/media/images/openstack/openstack-ansible/openstack-ansible-network.svg)

## Logikal
![Logical Topology](/media/images/openstack/openstack-ansible/openstack-ansible-logical.svg)

# Langkah Implementasi
## 1. Konfigurasi Port Bridge
Note : Lakukan langkah dibawah ini di semua node.
* Menambahkan interface bridge ”br-mgmt” yang nantinya akan dipakai sebagai management interface untuk container.
```
~$ sudo nano /etc/netplan/50-cloud-init.yaml
```
```
network:
    version: 2
    ethernets:
        ens3:
            dhcp4: false
        ens4:
            dhcp4: false
    bridges:
        br-mgmt:
            interfaces: [ens3]
            addresses: [<NodeIP>/24]
            routes:
            - to: default
              via: 192.168.4.1
            nameservers:
              addresses: [8.8.8.8]
            parameters:
                stp: false
                forward-delay: 0
```

* Terapkan konfigurasi yang sudah kita definisikan
```
~$ sudo netplan apply
```

## 2. Konfigurasi Akses Remote menggunakan SSH
Note : Lakukan langkah dibawah ini di semua node.
* Ubah hostname pada semua node.
```
Controller Node : ~$ sudo hostnamectl set-hostname pod-controller
Compute1 Node   : ~$ sudo hostnamectl set-hostname pod-compute1
Compute2 Node   : ~$ sudo hostnamectl set-hostname pod-compute2
```

* Tambahkan name resolver secara lokal, sehingga sistem bisa mengenali hostname tanpa bergantung pada DNS eksternal.
```
~$ sudo nano /etc/hosts
```
```
# Add this line below
192.168.4.10 pod-controller
192.168.4.20 pod-compute1
192.168.4.30 pod-compute2
```

* Membuat SSH Key agar bisa melakukan akses ke Node lain secara passwordless. 
```
~$ ssh-keygen -t rsa

# Copying public key to other Node
~$ ssh-copy-id pod-controller
~$ ssh-copy-id pod-compute1
~$ ssh-copy-id pod-compute2
```

* Mengubah password dari user root.
```
~$ sudo -i
~# passwd root
```

* Membuat SSH Key untuk user root. <br>
Note : Lakukan langkah dibawah ini di controller node.
```
~# ssh-keygen -t rsa

# Copying public key to other Node
~# ssh-copy-id pod-controller
~# ssh-copy-id pod-compute1
~# ssh-copy-id pod-compute2
```

## 3. Deploy OpenStack dengan OpenStack-Ansible
Note : Lakukan langkah dibawah ini di semua node.
* Memperbarui paket dan install depedensi yang dibutuhkan.
```
# Updating packages
~$ sudo apt update -y

# Installing dependecies
~$ sudo apt install build-essential git chrony openssh-server python3-dev sudo bridge-utils debootstrap openssh-server tcpdump vlan python3 -y
```

* Membuat physical volume dan volume group untuk penyimpan data-data dari service Cinder.
```
~# pvcreate /dev/vdb
~# vgcreate cinder-volumes /dev/vdb
```
<br>
Note : Lakukan langkah dibawah ini di controller node.

* Membuat file untuk swap.
```
~$ sudo fallocate -l 8G /swapfile
~$ sudo chmod 600 /swapfile
~$ sudo mkswap / swapfile
~$ sudo swapon /swapfile
~$ sudo vim /etc/fstab
```
```
# Add this line below
/swapfile swap swap defaults 0 0
```

*	Download paket OpenStack-Ansible.
```
# Change to root user
~$ sudo -i

# Clone openstack-ansible package
~# git clone -b stable/2024.1 https://opendev.org/openstack/openstack-ansible /opt/openstack-ansible
```

* Jalankan skrip bootstrap agar ansible dan paket-paket terkait terinstall.
```
~# cd /opt/openstack-ansible
/opt/openstack-ansible# scripts/bootstrap-ansible.sh
```

*	Menyalin template konfigurasi ke direktori **“/etc/openstack_deploy”**. Dan konfigurasi file utama, yaitu **“openstack_user_config.yml”**.
```
~# cp -avrp /opt/openstack-ansible/etc/openstack_deploy /etc/.
~# cd /etc/openstack_deploy/
/etc/openstack_deploy# nano openstack_user_config.yml
```
```
---
cidr_networks:
  container: 192.168.4.0/24

used_ips:
  - "192.168.4.0,192.168.4.10,192.168.4.20,192.168.4.30"

global_overrides:
  internal_lb_vip_address: 192.168.4.10
  external_lb_vip_address: 192.168.4.10
  management_bridge: "br-mgmt"

  provider_networks:
    - network:
        group_binds:
          - all_containers
          - hosts
        type: "raw"
        container_bridge: "br-mgmt"
        container_interface: "eth1"
        container_type: "veth"
        ip_from_q: "container"
        is_container_address: true
        is_ssh_address: true
        is_management_address: true

controller: &controllers
  pod-controller:
    ip: 192.168.4.10

compute: &computes
  pod-compute1:
    ip: 192.168.4.20
  pod-compute2:
    ip: 192.168.4.30

shared-infra_hosts: *controllers
repo-infra_hosts: *controllers
image_hosts: *controllers
orchestration_hosts: *controllers
placement-infra_hosts: *controllers
identity_hosts: *controllers
network_hosts: *controllers
compute-infra_hosts: *controllers
compute_hosts: *computes
haproxy_hosts: *controllers
storage-infra_hosts: *controllers
storage_hosts:
 pod-controller:
   ip: 192.168.4.10
   container_vars:
     cinder_storage_availability_zone: cinderAZ_1
     cinder_default_availability_zone: cinderAZ_1
     cinder_backends:
       lvm:
         volume_backend_name: LVM_iSCSI
         volume_driver: cinder.volume.drivers.lvm.LVMVolumeDriver
         volume_group: cinder-volumes
         iscsi_ip_address: "{{ cinder_storage_address }}"
       limit_container_types: cinder_volume

 pod-compute1:
   ip: 192.168.4.20
   container_vars:
     cinder_storage_availability_zone: cinderAZ_1
     cinder_default_availability_zone: cinderAZ_1
     cinder_backends:
       lvm:
         volume_backend_name: LVM_iSCSI
         volume_driver: cinder.volume.drivers.lvm.LVMVolumeDriver
         volume_group: cinder-volumes
         iscsi_ip_address: "{{ cinder_storage_address }}"
       limit_container_types: cinder_volume

 pod-compute2:
   ip: 192.168.4.30
   container_vars:
     cinder_storage_availability_zone: cinderAZ_1
     cinder_default_availability_zone: cinderAZ_1
     cinder_backends:
       lvm:
         volume_backend_name: LVM_iSCSI
         volume_driver: cinder.volume.drivers.lvm.LVMVolumeDriver
         volume_group: cinder-volumes
         iscsi_ip_address: "{{ cinder_storage_address }}"
       limit_container_types: cinder_volume
```

* Konfigurasi file **“user_variables.yml”** untuk menentukan variabel kustom yang akan diterapkan dalam deployment OpenStack-Ansible. 
```
/etc/openstack_deploy# nano user_variables.yml
```
```
apply_security_hardening: false

# SSL Configs
haproxy_ssl: false
horizon_enable_ssl: false
openstack_external_ssl: false

# Neutron Configs
neutron_plugin_type: "ml2.ovs"
neutron_provider_networks:
  network_flat_networks: "provider"
  network_mappings: "provider:br-ex"
  network_interface_mappings: "br-ex:ens4"
  network_types: "vxlan"
  network_vxlan_ranges: "1:1000"
neutron_ml2_drivers_type: "flat, vlan, vxlan"
neutron_plugin_base:
  - router
neutron_l2_population: True
neutron_ml2_mechanism_drivers: "openvswitch, l2population"
neutron_firewall_driver: "iptables_hybrid"

# Galera Configs
galera_cluster_name: openstack_galera_cluster
galera_max_connections: 3200

# Nova Configs
openstack_service_publicuri_proto: "http"
nova_console_type: novnc

# Heat Configs
heat_wsgi_processes: 1
heat_api_threads: 1
heat_engine_workers: 1

# RabbitMQ Configs
rabbitmq_use_ssl: False
rabbitmq_queue_replication: False
oslomsg_rabbit_quorum_queues: False
rabbitmq_monitoring_userid: monitoring
rabbitmq_upgrade: false
rabbitmq_erlang_extra_args: "+sbwt none +sbwtdcpu none +sbwtdio none +stbt nnts"
```

* Menjalankan skrip yang akan membuat password dari service dan komponen OpenStack.
```
~# cd /opt/openstack-ansible
/opt/openstack-ansible# ./scripts/pw-token-gen.py --file /etc/openstack_deploy/user_secrets.yml
```

* Menjalankan playbook agar service dan komponen dari OpenStack terinstall.
```
~# cd /opt/openstack-ansible/playbooks/
/opt/openstack-ansible/playbooks# openstack-ansible setup-hosts.yml
/opt/openstack-ansible/playbooks# openstack-ansible setup-infrastructure.yml
/opt/openstack-ansible/playbooks# openstack-ansible setup-openstack.yml
```

# Pengujian
* Akses container utility, dan menggunakan identitas OpenStack dengan file **“openrc”**. 
```
~# lxc-ls | grep "utility"
pod-controller-utility-container-ebab4dfe
~# lxc-attach pod-controller-utility-container-ebab4dfe
utility:/# source /root/openrc
```

* Install wget untuk download image cirros. Lalu menambahkan image Cirros ke OpenStack. 
```
utility:/# apt install wget -y
utility:/# wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
utility:/# openstack image create --disk-format qcow2 --container-format bare --public --file ./cirros-0.62-x86_64-disk.img cirros-0.6.2-rifky
```

* Menambahkan jaringan eksternal untuk OpenStack.
```
utility:/# openstack network create --share --external --provider-physical-network provider --provider-network-type flat external-net-rifky
utility:/# openstack subnet create --network external-net-rifky --gateway 20.6.6.1 --no-dhcp --subnet-range 20.6.6.0/24 external-subnet-rifky
```

* Menambahkan jaringan internal untuk OpenStack.
```
utility:/# openstack network create internal-net-rifky
utility:/# openstack subnet create --network internal-net-rifky --allocation-pool start=192.168.6.10, end=192.168.6.254 --dns-nameserver 8.8.8.8 --gateway 192.168.6.1 --subnet-range 192.168.6.0/24 internal-subnet-rifky
```

* Menambahkan router pada OpenStack, yang akan menghubungkan jaringan internal dan eksternal. 
```
utility:/# openstack router create router-rifky
utility:/# openstack router set -- external-gateway external-net-rifky router-rifky
utility:/# openstack router add subnet router-rifky internal-subnet-rifky
```

* Membuat security group yang didefinisikan agar bisa diakses secara SSH, dan bisa di ping. 
```
utility:/# openstack security group create security-group-rifky --description 'Allow SSH and ICMP'
utility:/# openstack security group rule create --protocol icmp security-group-rifky
utility:/# openstack security group rule create --protocol tcp --ingress --dst-port 22 security-group-rifky
```

* Membuat keypair yang akan digunakan untuk mengakses instance tanpa menggunakan password. 
```
utility:/# exit
~# cat /root/.ssh/id_rsa.pub
ssh-rsa xxxxx root@pod-controller
~# lxc-attach pod-controller-utility-container-ebab4dfe
utility:/# echo "ssh-rsa xxxxx root@pod-controller" > id_rsa.pub
utility:/# source /root/openrc
utility:/# openstack keypair create --public-key id_rsa.pub controller-key
```

* Membuat flavor untuk mendefinisikan spesifikasi yang bisa dipakai oleh instance. 
```
utility:/# openstack flavor create --ram 512 --disk 5 --vcpus 1 --public c2-small
```

* Membuat instance dengan resource yang sudah kita buat sebelumnya. Dan menambahkan floating IP, agar bisa diakses dari jaringan eksternal.
```
utility:/# openstack server create --flavor c2-small --image cirros-0.6.2-rifky --key-name controller-key --security-group security-group-rifky --network internal-net-rifky cirros-instance-rifky
utility:/# openstack floating ip create --floating-ip-address 20.6.6.200 external-net-rifky
utility:/# openstack server add floating ip cirros-instance-rifky 20.6.6.200
```

*	Pengujian akses internet dari instance Cirros.
```
$ cat /etc/os-release
PRETTY_NAME="CirrOS 0.6.2"
NAME="CirrOS"
VERSION_ID="0.6.2"
ID=cirros
HOME_URL="https://cirros-cloud.net"
BUG_REPORT_URL="https://github.com/cirros-dev/cirros/issues"

$ ping google.com
PING 216.58.199.238 (216.58.199.238) 56(84) bytes of data.
64 bytes from kix05s02-in-f238.1e100.net (216.58.199.238): icmp_seq=1 ttl=119
time=1.69 ms
64 bytes from kul09s15-in-f14.le100.net (216.58.199.238): icmp_seq=2 ttl=119
time=2.31 ms
--- 216.58.199.238 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.687/1.997/2.308/0.310 ms
```

* Pengujian ping dan akses SSH ke arah floating IP dari instance Cirros. 

```
controller:/# ping 20.6.6.200
PING 20.6.6.200 (20.6.6.200) 56(84) bytes of data.
64 bytes from 20.6.6.200: icmp_seq=1 ttl=62 time=3.53 ms
64 bytes from 20.6.6.200: icmp_seq=2 ttl=62 time=1.97 ms

--- 20.6.6.200 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.965/2.745/3.526/0.780 ms

controller:/# ssh -o 'PubkeyAcceptedKeyTypes +ssh-rsa' cirros@20.6.6.200
$ cat /etc/os-release
PRETTY_NAME="CirrOS 0.6.2"
NAME="CirrOS"
VERSION_ID="0.6.2"
ID=cirros
HOME_URL="https://cirros-cloud.net"
BUG_REPORT_URL="https://github.com/cirros-dev/cirros/issues"
```

# Referensi
* Penjelasan mengenai Cloud Computing.
 	* https://btech.id/id/news/what-is-cloud-computing/
* Penjelasan mengenai OpenStack.
 	* https://btech.id/en/news/mengenal-openstack-serta-fitur-dan-kelebihannya/
*	Deploy Openstack menggunakan Openstack-Ansible.
 	* https://docs.openstack.org/project-deploy-guide/openstack-ansible/2024.1/
*	Konfigurasi Openstack-Ansible.
 	* https://docs.openstack.org/openstack-ansible/latest/reference/inventory/openstack-user-config-reference.html
 	* https://satishdotpatel.github.io/build-openstack-cloud-using-openstack-ansible/
 	* https://satishdotpatel.github.io/openstack-ansible-multinode-ovn/
 	* https://medium.com/@travistruman/configuring-openstack-ansible-for-open-vswitch-b7e70e26009d
