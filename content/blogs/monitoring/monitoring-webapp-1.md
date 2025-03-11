---
cover:
    image: ''
    alt: "Monitoring System"
date: '2025-03-10T03:13:02Z'
title: '[PART 1] Automated WebApp Provisioning and Secure Monitoring'
tags: ["Monitoring", "Open Source", "Terraform", "Prometheus", "Grafana", "Ansible", "Container"]
category: ["Monitoring"]
ShowToc: true
---

Dalam lingkungan TI modern, banyak organisasi menghadapi tantangan dalam mengelola kombinasi aplikasi berbasis **SystemD** dan **container**. Kesulitan ini dapat menyebabkan kurangnya visibilitas terhadap performa sistem, meningkatkan risiko downtime yang tidak terdeteksi, dan menyulitkan tim dalam merespons masalah dengan cepat.  

Untuk mengatasi tantangan ini, pendekatan yang umum diterapkan adalah menetapkan satu server sebagai **pusat monitoring dan kontrol otomatisasi**. Server ini berperan dalam mengintegrasikan berbagai alat seperti **Terraform** untuk otomatisasi pembuatan infrastruktur berbasis container, serta **Ansible** untuk mengelola konfigurasi sistem, termasuk pengumpulan metrik dan pengelolaan aplikasi berbasis SystemD.  

Dari sisi pemantauan, **Prometheus** digunakan untuk mengumpulkan data performa secara real-time, sementara **Grafana** menyediakan visualisasi yang jelas. Selain itu, **Alertmanager** memungkinkan pengiriman notifikasi melalui berbagai saluran komunikasi seperti email dan Telegram, sehingga tim dapat segera merespons potensi masalah sebelum berdampak lebih luas.  

Pendekatan ini muncul dari kebutuhan mendesak untuk meningkatkan kontrol atas infrastruktur hybrid, memastikan stabilitas operasional, serta mendukung perubahan teknologi yang terus berkembang dengan lebih responsif dan efisien.

# Teori
## Apa itu Prometheus ?
Sistem monitoring open-source yang digunakan untuk mengumpulkan metrics dari server
atau aplikasi secara real-time. Data yang dikumpulkan dapat dianalisis dan dimanfaatkan untuk
memahami performa sistem, serta menyediakan mekanisme peringatan jika terjadi anomali. 

## Apa itu Grafana ?
Alat visualisasi data yang memungkinkan metrics ditampilkan dalam bentuk dashboard
interaktif. Dengan Grafana, data yang dikumpulkan dari sistem monitoring seperti Prometheus
dapat divisualisasikan dalam grafik yang lebih mudah dibaca dan dianalisis. 

## Apa itu Alert Manager?
Komponen dalam ekosistem Prometheus yang bertanggung jawab mengelola notifikasi.
Jika terjadi kondisi yang tidak diinginkan, seperti penggunaan CPU yang berlebihan atau server
mengalami kegagalan, Alert Manager akan mengirimkan peringatan melalui berbagai platform
komunikasi seperti email atau Telegram sesuai dengan aturan yang telah ditentukan. 

## Apa itu Terraform ?
Alat yang digunakan untuk mengelola infrastruktur cloud menggunakan kode
(Infrastructure as Code). Dengan Terraform, sumber daya seperti server, penyimpanan, dan
jaringan dapat dikonfigurasi secara deklaratif, sehingga proses deployment menjadi lebih efisien
dan terotomatisasi. 

## Apa itu Ansible ?
Alat otomatisasi yang digunakan untuk mengelola konfigurasi server dan aplikasi. Dengan
Ansible, kita bisa menjalankan perintah pada banyak server sekaligus, tanpa perlu mengaturnya
satu per satu. Ansible menggunakan file YAML untuk mendefinisikan tugas-tugas yang harus
dilakukan yang disebut dengan playbook. 

# Tools
* Ubuntu – Noble - v24.04
* Prometheus – v2.48.1
* Grafana – v10.2.2
* Alert Manager – v0.26.0
*  Node Exporter – v1.7.0
*  Nginx – v1.24.0
*  Nginx Exporter – v1.4.1
*  Docker – v28.0.0
*  cAdvisor – v0.47.2
*  Python – v3.12.3
*  Ansible – v2.17.9
*  Terraform – v1.11.0
*  PHP – v8.1.31
*  Composer – v2.7.1
*  Gmail
*  Telegram   

# Topologi
![Topologi](/media/images/monitoring/monitoring-webapp-topology.svg)

# Alur Kerja
![Topologi](/media/images/monitoring/monitoring-webapp-workflow.svg)

# Langkah Implementasi
## 1. Konfigurasi Akses Remote Menggunakan SSH.
Note : Lakukan langkah dibawah ini di semua node.
* Ubah hostname pada semua node.
```
Monitoring Node : ~$ sudo hostnamectl set-hostname server-monitoring
Client1 Node    : ~$ sudo hostnamectl set-hostname server-client1
Client2 Node    : ~$ sudo hostnamectl set-hostname server-client2
```

* Tambahkan name resolver secara lokal, sehingga sistem bisa mengenali hostname.
```
~$ sudo nano /etc/hosts
```
```
# Add this line below
192.168.4.10 server-monitoring
192.168.4.20 server-client1
192.168.4.30 server-client2
```

* Membuat SSH Key agar bisa melakukan akses ke Node lain secara passwordless. 
```
~$ ssh-keygen -t rsa

# Copying public key to other Node
~$ ssh-copy-id server-monitoring
~$ ssh-copy-id server-client1
~$ ssh-copy-id server-client2
```

## 2. Instalasi Tools Ansible. 
Note : Lakukan langkah dibawah ini di monitoring node.
* Melakukan update paket dan instalasi Ansible.
```
# Updating packages
~$ sudo apt update

# Installing ansible
~$ sudo apt install -y software-properties-common
~$ sudo add-apt-repository --yes --update ppa:ansible/ansible
~$ sudo apt install ansible
```

* Memastikan bahwa Ansible sudah berhasil diinstall. 
```
~$ ansible --version

ansible [core 2.17.9]
    config file = /etc/ansible/ansible.cfg
    configured module search path = ['/home/student/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
    ansible python module location = /usr/lib/python3/dist-packages/ansible
    ansible collection location = /home/student/.ansible/collections:/usr/share/ansible/collections
    executable location = /usr/bin/ansible
    python version = 3.12.3 (main, Feb 4 2025, 14:48:35)
[GCC 13.3.0] (/usr/bin/python3)
    jinja version = 3.1.2
    libyaml = True
```

## 3. Instalasi Tools Terraform. 
Note : Lakukan langkah dibawah ini di monitoring node.
* Melakukan update paket dan instalasi paket yang dibutuhkan. 
```
~$ sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
```

* Melakukan instalasi HashiCorp GPG key.
```
~$ wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
```

* Melakukan verifikasi GPG Key fingerprint. 
```
~$ gpg --no-default-keyring \
--keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
--fingerprint
```

*  Menambahkan repository HashiCorp dan instalasi Terraform.
```
# Adding HashiCorp Repository
~$ echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

# Updating packages and installing 
~$ sudo apt update
~$ sudo apt-get install terraform
```

## 4. Instalasi Node Exporter menggunakan Ansible. 
Note : Lakukan langkah dibawah ini di monitoring node.
* Menambahkan direktori untuk instalasi Node Exporter. 
```
~$ mkdir ansible-node-exporter
~$ cd ~/ansible-node-exporter
```

* Menambahkan file ansible.cfg yang berisi konfigurasi yang akan dipakai oleh Ansible.
```
~/ansible-node-exporter$ vim ansible.cfg
```
```
# Add this line below
[defaults]
inventory =. /inventory
remote_user=student
```

* Menambahkan file inventory yang berisi list dari semua node.
```
~/ansible-node-exporter$ vim inventory
```
```
# Add this line below
server-monitoring
server-client1
server-client2
```

* Menambahkan file playbook untuk instalasi Node Exporter.
```
~/ansible-node-exporter$ vim setup-node-exporter.yml
```
```
---
- name: Install Node Exporter and Set Up Systemd Service
  hosts: all
  become: yes
  vars:
    node_exporter_dir: "/opt/node_exporter"

  tasks:
    - name: Create the {{ node_exporter_dir }} directory
      file:
        path: "{{ node_exporter_dir }}"
        state: directory
        mode: '0755'

    - name: Download Node Exporter
      get_url:
        url: "https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz"
        dest: "{{ node_exporter_dir }}/node_exporter-1.7.0.linux-amd64.tar.gz"
        mode: '0644'

    - name: Extract Node Exporter using tar
      command: "tar -xzf {{ node_exporter_dir }}/node_exporter-1.7.0.linux-amd64.tar.gz -C {{ node_exporter_dir }}"

    - name: Remove the tarball after extraction
      file:
        path: "{{ node_exporter_dir }}/node_exporter-1.7.0.linux-amd64.tar.gz"
        state: absent

    - name: Make node_exporter executable
      file:
        path: "{{ node_exporter_dir }}/node_exporter-1.7.0.linux-amd64/node_exporter"
        mode: '0755'

    - name: Create systemd service for node_exporter
      copy:
        dest: /etc/systemd/system/node_exporter.service
        content: |
          [Unit]
          Description=Node Exporter
          After=network.target

          [Service]
          User=root
          ExecStart={{ node_exporter_dir }}/node_exporter-1.7.0.linux-amd64/node_exporter
          Restart=always

          [Install]
          WantedBy=multi-user.target
        mode: '0644'

    - name: Reload systemd daemon
      systemd:
        daemon_reload: yes

    - name: Enable and start node_exporter service
      systemd:
        name: node_exporter
        enabled: yes
        state: started

    - name: Verify node_exporter service status
      systemd:
        name: node_exporter
        state: started
        enabled: yes
```

* Menjalankan playbook agar instalasi Node Exporter dijalankan.
```
~/ansible-node-exporter$ ansible-playbook setup-node-exporter.yml
```

## 5. Instalasi Docker menggunakan Ansible.
Note : Lakukan langkah dibawah ini di monitoring node.
* Membuat direktori untuk instalasi Docker.
```
~$ mkdir ansible-docker
~$ cd ~/ansible-docker
```

* Menambahkan file ansible.cfg yang berisi konfigurasi yang akan dipakai oleh Ansible.
```
~/ansible-docker$ vim ansible.cfg
```
```
# Add this line below
[defaults]
inventory =. /inventory
remote_user=student
```

* Menambahkan file inventory yang berisi list dari node yang akan diinstal Docker.
```
~/ansible-docker$ vim inventory
```
```
# Add this line below
[docker_host]
server-client2
```

* Menambahkan file playbook untuk instalasi Docker.
```
~/ansible-docker$ vim setup-docker.yml
```
```
---
- name: Setup Docker with Custom Configuration
  hosts: docker_host
  become: yes
  vars:
    docker_daemon_config:
      hosts: ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"]
      experimental: true
      metrics_addr: "192.168.4.30:9323"
    override_config:
      exec_start: "/usr/bin/dockerd"

  tasks:
    - name: Install required packages
      apt:
        name:
          - ca-certificates
          - curl
        state: present
        update_cache: yes

    - name: Create /etc/apt/keyrings if it does not exist
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    - name: Download Docker GPG key
      get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /etc/apt/keyrings/docker.asc
        mode: '0644'

    - name: Configure Docker APT repository
      blockinfile:
        path: /etc/apt/sources.list.d/docker.list
        block: |
          deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release | lower }} stable

    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install Docker
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present

    - name: Add docker group if it does not exist
      group:
        name: docker
        state: present

    - name: Add user to docker group
      user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes

    - name: Create Docker daemon.json file
      copy:
        dest: /etc/docker/daemon.json
        content: |
          {
            "hosts": {{ docker_daemon_config.hosts | to_json }},
            "experimental": {{ docker_daemon_config.experimental }},
            "metrics-addr": "{{ docker_daemon_config.metrics_addr }}"
          }
        owner: root
        group: root
        mode: '0644'

    - name: Create Docker systemd override directory
      file:
        path: /etc/systemd/system/docker.service.d
        state: directory
        mode: '0755'

    - name: Create Docker systemd override file
      copy:
        dest: /etc/systemd/system/docker.service.d/override.conf
        content: |
          [Service]
          ExecStart=
          ExecStart={{ override_config.exec_start }}
        owner: root
        group: root
        mode: '0644'

    - name: Reload systemd to apply Docker configuration
      systemd:
        daemon_reload: yes

    - name: Restart Docker service
      systemd:
        name: docker
        state: restarted
        enabled: yes

    - name: Verify Docker service status
      systemd:
        name: docker
        state: started
        enabled: yes
```

* Menjalankan playbook untuk instalasi Docker. 
```
~/ansible-docker$ ansible-playbook setup-docker.yml
```
## 6. Deploy Aplikasi Web Laravel dan Nginx Exporter menggunakan Ansible. 
Note : Lakukan langkah dibawah ini di monitoring node.
* Membuat direktori untuk deploy aplikasi web Laravel dan Nginx Exporter. 
```
~$ mkdir ansible-laragigs
~$ cd ansible-laragigs
```

* Membuat Ansible roles dan ansible.cfg.
```
~/ansible-laragigs$ ansible-galaxy init laragigs
~/ansible-laragigs$ vim ansible.cfg
```
```
# Add this line below
[defaults]
inventory =. /inventory
remote_user=student
```

* Menambahkan file inventory yang berisi list dari node yang akan diinstal aplikasi web.
```
~/ansible-laragigs$ vim inventory
```
```
# Add this line below
[webapp]
server-client1
```

* Menambahkan task untuk install paket dan dependensi. 
```
~/ansible-laragigs$ vim laragigs/tasks/install.yml
```
```
- name: Install software-properties-common
  apt:
    name: software-properties-common
    state: present

- name: Add PPA for PHP from Ondrej
  shell: |
    sudo add-apt-repository -y ppa:ondrej/php
  become: yes

- name: Update apt cache
  apt:
    update_cache: yes
    force_apt_get: yes

- name: Install required packages
  apt:
    name: "{{ item }}"
    state: present
  loop: "{{ packages }}"
```

* Menambahkan task untuk konfigurasi dan instalasi database MySQL.
```
~/ansible-laragigs$ vim laragigs/tasks/database.yml
```
```
- name: Install required packages
  apt:
    name: mysql-server
    state: present

- name: Create MySQL database and user
  command: >
    mysql -u root -e "CREATE DATABASE IF NOT EXISTS {{ db_database }};
    CREATE USER IF NOT EXISTS '{{ db_username }}'@'localhost' IDENTIFIED BY '{{ db_password }}';
    GRANT ALL PRIVILEGES ON {{ db_database }}.* TO '{{ db_username }}'@'localhost';"
```

* Menambahkan task untuk konfigurasi dan deployment aplikasi web Laravel dan Nginx. 
```
~/ansible-laragigs$ vim laragigs/tasks/configure.yml
```
```
- name: Git Clone Laravel
  become: false
  git:
    repo: https://github.com/bradtraversy/laragigs.git
    dest: "{{ remote_dir }}"
    force: true

- name: Install Laravel Dependencies
  become: false
  composer:
    command: install
    arguments: "--no-dev --optimize-autoloader"
    working_dir: "{{ remote_dir }}"

- name: Configure env for the app
  become: false
  template:
    src: env.j2
    dest: "{{ remote_dir }}/.env"

- name: Generate application key
  become: false
  command: php artisan key:generate
  args:
    chdir: "{{ remote_dir }}"

- name: Migrate database
  become: false
  command: php artisan migrate
  args:
    chdir: "{{ remote_dir }}"

- name: Move from remote directory to app directory
  shell: mv "{{ remote_dir }}/" "{{ app_dir }}"

- name: Change permission for laragigs app
  file:
    path: "{{ app_dir }}"
    owner: www-data
    group: www-data
    recurse: yes

- name: Set permissions to 755 for /var/www/laragigs
  file:
    path: "{{ app_dir }}"
    mode: '0755'
    recurse: yes

- name: Creating a symbolic link for storage
  command: sudo php artisan storage:link
  args:
    chdir: "{{ app_dir }}"

- name: Configure laragigs host
  template:
    src: laragigs.j2
    dest: "{{ nginx_dir }}/sites-available/laragigs"

- name: Create a symbolic link from sites-available to sites-enabled
  file:
    src: "{{ nginx_dir }}/sites-available/laragigs"
    dest: "{{ nginx_dir }}/sites-enabled/laragigs"
    state: link

- name: Remove {{ nginx_dir }}/sites-enabled/default
  file:
    path: "{{ nginx_dir }}/sites-enabled/default"
    state: absent

- name: Restart Nginx
  systemd:
    name: nginx
    state: restarted
```

* Menambahkan task untuk instalasi dan konfigurasi Nginx Exporter. 
```
~/ansible-laragigs$ vim laragigs/tasks/exporter.yml
```
```
- name: Create the {{ nginx_exporter_dir }} directory
  file:
    path: "{{ nginx_exporter_dir }}"
    state: directory
    mode: '0755'

- name: Download Nginx Prometheus Exporter
  get_url:
    url: "https://github.com/nginx/nginx-prometheus-exporter/releases/download/v1.4.1/nginx-prometheus-exporter_1.4.1_linux_amd64.tar.gz"
    dest: "{{ nginx_exporter_dir }}/nginx-prometheus-exporter_1.4.1_linux_amd64.tar.gz"
    mode: '0644'

- name: Extract Nginx Prometheus Exporter using tar
  command: "tar -xzf {{ nginx_exporter_dir }}/nginx-prometheus-exporter_1.4.1_linux_amd64.tar.gz -C {{ nginx_exporter_dir }}"

- name: Remove the tarball after extraction
  file:
    path: "{{ nginx_exporter_dir }}/nginx-prometheus-exporter_1.4.1_linux_amd64.tar.gz"
    state: absent

- name: Make nginx-prometheus-exporter executable
  file:
    path: "{{ nginx_exporter_dir }}/nginx-prometheus-exporter"
    mode: '0755'

- name: Configure Apache for Laravel
  template:
    src: nginx_exporter.j2
    dest: /etc/systemd/system/nginx_exporter.service

- name: Reload systemd daemon
  systemd:
    daemon_reload: yes

- name: Enable and start nginx_exporter service
  systemd:
    name: nginx_exporter
    enabled: yes
    state: started
```

* Menambahkan task yang mengurutkan langkah dijalankannya task yang udah dibuat. 
```
~/ansible-laragigs$ vim laragigs/tasks/main.yml
```
```
---
# tasks file for laragigs
- name: Include installation tasks
  include_tasks: install.yml

- name: Setup database
  when: ansible_play_hosts | length == 1
  include_tasks: database.yml

- name: Check if /var/www/laragigs exists
  stat:
    path: /var/www/laragigs
  register: laragigs_dir

- name: Running configure, skip setup if /var/www/laragigs exists
  when: laragigs_dir.stat.exists == false
  include_tasks: configure.yml

- name: Setup nginx exporter
  include_tasks: exporter.yml
```

* Menambahkan variable yang akan digunakan oleh task yang udah dibuat. 
```
~/ansible-laragigs$ mkdir laragigs/vars/main
~/ansible-laragigs$ mv laragigs/vars/main.yml laragigs/vars/main
~/ansible-laragigs$ vim laragigs/vars/main/main.yml
```
```
---
# vars file for laragigs
packages:
  - php8.1
  - php8.1-fpm
  - php8.1-mysql
  - php8.1-mbstring
  - php8.1-xml
  - php8.1-zip
  - php8.1-curl
  - php8.1-common
  - php8.1-intl
  - git
  - nginx
  - composer
db_username: laragigs_user
db_database: laragigs
remote_dir: /home/student/laragigs
app_dir: /var/www/laragigs
nginx_dir: /etc/nginx
nginx_exporter_dir: /opt/nginx_exporter
```

* Menambahkan vault untuk menyimpan variable yang terjaga. 
```
~/ansible-laragigs$ ansible-vault create laragigs/vars/main/secrets.yml
New Vault password: <inputVaultPass>
Confirm New Vault password: <inputVaultPass>

# Add this lines after input a password
db_password: "<yourDBPass>"
```

* Menambahkan file template untuk environment variable yang akan dipakai aplikasi web. 
```
~/ansible-laragigs$ vim laragigs/templates/env.j2
```
```
APP_NAME=Laravel
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

LOG_CHANNEL=stack
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=debug

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE="{{ db_database }}"
DB_USERNAME="{{ db_username }}"
DB_PASSWORD="{{ db_password }}"

BROADCAST_DRIVER=log
CACHE_DRIVER=file
FILESYSTEM_DISK=local
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120

MEMCACHED_HOST=127.0.0.1

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS="hello@example.com"
MAIL_FROM_NAME="${APP_NAME}"

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=
AWS_USE_PATH_STYLE_ENDPOINT=false

PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_APP_CLUSTER=mt1

MIX_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
MIX_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
```

* Menambahkan file template untuk konfigurasi Nginx. 
```
~/ansible-laragigs$ vim laragigs/templates/laragigs.j2
```
```
server {
    listen 80;
    server_name _;

    root {{ app_dir }}/public;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock; # Sesuaikan dengan versi PHP
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }

    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 365d;
        log_not_found off;
    }

    location /nginx/status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }

    error_log /var/log/nginx/laragigs_error.log;
    access_log /var/log/nginx/laragigs_access.log;
}
```

* Menambahkan file template untuk konfigurasi Nginx Exporter. 
```
~/ansible-laragigs$ vim laragigs/templates/node_exporter.j2
```
```
[Unit]
Description=Nginx Exporter

[Service]
User=root
ExecStart={{ nginx_exporter_dir }}/nginx-prometheus-exporter --nginx.scrape-uri=http://127.0.0.1/nginx/status

[Install]
WantedBy=default.target
```

* Menambahkan playbook yang berisi tasks dari Ansible roles.
```
~/ansible-laragigs$ vim main.yml
``` 
```
- name: Use role playbook
  hosts: webapp
  become: true
  pre_tasks:
    - name: pre_tasks message
      debug:
        msg: 'Prepare to deploying laragigs app.'

  roles:
    - laragigs

  post_tasks:
    - name: post_tasks message
      debug:
        msg: 'Laragigs app is deployed!'
```

* Semua file template, tasks, dan lainnya sudah dibuat. Lalu jalankan playbook. 
```
~/ansible-laragigs$ ansible-playbook main.yml --ask-vault-pass
Vault password: <inputVaultPass>
```

## 7. Deploy Aplikasi Web berbasis Container menggunakan Terraform. 
* Membuat direktori untuk aplikasi web berbasis container. 
```
~$ mkdir terraform-docker-apps
~$ cd terraform-docker-apps
```
* Menambahkan Docker sebagai provider.
```
~/terraform-docker-apps$ vim provider.tf
```
```
terraform {
  required_providers {
    docker = {
      source = "kreuzwerker/docker"
      version = "3.0.2"
    }
  }
}

provider "docker" {
  host = "tcp://192.168.4.30:2375"
}
```

* Membuat file untuk membuat Docker image
```
~/terraform-docker-apps$ vim image.tf
```
```
# Pull cAdvisor image
resource "docker_image" "image-cadvisor" {
  name = "gcr.io/cadvisor/cadvisor:latest"
}

# Build 2048-apps image
resource "docker_image" "image-2048" {
  name = "2048-apps"
  build {
    context    = "2048-apps"
    dockerfile = "Dockerfile"
  }
}

# Build tic-tac-toe-apps image
resource "docker_image" "image-tic-tac-toe" {
  name = "tic-tac-toe-apps"
  build {
    context    = "tic-tac-toe-apps"
    dockerfile = "Dockerfile"
  }
}

# Pull Busybox image
resource "docker_image" "image-busybox" {
  name = "busybox:latest"
}
```

* Menambahkan file untuk membuat Docker container
```
~/terraform-docker-apps$ vim main.tf
```
```
# Run cAdvisor container
resource "docker_container" "container-cadvisor" {
  name  = "cadvisor"
  image = docker_image.image-cadvisor.image_id

  ports {
    internal = 8080
    external = 8080
  }

  depends_on = [
    docker_container.container-2048,
    docker_container.container-tic-tac-toe,
    docker_container.container-down-apps-1,
    docker_container.container-down-apps-2,
    docker_container.container-down-apps-3,
    docker_container.container-down-apps-4
  ]

  volumes {
    container_path = "/rootfs"
    host_path      = "/"
    read_only      = true
  }

  volumes {
    container_path = "/var/run"
    host_path      = "/var/run"
  }

  volumes {
    container_path = "/sys"
    host_path      = "/sys"
    read_only      = true
  }

  volumes {
    container_path = "/var/lib/docker/"
    host_path      = "/var/lib/docker"
    read_only      = true
  }

}

resource "docker_container" "container-2048" {
  name  = "2048-apps"
  image = docker_image.image-2048.image_id

  ports {
    internal = 80
    external = 8081
  }
}

resource "docker_container" "container-tic-tac-toe" {
  name  = "tic-tac-toe-apps"
  image = docker_image.image-tic-tac-toe.image_id

  ports {
    internal = 80
    external = 8082
  }
}

resource "docker_container" "container-down-apps-1" {
  name    = "down-apps-1"
  image   = docker_image.image-busybox.image_id
  command = ["/bin/sh", "-c", "echo 'This container will run for 1 hour' && sleep 900"]
}

resource "docker_container" "container-down-apps-2" {
  name    = "down-apps-2"
  image   = docker_image.image-busybox.image_id
  command = ["/bin/sh", "-c", "echo 'This container will run for 1 hour' && sleep 900"]
}
resource "docker_container" "container-down-apps-3" {
  name    = "down-apps-3"
  image   = docker_image.image-busybox.image_id
  command = ["/bin/sh", "-c", "echo 'This container will run for 1 hour' && sleep 900"]
}

resource "docker_container" "container-down-apps-4" {
  name    = "down-apps-4"
  image   = docker_image.image-busybox.image_id
  command = ["/bin/sh", "-c", "echo 'This container will run for 1 hour' && sleep 900"]
}
```

* Download source code dari aplikasi web 2048. 
```
~/terraform-docker-apps$ mkdir 2048-apps
~/terraform-docker-apps$ cd 2048-apps
~/terraform-docker-apps/2048-apps$ mkdir app
~/terraform-docker-apps/2048-apps$ git clone https://github.com/gabrielecirulli/2048.git app
```

* Membuat Dockerfile untuk aplikasi web 2048.
```
~/terraform-docker-apps/2048-apps$ vim Dockerfile
```
```
# Add this lines
FROM httpd:alpine
COPY ./app/ /usr/local/apache2/htdocs/
EXPOSE 80
```

* Download source code dari aplikasi web TicTacToe.
```
~/terraform-docker-apps$ mkdir tic-tac-toe-apps
~/terraform-docker-apps$ cd tic-tac-toe-apps
~/terraform-docker-apps/tic-tac-toe-apps$ mkdir app
~/terraform-docker-apps/tic-tac-toe-apps$ git clone https://github.com/Aklilu-Mandefro/javascript-Tic-Tac-Toe-game-app.git app
```

* Membuat Dockerfile untuk aplikasi web TicTacToe.
```
~/terraform-docker-apps/tic-tac-toe-apps$ vim Dockerfile
```
```
# Add this lines
FROM nginx:alpine
COPY ./app/ /usr/share/nginx/html/
EXPOSE 80
```

* Mengelola dan menerapkan konfigurasi dengan Terraform.
```
~/terraform-docker-apps$ terraform init
~/terraform-docker-apps$ terraform fmt
~/terraform-docker-apps$ terraform plan
~/terraform-docker-apps$ terraform apply --auto-approve
```

<br>

Untuk part selanjutnya bisa klik [disini](https://rifkyards.github.io/blogs/monitoring/monitoring-webapp-2/).