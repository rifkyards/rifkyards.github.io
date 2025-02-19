---
cover:
  image: /media/images/openstack/horizon-with-yuyu/horizon-with-yuyu-banner.svg
  alt: "Horizon with Yuyu Banner"

date: '2025-02-19T03:33:46Z'
title: 'Using Horizon and Yuyu to Track Your OpenStack Resource'
tags: ["OpenStack", "Open Source", "Add-ons"]
category: ["OpenStack"]
ShowToc: true
---
Pada blog sebelumnya tentang [Deploying Openstack with OpenStack-Ansible](https://rifkyards.github.io/blogs/openstack/openstack-ansible/), telah dibahas bagaimana OpenStack memungkinkan pengelolaan banyak server secara otomatis dan efisien, sehingga menciptakan infrastruktur cloud yang lebih stabil dan mudah dikembangkan.

Namun, untuk pengelolaan biaya dan tagihan terkait penggunaan sumber daya OpenStack, dibutuhkan sebuah solusi tambahan yang memudahkan pemantauan dan perhitungan biaya. **Yuyu Billing** hadir sebagai plug-in untuk OpenStack yang memungkinkan pengelolaan tagihan secara otomatis dan efisien. Dengan **Yuyu**, Anda dapat menghitung biaya untuk berbagai fitur OpenStack, seperti instance flavors, volumes, floating IPs, routers, snapshots, dan images, sehingga memudahkan pengelolaan anggaran dan memastikan transparansi biaya dalam lingkungan cloud.

# Teori
## Apa itu Horizon?
Horizon adalah dashboard berbasis web untuk OpenStack yang memungkinkan pengguna mengelola layanan OpenStack melalui antarmuka grafis. Dengan Horizon, pengguna dapat mengelola instance, jaringan, penyimpanan, dan sumber daya lainnya tanpa perlu menggunakan CLI atau API secara langsung.

## Apa itu Yuyu Billing?
Yuyu merupakan plug-in untuk OpenStack yang memudahkan pengelolaan tagihan dengan memungkinkan pengguna untuk menghitung biaya berbagai fitur OpenStack, seperti Instance Flavors, Volumes, Floating IPs, Routers, Snapshots, dan Images. Plugin ini membantu pengguna mengelola biaya terkait penggunaan sumber daya di OpenStack secara lebih efisien.

# Tools
* Ubuntu – Jammy – v22.04
*	OpenStack – Caracal – v2024.1
*	OpenStack-Ansible – v2024.1
*	Python – v3.10.12
*	Ansible - v2.15.9
*	Horizon – v23.4.0
*	Heat Dashboard – v11.0.0
*	Yuyu API & Event Monitor – Last updated on Oct 2, 2023
*	Yuyu Dashboard – Last updated on Dec 10, 2023
*	Django – v3.2
*	Python-Memcached – v1.59

# Topologi
![Topologi](/media/images/openstack/horizon-with-yuyu/horizon-with-yuyu-topology.svg)

# Langkah Implementasi
## 1. Deploy Horizon Sebagai Halaman Dashboard dari Openstack
Note : Lakukan langkah dibawah ini di controller node.
* Melakukan update paket dan instalasi paket beserta dependensi yang dibutuhkan oleh horizon. 
```
# Update package
~# apt update -y

# Install package and dependencies
~# apt install python3 python3-dev python3-venv python3-distutils apache2 libapache2-mod-wsgi-py3 libmemcached-tools python3-setuptools python3-virtualenv python3-pip -y
```

* Download file-file yang dibutuhkan oleh Horizon.
```
~# cd /var/www/html/
/var/www/html# git clone -b 23.4.0 https://opendev.org/openstack/horizon.git
```

* Melakukan instalasi depedensi dan library yang dibutuhkan oleh Horizon. 

```
/var/www/html# cd horizon
/var/www/html/horizon# pip install -U pip
/var/www/html/horizon# wget https://opendev.org/openstack/requirements/raw/branch/stable/2024.1/upper-constraints.txt

/var/www/html/horizon# sed -i 's/horizon === \([0-9.]\+\)/horizon === 23.4.0/g' upper-constraints.txt

/var/www/html/horizon# pip install -c upper-constraints.txt .
/var/www/html/horizon# pip install -r requirements.txt
/var/www/html/horizon# pip install "Django>=3.2,<3.3" --upgrade
/var/www/html/horizon# pip install --upgrade --force-reinstall python-memcached==1.59
```

* Menyalin file template konfigurasi Horizon. Dan menambahkan konfigurasi untuk Horizon.
```
/var/www/html/horizon# cp openstack_dashboard/local/local_settings.py.example openstack_dashboard/local/local_settings.py
/var/www/html/horizon# nano openstack_dashboard/local/local_settings.py
```
```
DEBUG = True

WEBROOT = "/"

ALLOWED_HOSTS = ['*']

#Yuyu Configs
YUYU_URL = "http://192.168.4.10:8182"
CURRENCIES = ('IDR',)
DEFAULT_CURRENCY = "IDR"

CACHES = {
  'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '<IP dari Memcached Container>:11211',
  },
}

OPENSTACK_KEYSTONE_URL = "http://192.168.4.10:5000/v3"

OPENSTACK_KEYSTONE_DOMAIN_CHOICES = (
  ('Default', 'default' )
)

OPENSTACK_KEYSTONE_BACKEND = {
        'name': 'native',
        'can_edit_group': True,
        'can_edit_user': True,
        'can_edit_role': True,
        'can_edit_project' : True,
        'can_edit_domain': True,
}

AVAILABLE_THEMES = [
('default', 'Default', 'themes/default'),
('material', 'Material', 'themes/material' ),
('example', 'Example', 'themes/example' )
]
```

* Untuk mendapatkan IP dari Memcached, bisa dengan perintah di bawah. Gunakan IP dari management network.
```
~# lxc-ls -f | grep "memcached"
pod-controller-memcached-container-46f8dbab      RUNNING 1         onboot, openstack 10.0.3.226, 192.168.4.183 -    false
```

* Mendownload paket halaman dahsboard untuk service Heat. Lalu menyalin konfigurasi dari paket tersebut ke Horizon. 
```
~# pip install heat-dashboard==11.0.0
~# cp -n /usr/local/lib/python3.10/dist-packages/heat_dashboard/enabled/_[1-9]*. py /var/www/html/horizon/openstack_dashboard/enabled/
```

* Menjalankan perintah untuk ntuk mengumpulkan dan memindahkan file statis (seperti CSS, JavaScript, gambar).
```
/var/www/html/horizon# ./manage.py collectstatic
```

* Menjalankan perintah agar aplikasi Django Horizon bisa dijalankan dengan WSGI (untuk menghubungkan aplikasi Python dengan server web) dan Apache sebagai server web untuk melayani aplikasi tersebut.
```
/var/www/html/horizon# ./manage.py make_web_conf --wsgi
/var/www/html/horizon# ./manage.py make_web_conf --apache > /etc/apache2/sites-available/horizon.conf
```

* Menjalankan perintah untuk mengonfigurasi Apache untuk menggunakan situs horizon.conf dan menonaktifkan situs default, serta mengubah konfigurasi HAProxy. 
```
~# a2ensite horizon.conf && a2dissite 000-default.conf
~# nano /etc/haproxy/haproxy.cfg

## Add comment for frontend that listen to port 80 ##

#frontend base-front-1
#   bind 192.168.4.10:80
#   option httplog
#   option forwardfor except 127.0.0.0/8
#   use_backend %[path, map_reg(/etc/haproxy/base_regex.map)]
#   mode http
```

* Menjalankan perintah untuk mengubah kepemilikan dan izin akses file untuk Horizon agar bisa digunakan oleh Apache.
```
/var/www/html/horizon# chown -R www-data:www-data openstack_dashboard/local/.secret_key_store
/var/www/html/horizon# chown -R www-data:www-data static/
/var/www/html/horizon# chmod a+x openstack_dashboard/wsgi.py
```

## 2. Konfigurasi Yuyu Billing API, Event Monitor dan Dashboard
Note : Lakukan langkah dibawah ini di controller node.
* Melihat nama dari container yang dibuat oleh Openstack-Ansible. 
```
~# lxc-ls
pod-controller-cinder-api-container-09654180     pod-controller-galera-container-6bcebebb
pod-controller-glance-container-b31353e2         pod-controller-heat-api-container-aec695b2
pod-controller-keystone-container-aa059c5b       pod-controller-memcached-container-46f8dbab
pod-controller-neutron-server-container-3335ac4f pod-controller-nova-api-container-57ed6068
pod-controller-placement-container-806005d7      pod-controller-rabbit-mq-container-fa6b4b03
pod-controller-repo-container-45f85cad           pod-controller-utility-container-ebab4dfe
```

* Menjalankan perintah untuk mendapatkan password dari RabbitMQ. 
```
~# cat /etc/openstack_deploy/user_secrets.yml | grep rabbitmq_monitoring_password
rabbitmq_monitoring_password: <Monitoring Password>
```
* Mendapatkan IP dari container RabbitMQ, serta konfigurasi user monitoring agar memiliki akses baca pada semua data di RabbitMQ. Disini kita akan memakai IP yang berasal dari management network.
```
~# lxc-ls -f | grep rabbit
pod-controller-rabbit-mq-container-fa6b4b03      RUNNING 1         onboot, openstack 10.0.3.137, 192.168.4.182 -    false

~# lxc-attach pod-controller-rabbit-mq-container-fa6b4b03
rabbit-mq:/# rabbitmqctl set_permissions -p / monitoring ".*" ".*" ".*"
```
* Menambahkan konfigurasi agar service Nova mengirimkan data ke RabbitMQ yang akan digunakan oleh Yuyu
```
/# lxc-attach <Nova Container>
nova:/# apt install nano
nova:/# nano /etc/nova/nova.conf

# Modify this group configs
[oslo_messaging_notifications]
driver = messagingv2
topics = notifications
transport_url = rabbit://monitoring:<Monitoring Password>@<RabbitMQ IP>:5672/

#Also add this 
[notifications]
notify_on_state_change = vm_and_task_state
notification_format = unversioned

nova:/# systemctl restart nova-scheduler
```

* Menambahkan konfigurasi agar service Cinder mengirimkan data ke RabbitMQ yang akan digunakan oleh Yuyu.
```
/# lxc-attach <Cinder Container>
cinder:/# apt install nano
cinder:/# nano /etc/cinder/cinder.conf

# Modify this group configs
[oslo_messaging_notifications]
driver = messagingv2
topics = notifications
transport_url = rabbit://monitoring:<Monitoring Password>@<RabbitMQ IP>:5672/

cinder:/# systemctl restart cinder-scheduler
```
* Menambahkan konfigurasi agar service Neutron mengirimkan data ke RabbitMQ yang akan digunakan oleh Yuyu.  
```
/# lxc-attach <Neutron Container>
neutron:/# apt install nano
neutron:/# nano /etc/neutron/neutron.conf

# Modify this group configs
[oslo_messaging_notifications]
driver = messagingv2
topics = notifications
transport_url = rabbit://monitoring:<Monitoring Password>@<RabbitMQ IP>:5672/

neutron:/# systemctl restart neutron-server
```

* Menambahkan konfigurasi agar service Keystone mengirimkan data ke RabbitMQ yang akan digunakan oleh Yuyu.  
```
/# lxc-attach <Keystone Container>
keystone:/# apt install nano
keystone:/# nano /etc/keystone/keystone. conf

# Modify this group configs
[oslo_messaging_notifications]
driver = messagingv2
topics = notifications
transport_url = rabbit://monitoring:<Monitoring Password>@<RabbitMQ IP>:5672/

keystone:/# systemctl restart keystone-wsgi-public
```

* Download file-file yang dibutuhkan oleh Yuyu, dan mendownload depedensi dan library yang dibutuhkan. 
```
~# cd /var/
~# git clone https://github.com/Yuyu-billing/yuyu.git
~# cd yuyu
/var/yuyu# virtualenv env
/var/yuyu# source env/bin/activate
(env): /var/yuyu# pip install -r requirements.txt
```

* Menambahkan konfigurasi untuk koneksi ke RabbitMQ dan Email yang akan dipakai.
```
(env):/var/yuyu# cp yuyu/local_settings.py.sample yuyu/local_settings.py
(env):/var/yuyu# vi yuyu/local_settings.py
```
```
YUYU_NOTIFICATION_URL = "rabbit://monitoring:<RabbitMQ Pass>@<RabbitMQ IP>:5672//"
YUYU_NOTIFICATION_TOPICS = ["notifications"]

# Currency Configuration
CURRENCIES = ('IDR',)
DEFAULT_CURRENCY = "IDR"

# Email Setting
EMAIL_TAG = '[YUYU]'
DEFAULT_FROM_EMAIL = 'no-reply@btech.id'

EMAIL_BACKEND = 'django.core.mail.backends.smtp. EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_HOST_USER = '<paste your gmail account here>'
EMAIL_HOST_PASSWORD = '<paste Google password or app password here>'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
```

* Memodifikasi konfigurasi pada Yuyu agar menjalankan dan menerima request dari IP Controller.
```
(env):/var/yuyu# nano script/yuyu_api.service

# Change IP in this line
ExecStart={{yuyu_dir}}/env/bin/gunicorn yuyu.wsgi --workers 2 --bind 192.168.4.10:8182 -- log-file=logs/gunicorn.log
```
```
(env):/var/yuyu# nano yuyu/settings.py

# Add IP in this line
ALLOWED_HOSTS = ["192.168.4.10"]
```

* Menjalankan skrip untuk instalasi Yuyu API.
```
(env):/var/yuyu# ./bin/setup_api.sh
(env):/var/yuyu# systemctl enable yuyu_api
(env):/var/yuyu# systemctl start yuyu_api
(env):/var/yuyu# systemctl status yuyu_api
```

* Menjalankan skrip untuk instalasi Yuyu Event Monitor.
```
(env):/var/yuyu# ./bin/setup_event_monitor.sh
(env):/var/yuyu# systemctl enable yuyu_event_monitor
(env):/var/yuyu# systemctl start yuyu_event_monitor
(env):/var/yuyu# systemctl status yuyu_event_monitor
```

* Konfigurasi crontab agar skrip pengiriman invoice dijalankan setiap bulannya.
```
(env):/var/yuyu# crontab -e

#Add this line
1 0 1 * * /var/yuyu/bin/process_invoice.sh

(env):/var/yuyu# deactivate
```

* Download file-file yang dibutuhkan oleh Yuyu Dashboard. 
```
/var/yuyu# cd /var/
/var/yuyu# git clone https://github.com/Yuyu-billing/yuyu_dashboard.git
/var/yuyu# cd yuyu_dashboard
```

* Menjalankan skrip untuk instalasi Yuyu Dashboard. 
```
/var/yuyu_dashboard# ./setup_yuyu.sh
Specify Horizon Location (ex: /etc/horizon):
/var/www/html/horizon

/var/yuyu_dashboard# pip3 install -r requirements.txt
```

* Menjalankan perintah untuk restart HAProxy, Apache2, dan Memcached.
```
~# systemctl restart haproxy apache2
~# lxc-ls | grep "memcached"
pod-controller-memcached-container-46f8dbab

~# lxc-attach pod-controller-memcached-container-46f8dbab systemctl restart memcached
```

## 3. Setting Yuyu Billing melalui Dashboard

*	Mendapatkan password dari user admin <br>
  Note : Lakukan langkah dibawah ini di controller node. 
  ```
  ~# cat /etc/openstack_deploy/user_secrets.yml | grep keystone_auth_admin
  keystone_auth_admin_password: <Password>
  ```

* Akses Dashboard OpenStack dengan mengakses IP dari Controller.
![Horizon Dashboard Login](/media/images/openstack/horizon-with-yuyu/horizon-with-yuyu-dashboard.png)

* Untuk pengaturan Yuyu Billing bisa ikuti langkah-langkah berikut :
  * [Pengaturan harga melalui panel **“Admin” > “Billing” > “Price Configuration”**.](https://yuyu-billing.dev/price/)
  * Pengaturan informasi mengenai perusahaan melalui panel **“Admin” > “Billing” > “Billing Setting” > “Update Setting”**.
  ![Billing Setting](/media/images/openstack/horizon-with-yuyu/horizon-with-yuyu-billing.png)
  * Setelah mengisi informasi harga dan perusahaan, baru menyalakan Billing Setting.
  ![Billing Setting](/media/images/openstack/horizon-with-yuyu/horizon-with-yuyu-enable.png)

# Pengujian

* Mencoba akses panel “Admin” > “Billing” > “Billing Overview”. Pastikan data yang ditampilkan sesuai dengan  resource OpenStack yang sudah terpakai.
![Billing Overview](/media/images/openstack/horizon-with-yuyu/horizon-with-yuyu-overwiew.png)
  
# Referensi
* Penjelasan mengenai Yuyu Billing.
 	* https://yuyu-billing.dev/introduction/
*	Deploy Horizon secara manual.
 	* https://vianaja.github.io/blog-najwan/2024-12-02-secure-openstack-horizon-yuyu/
 	* https://docs.openstack.org/horizon/latest/install/install-ubuntu.html
 	* https://docs.openstack.org/releasenotes/horizon/2024.1.html
*	Deploy Heat Dashboard secara manual.
 	* https://docs.openstack.org/heat-dashboard/latest/install/index.html
*	Deploy Yuyu Billing.
 	* https://yuyu-billing.dev/pre-installation/
 	* https://yuyu-billing.dev/installation/
