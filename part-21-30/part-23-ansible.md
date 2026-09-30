# Part 23: Ansible สำหรับ Configuration Management

## บทนำ

Ansible คือ open-source automation tool ที่ใช้สำหรับ Configuration Management, Application Deployment, และ Task Automation Ansible ใช้ SSH เพื่อ connect ไปยัง remote servers โดยไม่ต้องติดตั้ง agent บน target machine

**ทำไมต้องใช้ Ansible?**
- **Agentless**: ไม่ต้องติดตั้ง software เพิ่มบน server
- **Idempotent**: รัน playbook ซ้ำได้โดยไม่เกิดผลลัพธ์ที่แตกต่าง
- **Simple YAML**: เรียนรู้ง่าย ไม่ต้องเขียน code
- **Push-based**: Control machine ส่ง configuration ไปยัง servers

---

## สารบัญ

1. Ansible คืออะไร
2. ติดตั้ง Ansible
3. Inventory Files
4. Playbooks, Roles, Tasks
5. YAML Syntax
6. Modules
7. Variables และ Facts
8. Handlers
9. Tags
10. Ansible Vault
11. Dynamic Inventory
12. เปรียบเทียบกับ Chef/Puppet
13. CI/CD Integration
14. Ansible Tower/AWX
15. แบบฝึกหัด

---

## 1. Ansible คืออะไร?

### สถาปัตยกรรมของ Ansible

```
Control Node (คุณ)
      |
      | SSH
      |
 +---------+     +---------+     +---------+
 | Node 1  |     | Node 2  |     | Node 3  |
 | (web)   |     | (db)    |     | (cache) |
 +---------+     +---------+     +---------+
```

### Components หลัก

- **Control Node**: เครื่องที่รัน Ansible
- **Managed Nodes**: เครื่อง server ที่ Ansible จัดการ
- **Inventory**: รายการ managed nodes
- **Playbook**: YAML file ที่ระบุ tasks ที่ต้องทำ
- **Module**: unit of work (apt, yum, copy, service, etc.)
- **Role**: collection ของ playbooks, templates, files
- **Plugin**: ขยาย Ansible functionality
- **Facts**: ข้อมูลเกี่ยวกับ managed nodes

---

## 2. ติดตั้ง Ansible

### ติดตั้งบน Ubuntu/Debian

```bash
# วิธีที่ 1: ผ่าน apt (version อาจเก่า)
sudo apt update
sudo apt install ansible

# วิธีที่ 2: ผ่าน pip (แนะนำ - ได้ version ล่าสุด)
pip3 install ansible

# วิธีที่ 3: ผ่าน PPA
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible

# ตรวจสอบ version
ansible --version

# ติดตั้ง collection เพิ่มเติม
ansible-galaxy collection install community.general
ansible-galaxy collection install community.mysql
ansible-galaxy collection install amazon.aws
```

### ติดตั้งบน macOS

```bash
brew install ansible

# หรือผ่าน pip
pip3 install ansible
```

### ตรวจสอบการติดตั้ง

```bash
# ดู version
ansible --version

# Test connection ไปยัง localhost
ansible localhost -m ping

# ผลลัพธ์ที่ถูกต้อง:
# localhost | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

### SSH Key Setup

```bash
# สร้าง SSH key
ssh-keygen -t ed25519 -C "ansible-control" -f ~/.ssh/ansible_key

# Copy public key ไปยัง managed nodes
ssh-copy-id -i ~/.ssh/ansible_key.pub user@192.168.1.100
ssh-copy-id -i ~/.ssh/ansible_key.pub user@192.168.1.101

# ทดสอบ
ssh -i ~/.ssh/ansible_key user@192.168.1.100
```

---

## 3. Inventory Files

### Static Inventory

```ini
# inventory/hosts.ini

# Ungrouped hosts
192.168.1.50
server.example.com

# Web servers group
[webservers]
web1.example.com ansible_host=192.168.1.100
web2.example.com ansible_host=192.168.1.101
web3.example.com ansible_host=192.168.1.102

# Database servers group
[databases]
db1.example.com ansible_host=192.168.1.200
db2.example.com ansible_host=192.168.1.201

# Cache servers group
[caches]
redis1.example.com ansible_host=192.168.1.300

# Group of groups
[production:children]
webservers
databases
caches

# Staging environment
[staging]
staging-web.example.com
staging-db.example.com

# Variables สำหรับ group
[webservers:vars]
ansible_user=ubuntu
ansible_python_interpreter=/usr/bin/python3
http_port=80

[databases:vars]
ansible_user=ubuntu
db_port=5432

# Variables สำหรับทุก host
[all:vars]
ansible_ssh_private_key_file=~/.ssh/ansible_key
```

### YAML Inventory

```yaml
# inventory/hosts.yaml

all:
  vars:
    ansible_ssh_private_key_file: ~/.ssh/ansible_key
    ansible_python_interpreter: /usr/bin/python3
  
  children:
    production:
      children:
        webservers:
          hosts:
            web1.example.com:
              ansible_host: 192.168.1.100
              nginx_port: 80
              nginx_ssl_port: 443
            web2.example.com:
              ansible_host: 192.168.1.101
              nginx_port: 80
              nginx_ssl_port: 443
          vars:
            ansible_user: ubuntu
            http_port: 80
        
        databases:
          hosts:
            db1.example.com:
              ansible_host: 192.168.1.200
              db_role: primary
            db2.example.com:
              ansible_host: 192.168.1.201
              db_role: replica
          vars:
            ansible_user: ubuntu
            postgresql_version: "15"
        
        caches:
          hosts:
            redis1.example.com:
              ansible_host: 192.168.1.300
          vars:
            redis_port: 6379
    
    staging:
      hosts:
        staging-web.example.com:
          ansible_host: 10.0.1.100
          nginx_port: 8080
        staging-db.example.com:
          ansible_host: 10.0.1.200
```

### Host Variables และ Group Variables

```
inventory/
├── hosts.yaml
├── group_vars/
│   ├── all.yaml           # ตัวแปรสำหรับทุก host
│   ├── webservers.yaml    # ตัวแปรสำหรับ webservers group
│   ├── databases.yaml     # ตัวแปรสำหรับ databases group
│   └── production/        # หรือเป็น directory
│       ├── vars.yaml
│       └── vault.yaml     # encrypted variables
└── host_vars/
    ├── web1.example.com.yaml
    └── db1.example.com.yaml
```

```yaml
# inventory/group_vars/all.yaml
---
timezone: Asia/Bangkok
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org

common_packages:
  - vim
  - curl
  - wget
  - git
  - htop
  - net-tools

ansible_user: ubuntu
ansible_become: true
ansible_become_method: sudo
```

```yaml
# inventory/group_vars/webservers.yaml
---
nginx_version: "1.24"
nginx_worker_processes: auto
nginx_worker_connections: 1024

ssl_certificate_path: /etc/nginx/ssl/cert.pem
ssl_key_path: /etc/nginx/ssl/key.pem

app_port: 8080
```

```yaml
# inventory/host_vars/web1.example.com.yaml
---
nginx_server_name: web1.example.com
backup_enabled: true
monitoring_enabled: true
```

### Inventory Commands

```bash
# List all hosts
ansible-inventory -i inventory/hosts.yaml --list

# ดู inventory graph
ansible-inventory -i inventory/hosts.yaml --graph

# ดู host variables
ansible-inventory -i inventory/hosts.yaml --host web1.example.com

# Ping ทุก hosts ใน group
ansible webservers -i inventory/hosts.yaml -m ping

# Run command บน production
ansible production -i inventory/hosts.yaml -m command -a "uptime"
```

---

## 4. Playbooks, Roles, Tasks

### Playbook Structure

```yaml
# site.yaml
---
- name: Configure web servers
  hosts: webservers
  become: true
  gather_facts: true
  
  vars:
    app_version: "2.0"
    deploy_user: "deploy"
  
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600
  
  roles:
    - common
    - nginx
    - app
  
  tasks:
    - name: Verify nginx is running
      service:
        name: nginx
        state: started
  
  post_tasks:
    - name: Run smoke tests
      uri:
        url: "http://localhost:80/health"
        status_code: 200
  
  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted

- name: Configure database servers
  hosts: databases
  become: true
  
  roles:
    - common
    - postgresql
```

### Role Structure

```
roles/
├── common/
│   ├── tasks/
│   │   └── main.yaml
│   ├── handlers/
│   │   └── main.yaml
│   ├── templates/
│   │   └── ntp.conf.j2
│   ├── files/
│   │   └── sudoers
│   ├── vars/
│   │   └── main.yaml
│   ├── defaults/
│   │   └── main.yaml      # default variable values
│   └── meta/
│       └── main.yaml
├── nginx/
│   ├── tasks/
│   │   ├── main.yaml
│   │   ├── install.yaml
│   │   └── configure.yaml
│   ├── handlers/
│   │   └── main.yaml
│   ├── templates/
│   │   ├── nginx.conf.j2
│   │   └── vhost.conf.j2
│   └── defaults/
│       └── main.yaml
└── app/
    ├── tasks/
    │   └── main.yaml
    ├── templates/
    │   └── app.service.j2
    └── defaults/
        └── main.yaml
```

### Common Role

```yaml
# roles/common/tasks/main.yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yaml"
  tags: [always]

- name: Update package cache
  package:
    update_cache: true
  when: ansible_os_family == "Debian"
  tags: [packages]

- name: Install common packages
  package:
    name: "{{ common_packages }}"
    state: present
  tags: [packages]

- name: Set timezone
  timezone:
    name: "{{ timezone }}"
  tags: [system]

- name: Configure NTP
  template:
    src: ntp.conf.j2
    dest: /etc/ntp.conf
    owner: root
    group: root
    mode: '0644'
  notify: restart ntp
  tags: [ntp]

- name: Create deploy user
  user:
    name: "{{ deploy_user }}"
    groups: ["{{ sudo_group }}"]
    shell: /bin/bash
    create_home: true
    state: present
  tags: [users]

- name: Add SSH public key for deploy user
  authorized_key:
    user: "{{ deploy_user }}"
    key: "{{ lookup('file', 'files/deploy_key.pub') }}"
    state: present
  tags: [users]

- name: Configure sudoers for deploy user
  copy:
    src: sudoers
    dest: /etc/sudoers.d/deploy
    owner: root
    group: root
    mode: '0440'
    validate: 'visudo -cf %s'
  tags: [users]

- name: Disable root SSH login
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin no'
    state: present
  notify: restart sshd
  tags: [security]

- name: Set up firewall (UFW)
  ufw:
    state: enabled
    policy: deny
    direction: incoming
  tags: [firewall]

- name: Allow SSH
  ufw:
    rule: allow
    port: ssh
    proto: tcp
  tags: [firewall]
```

### Nginx Role

```yaml
# roles/nginx/tasks/main.yaml
---
- name: Install nginx
  import_tasks: install.yaml
  tags: [nginx, install]

- name: Configure nginx
  import_tasks: configure.yaml
  tags: [nginx, configure]
```

```yaml
# roles/nginx/tasks/install.yaml
---
- name: Install nginx
  package:
    name: nginx
    state: present

- name: Ensure nginx is started and enabled
  service:
    name: nginx
    state: started
    enabled: true
```

```yaml
# roles/nginx/tasks/configure.yaml
---
- name: Create nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
  notify: reload nginx

- name: Create virtual host configuration
  template:
    src: vhost.conf.j2
    dest: "/etc/nginx/sites-available/{{ app_name }}"
    owner: root
    group: root
    mode: '0644'
  notify: reload nginx

- name: Enable virtual host
  file:
    src: "/etc/nginx/sites-available/{{ app_name }}"
    dest: "/etc/nginx/sites-enabled/{{ app_name }}"
    state: link
  notify: reload nginx

- name: Remove default nginx site
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: reload nginx

- name: Create SSL directory
  file:
    path: /etc/nginx/ssl
    state: directory
    owner: root
    group: root
    mode: '0700'
  when: nginx_ssl_enabled | default(false)

- name: Test nginx configuration
  command: nginx -t
  changed_when: false
```

```jinja2
{# roles/nginx/templates/nginx.conf.j2 #}
user www-data;
worker_processes {{ nginx_worker_processes | default('auto') }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
    multi_accept on;
    use epoll;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;
    
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;
    
    # Compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json 
               application/javascript application/rss+xml 
               application/atom+xml image/svg+xml;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate={{ nginx_rate_limit | default('10r/s') }};
    
    include /etc/nginx/sites-enabled/*;
}
```

```jinja2
{# roles/nginx/templates/vhost.conf.j2 #}
server {
    listen {{ nginx_port | default(80) }};
    server_name {{ nginx_server_name }};
    
    {% if nginx_ssl_enabled | default(false) %}
    listen {{ nginx_ssl_port | default(443) }} ssl http2;
    ssl_certificate {{ ssl_certificate_path }};
    ssl_certificate_key {{ ssl_key_path }};
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Redirect HTTP to HTTPS
    if ($scheme = http) {
        return 301 https://$server_name$request_uri;
    }
    {% endif %}
    
    root /var/www/{{ app_name }};
    index index.html;
    
    # Proxy to app
    location /api {
        proxy_pass http://127.0.0.1:{{ app_port | default(8080) }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Rate limiting
        limit_req zone=api burst={{ nginx_rate_limit_burst | default(20) }} nodelay;
    }
    
    # Static files
    location / {
        try_files $uri $uri/ /index.html;
        expires 1d;
        add_header Cache-Control "public, no-transform";
    }
    
    # Health check
    location /health {
        access_log off;
        proxy_pass http://127.0.0.1:{{ app_port | default(8080) }}/health;
    }
    
    {% for location in extra_locations | default([]) %}
    location {{ location.path }} {
        {{ location.config }}
    }
    {% endfor %}
}
```

---

## 5. YAML Syntax

### Task Syntax

```yaml
# tasks/example.yaml
---
# Basic task
- name: Install nginx
  apt:
    name: nginx
    state: present

# Task with multiple options
- name: Create directory with permissions
  file:
    path: /var/app
    state: directory
    owner: app
    group: app
    mode: '0755'
    recurse: true

# Task with loop
- name: Install multiple packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - redis-server
    - postgresql
    - python3-pip

# Loop with dict
- name: Create users
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    shell: "{{ item.shell | default('/bin/bash') }}"
  loop:
    - { name: alice, groups: ["sudo", "docker"] }
    - { name: bob, groups: ["docker"] }
    - { name: charlie, groups: ["www-data"], shell: "/usr/sbin/nologin" }

# Conditional task
- name: Install Apache (Debian only)
  apt:
    name: apache2
    state: present
  when:
    - ansible_os_family == "Debian"
    - not use_nginx | default(false)

# Register output
- name: Check if app is running
  command: systemctl is-active myapp
  register: app_status
  ignore_errors: true
  changed_when: false

- name: Print app status
  debug:
    msg: "App is {{ app_status.stdout }}"

- name: Start app if not running
  service:
    name: myapp
    state: started
  when: app_status.rc != 0

# Block สำหรับ group tasks
- name: Configure application
  block:
    - name: Create app directory
      file:
        path: /opt/myapp
        state: directory
    
    - name: Deploy app
      copy:
        src: app.tar.gz
        dest: /opt/myapp/app.tar.gz
    
    - name: Extract app
      unarchive:
        src: /opt/myapp/app.tar.gz
        dest: /opt/myapp
        remote_src: true
  
  rescue:
    - name: Clean up on failure
      file:
        path: /opt/myapp
        state: absent
    
    - name: Send failure notification
      uri:
        url: "{{ slack_webhook_url }}"
        method: POST
        body_format: json
        body:
          text: "Deployment failed on {{ inventory_hostname }}"
  
  always:
    - name: Log deployment attempt
      lineinfile:
        path: /var/log/deployments.log
        line: "{{ ansible_date_time.iso8601 }} - Deployment attempted"
        create: true
```

---

## 6. Modules

### Package Management

```yaml
# tasks/packages.yaml
---
# APT (Debian/Ubuntu)
- name: Update apt cache
  apt:
    update_cache: true
    cache_valid_time: 3600

- name: Install package
  apt:
    name:
      - nginx
      - postgresql
      - redis-server
    state: present

- name: Remove package
  apt:
    name: apache2
    state: absent
    purge: true

- name: Upgrade all packages
  apt:
    upgrade: dist
    update_cache: true

# YUM (RHEL/CentOS)
- name: Install package (YUM)
  yum:
    name:
      - httpd
      - mariadb-server
    state: present

# Package (ทั้ง apt และ yum)
- name: Install using generic package module
  package:
    name: vim
    state: present

# Pip packages
- name: Install Python packages
  pip:
    name:
      - flask
      - gunicorn
      - psycopg2-binary
    state: present
    virtualenv: /opt/myapp/venv

# NPM packages
- name: Install Node.js packages globally
  npm:
    name: pm2
    global: true
    state: present
```

### File Operations

```yaml
# tasks/files.yaml
---
# Copy file
- name: Copy config file
  copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: true

# Copy with content
- name: Create file with content
  copy:
    content: |
      # Managed by Ansible
      export APP_ENV={{ app_env }}
      export APP_PORT={{ app_port }}
      export DB_HOST={{ db_host }}
    dest: /etc/myapp/env
    owner: app
    group: app
    mode: '0600'

# Template
- name: Configure app with template
  template:
    src: app.conf.j2
    dest: /etc/myapp/app.conf
    owner: root
    group: root
    mode: '0644'
  notify: restart myapp

# Create directory
- name: Create directories
  file:
    path: "{{ item }}"
    state: directory
    owner: app
    group: app
    mode: '0755'
  loop:
    - /opt/myapp
    - /opt/myapp/logs
    - /opt/myapp/config
    - /var/run/myapp

# Create symlink
- name: Create symlink
  file:
    src: /opt/myapp/current
    dest: /var/www/app
    state: link

# Delete file
- name: Remove old config
  file:
    path: /etc/myapp/old.conf
    state: absent

# Line in file
- name: Add line to hosts file
  lineinfile:
    path: /etc/hosts
    line: "192.168.1.100 db.internal"
    state: present

# Replace line
- name: Change SSH port
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?Port '
    line: 'Port 2222'
    state: present
  notify: restart sshd

# Blockinfile
- name: Add block to config
  blockinfile:
    path: /etc/sysctl.conf
    marker: "# {mark} ANSIBLE MANAGED - Performance Tuning"
    block: |
      net.core.somaxconn = 65535
      net.ipv4.tcp_max_syn_backlog = 65535
      vm.swappiness = 10
      vm.dirty_ratio = 15

# Fetch file from remote
- name: Fetch log file
  fetch:
    src: /var/log/myapp/error.log
    dest: "logs/{{ inventory_hostname }}-error.log"
    flat: true

# Find files
- name: Find old log files
  find:
    paths: /var/log/myapp
    patterns: "*.log.gz"
    age: 30d
    recurse: true
  register: old_logs

- name: Delete old log files
  file:
    path: "{{ item.path }}"
    state: absent
  loop: "{{ old_logs.files }}"
```

### Service Management

```yaml
# tasks/services.yaml
---
- name: Start and enable nginx
  service:
    name: nginx
    state: started
    enabled: true

- name: Reload nginx
  service:
    name: nginx
    state: reloaded

- name: Restart postgresql
  service:
    name: postgresql
    state: restarted

- name: Stop and disable apache
  service:
    name: apache2
    state: stopped
    enabled: false

# systemd specific
- name: Create systemd service
  template:
    src: myapp.service.j2
    dest: /etc/systemd/system/myapp.service
  notify:
    - reload systemd
    - restart myapp

- name: Enable service
  systemd:
    name: myapp
    enabled: true
    daemon_reload: true
```

### Command Execution

```yaml
# tasks/commands.yaml
---
# Command module (no shell expansion)
- name: Run command
  command: /usr/bin/myapp --config /etc/myapp/config.yaml
  register: cmd_output
  changed_when: cmd_output.rc == 0

# Shell module (shell expansion available)
- name: Run shell command
  shell: |
    cd /opt/myapp
    ./scripts/migrate.sh 2>&1 | tee /var/log/migration.log
  args:
    chdir: /opt/myapp
  environment:
    DB_URL: "postgresql://{{ db_user }}:{{ db_password }}@{{ db_host }}/{{ db_name }}"

# Only run when file doesn't exist
- name: Initialize database
  command: /opt/myapp/bin/init-db.sh
  args:
    creates: /var/lib/myapp/.initialized

# Script
- name: Run local script on remote
  script: scripts/setup.sh arg1 arg2
  args:
    executable: /bin/bash

# Raw module (SSH only, no Python required)
- name: Bootstrap server (no Python)
  raw: "apt-get install -y python3"
  changed_when: true
```

### Database

```yaml
# tasks/database.yaml
---
# PostgreSQL
- name: Create PostgreSQL database
  community.postgresql.postgresql_db:
    name: myapp
    encoding: UTF-8
    locale: en_US.UTF-8
    state: present
  become_user: postgres

- name: Create PostgreSQL user
  community.postgresql.postgresql_user:
    name: myapp_user
    password: "{{ db_password }}"
    role_attr_flags: CREATEDB,NOSUPERUSER
    state: present
  become_user: postgres

- name: Grant database privileges
  community.postgresql.postgresql_privs:
    database: myapp
    roles: myapp_user
    objs: ALL_IN_SCHEMA
    privs: SELECT,INSERT,UPDATE,DELETE
    state: present
  become_user: postgres

# MySQL/MariaDB
- name: Create MySQL database
  community.mysql.mysql_db:
    name: myapp
    state: present
    login_user: root
    login_password: "{{ mysql_root_password }}"

- name: Create MySQL user
  community.mysql.mysql_user:
    name: myapp_user
    password: "{{ db_password }}"
    priv: "myapp.*:ALL"
    host: "%"
    state: present
    login_user: root
    login_password: "{{ mysql_root_password }}"
```

---

## 7. Variables และ Facts

### Variable Types

```yaml
# playbook variables
vars:
  app_name: myapp
  app_version: "2.0"
  app_port: 8080

# Variable files
vars_files:
  - vars/common.yaml
  - "vars/{{ ansible_os_family }}.yaml"
  - vars/secrets.yaml  # encrypted with vault

# Runtime variables (-e flag)
# ansible-playbook site.yaml -e "app_version=2.1 deploy=true"
```

### Facts

```yaml
# tasks/facts.yaml
---
# Facts จะถูก gather โดย default
# เข้าถึงผ่าน ansible_facts หรือ ansible_ prefix

- name: Show OS information
  debug:
    msg: |
      OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
      Architecture: {{ ansible_architecture }}
      Hostname: {{ ansible_hostname }}
      IP: {{ ansible_default_ipv4.address }}
      Memory: {{ ansible_memtotal_mb }} MB
      CPUs: {{ ansible_processor_count }}
      Python: {{ ansible_python_version }}

# Custom facts
- name: Create custom fact directory
  file:
    path: /etc/ansible/facts.d
    state: directory

- name: Create application fact
  copy:
    content: |
      [app]
      version={{ app_version }}
      deployed_at={{ ansible_date_time.iso8601 }}
      deployed_by={{ ansible_user_id }}
    dest: /etc/ansible/facts.d/myapp.fact
    mode: '0644'

# ใช้ custom facts
- name: Check app version
  debug:
    msg: "App version: {{ ansible_local.myapp.app.version }}"

# Disable facts gathering (เร็วขึ้น)
- hosts: webservers
  gather_facts: false
  
  tasks:
    - name: Just install package (no facts needed)
      apt:
        name: vim
        state: present

# Gather specific facts
- hosts: webservers
  gather_facts: true
  gather_subset:
    - network
    - hardware
```

### Jinja2 Filters

```yaml
# tasks/filters.yaml
---
- name: String filters
  debug:
    msg: |
      upper: {{ app_name | upper }}
      lower: {{ app_name | lower }}
      title: {{ app_name | title }}
      length: {{ app_name | length }}
      replace: {{ app_name | replace('app', 'service') }}
      trim: {{ '  hello  ' | trim }}
      default: {{ undefined_var | default('fallback') }}

- name: List filters
  debug:
    msg: |
      join: {{ ['a', 'b', 'c'] | join(', ') }}
      sort: {{ [3, 1, 2] | sort | list }}
      unique: {{ [1, 2, 2, 3] | unique | list }}
      min: {{ [3, 1, 2] | min }}
      max: {{ [3, 1, 2] | max }}
      first: {{ [1, 2, 3] | first }}
      last: {{ [1, 2, 3] | last }}
      flatten: {{ [[1, 2], [3, 4]] | flatten }}

- name: Dict filters
  debug:
    msg: |
      keys: {{ my_dict | dict2items | map(attribute='key') | list }}
      values: {{ my_dict | dict2items | map(attribute='value') | list }}
      combine: {{ {'a': 1} | combine({'b': 2}) }}

- name: Type filters
  debug:
    msg: |
      int: {{ '42' | int }}
      float: {{ '3.14' | float }}
      bool: {{ 'true' | bool }}
      string: {{ 42 | string }}
      list: {{ 'a,b,c' | split(',') }}

- name: JSON/YAML filters
  debug:
    msg: |
      to_json: {{ my_dict | to_json }}
      to_yaml: {{ my_dict | to_yaml }}
      to_nice_json: {{ my_dict | to_nice_json }}
      from_json: {{ json_string | from_json }}

- name: Hash filters
  debug:
    msg: |
      md5: {{ 'hello' | md5 }}
      sha1: {{ 'hello' | sha1 }}
      sha256: {{ 'hello' | hash('sha256') }}
      password_hash: {{ 'mypassword' | password_hash('sha512') }}
```

---

## 8. Handlers

```yaml
# handlers/main.yaml
---
- name: restart nginx
  service:
    name: nginx
    state: restarted

- name: reload nginx
  service:
    name: nginx
    state: reloaded

- name: restart postgresql
  service:
    name: postgresql
    state: restarted

- name: restart myapp
  service:
    name: myapp
    state: restarted

- name: reload systemd
  systemd:
    daemon_reload: true

- name: restart sshd
  service:
    name: sshd
    state: restarted

# Handler ที่ call handler อื่น
- name: rebuild application
  command: /opt/myapp/scripts/rebuild.sh
  listen: "restart myapp"

# Handlers ใน playbook
- name: Configure services
  hosts: webservers
  
  tasks:
    - name: Update nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - test nginx config
        - reload nginx
  
  handlers:
    - name: test nginx config
      command: nginx -t
      changed_when: false
    
    - name: reload nginx
      service:
        name: nginx
        state: reloaded
      listen: "reload nginx"
```

---

## 9. Tags

```yaml
# playbook with tags
---
- hosts: webservers
  tasks:
    - name: Update packages
      apt:
        upgrade: dist
      tags:
        - update
        - packages
        - maintenance
    
    - name: Install nginx
      apt:
        name: nginx
        state: present
      tags:
        - nginx
        - install
        - packages
    
    - name: Configure nginx
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags:
        - nginx
        - configure
      notify: reload nginx
    
    - name: Deploy application
      include_tasks: deploy.yaml
      tags:
        - deploy
        - app
    
    - name: Run smoke tests
      include_tasks: smoke_tests.yaml
      tags:
        - test
        - smoke
        - never  # ต้อง explicit ระบุจึงจะรัน
```

```bash
# Run tasks เฉพาะ tag
ansible-playbook site.yaml --tags "nginx,deploy"

# Skip tasks ด้วย tag
ansible-playbook site.yaml --skip-tags "update,maintenance"

# List tags
ansible-playbook site.yaml --list-tags

# Run tasks ที่ marked "never"
ansible-playbook site.yaml --tags "never,smoke"
```

---

## 10. Ansible Vault

Ansible Vault ใช้สำหรับ encrypt sensitive data

```bash
# สร้าง encrypted file ใหม่
ansible-vault create secrets.yaml

# Encrypt ไฟล์ที่มีอยู่แล้ว
ansible-vault encrypt vars/secrets.yaml

# Decrypt ไฟล์
ansible-vault decrypt vars/secrets.yaml

# แก้ไข encrypted file
ansible-vault edit vars/secrets.yaml

# View encrypted file
ansible-vault view vars/secrets.yaml

# Rekey (เปลี่ยน password)
ansible-vault rekey vars/secrets.yaml

# Encrypt ค่าเดี่ยว (inline encryption)
ansible-vault encrypt_string 'my-secret-password' --name 'db_password'

# ผลลัพธ์:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   66386439...
```

### Vault Files

```yaml
# vars/vault.yaml (encrypted)
---
db_password: super-secret-password
redis_password: another-secret
api_key: my-api-key-123
ssl_private_key: |
  -----BEGIN RSA PRIVATE KEY-----
  MIIEowIBAAKCAQEA...
  -----END RSA PRIVATE KEY-----
```

```yaml
# vars/main.yaml (unencrypted)
---
db_host: localhost
db_port: 5432
db_name: myapp
db_user: myapp_user
db_password: "{{ vault_db_password }}"  # reference to vault variable
```

```bash
# รัน playbook พร้อม vault password
ansible-playbook site.yaml --ask-vault-pass

# ใช้ password file (ไม่ควร commit ไป git!)
echo "my-vault-password" > ~/.vault-password
chmod 600 ~/.vault-password
ansible-playbook site.yaml --vault-password-file ~/.vault-password

# ใช้ environment variable
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault-password
ansible-playbook site.yaml

# Multiple vault IDs
ansible-playbook site.yaml \
  --vault-id dev@~/.vault-password-dev \
  --vault-id prod@~/.vault-password-prod
```

### Vault ใน CI/CD (GitHub Actions)

```yaml
# .github/workflows/ansible.yml
- name: Create vault password file
  run: echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > .vault-password

- name: Run Ansible playbook
  run: |
    ansible-playbook \
      -i inventory/production.yaml \
      --vault-password-file .vault-password \
      site.yaml

- name: Remove vault password file
  if: always()
  run: rm -f .vault-password
```

---

## 11. Dynamic Inventory

### AWS EC2 Dynamic Inventory

```bash
# ติดตั้ง amazon.aws collection
ansible-galaxy collection install amazon.aws

# สร้าง inventory config
cat > inventory/aws_ec2.yaml << 'EOF'
plugin: amazon.aws.aws_ec2
regions:
  - ap-southeast-1
  - ap-southeast-2

filters:
  instance-state-name: running
  tag:ManagedBy: ansible
  
groups:
  webservers: "'webserver' in tags.Role"
  databases: "'database' in tags.Role"
  production: "tags.Environment == 'production'"
  staging: "tags.Environment == 'staging'"

keyed_groups:
  - key: tags.Environment
    prefix: env
  - key: tags.Role
    prefix: role
  - key: placement.region
    prefix: aws_region

compose:
  ansible_host: public_ip_address
  ansible_user: "'ubuntu' if image_id.startswith('ami-0c') else 'ec2-user'"

hostnames:
  - tag:Name
  - dns-name
  - private-ip-address
EOF

# ทดสอบ
ansible-inventory -i inventory/aws_ec2.yaml --list
ansible-inventory -i inventory/aws_ec2.yaml --graph

# รัน playbook
ansible-playbook -i inventory/aws_ec2.yaml site.yaml
```

### GCP Dynamic Inventory

```yaml
# inventory/gcp_compute.yaml
plugin: google.cloud.gcp_compute
projects:
  - my-gcp-project
regions:
  - asia-southeast1
auth_kind: application
filters:
  - status = RUNNING
  - labels.managed_by = ansible

groups:
  webservers: "'webserver' in labels"
  databases: "'database' in labels"

compose:
  ansible_host: networkInterfaces[0].accessConfigs[0].natIP
  ansible_user: "'ubuntu'"
```

### Custom Dynamic Inventory Script

```python
#!/usr/bin/env python3
# inventory/custom_inventory.py

import json
import argparse
import sys
import requests

def get_inventory():
    """ดึง inventory จาก internal API"""
    
    # ดึงข้อมูลจาก CMDB หรือ service discovery
    response = requests.get(
        "https://cmdb.internal/api/servers",
        headers={"Authorization": "Bearer my-token"}
    )
    servers = response.json()
    
    inventory = {
        "_meta": {
            "hostvars": {}
        },
        "all": {
            "hosts": [],
            "vars": {
                "ansible_user": "ubuntu",
                "ansible_ssh_private_key_file": "~/.ssh/ansible_key"
            }
        }
    }
    
    groups = {}
    
    for server in servers:
        hostname = server["hostname"]
        inventory["all"]["hosts"].append(hostname)
        
        # เพิ่ม host variables
        inventory["_meta"]["hostvars"][hostname] = {
            "ansible_host": server["ip_address"],
            "server_role": server["role"],
            "environment": server["environment"],
            "region": server["region"]
        }
        
        # จัดกลุ่มตาม role
        role = server["role"]
        if role not in groups:
            groups[role] = {"hosts": []}
        groups[role]["hosts"].append(hostname)
        
        # จัดกลุ่มตาม environment
        env = server["environment"]
        env_key = f"env_{env}"
        if env_key not in groups:
            groups[env_key] = {"hosts": []}
        groups[env_key]["hosts"].append(hostname)
    
    inventory.update(groups)
    return inventory

def get_host(hostname):
    """ดึง variables สำหรับ host เดียว"""
    response = requests.get(
        f"https://cmdb.internal/api/servers/{hostname}",
        headers={"Authorization": "Bearer my-token"}
    )
    return response.json()

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--list", action="store_true")
    parser.add_argument("--host")
    args = parser.parse_args()
    
    if args.list:
        print(json.dumps(get_inventory(), indent=2))
    elif args.host:
        print(json.dumps(get_host(args.host), indent=2))
    else:
        parser.print_help()
        sys.exit(1)
```

---

## 12. เปรียบเทียบกับ Chef/Puppet

| Feature | Ansible | Chef | Puppet |
|---------|---------|------|--------|
| Language | YAML | Ruby DSL | Puppet DSL |
| Architecture | Push (agentless) | Pull (agent) | Pull (agent) |
| Learning Curve | ต่ำ | สูง | ปานกลาง |
| Setup | ง่าย | ซับซ้อน | ปานกลาง |
| Scalability | ดี | ดีมาก | ดีมาก |
| Community | ใหญ่ | ปานกลาง | ปานกลาง |
| Enterprise | Ansible Tower/AWX | Chef Automate | Puppet Enterprise |

---

## 13. CI/CD Integration

### GitHub Actions + Ansible

```yaml
# .github/workflows/deploy.yml
name: Deploy with Ansible

on:
  push:
    branches: [main]
    paths:
      - 'ansible/**'
      - '.github/workflows/deploy.yml'

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
    
    - name: Install Ansible and dependencies
      run: |
        pip install ansible boto3 botocore
        ansible-galaxy collection install -r ansible/requirements.yaml
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions-ansible
        aws-region: ap-southeast-1
    
    - name: Setup SSH key
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.ANSIBLE_SSH_PRIVATE_KEY }}" > ~/.ssh/ansible_key
        chmod 600 ~/.ssh/ansible_key
        ssh-keyscan -H ${{ secrets.KNOWN_HOSTS }} >> ~/.ssh/known_hosts 2>/dev/null
    
    - name: Create vault password file
      run: |
        echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > .vault-password
        chmod 600 .vault-password
    
    - name: Run Ansible Lint
      run: |
        pip install ansible-lint
        ansible-lint ansible/site.yaml
    
    - name: Run Ansible Playbook (Check Mode)
      run: |
        ansible-playbook \
          -i ansible/inventory/aws_ec2.yaml \
          --vault-password-file .vault-password \
          --check --diff \
          ansible/site.yaml
    
    - name: Run Ansible Playbook (Apply)
      run: |
        ansible-playbook \
          -i ansible/inventory/aws_ec2.yaml \
          --vault-password-file .vault-password \
          -v \
          ansible/site.yaml
      env:
        ANSIBLE_FORCE_COLOR: "1"
        ANSIBLE_HOST_KEY_CHECKING: "False"
    
    - name: Cleanup sensitive files
      if: always()
      run: |
        rm -f .vault-password
        rm -f ~/.ssh/ansible_key
    
    - name: Notify on success
      if: success()
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {"text": "✅ Ansible deployment completed successfully on production"}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Ansible Requirements File

```yaml
# ansible/requirements.yaml
---
roles:
  - name: geerlingguy.nginx
    version: "3.2.0"
  - name: geerlingguy.postgresql
    version: "3.4.2"
  - src: git+https://github.com/my-org/ansible-role-myapp.git
    name: myapp
    version: "v1.0.0"

collections:
  - name: community.general
    version: ">=7.0.0"
  - name: community.mysql
    version: ">=3.0.0"
  - name: amazon.aws
    version: ">=6.0.0"
  - name: google.cloud
    version: ">=1.0.0"
```

---

## 14. Ansible Tower/AWX

AWX คือ open-source version ของ Ansible Tower

```bash
# ติดตั้ง AWX บน Kubernetes
helm repo add awx-operator https://ansible.github.io/awx-operator/
helm repo update

# สร้าง namespace
kubectl create namespace awx

# ติดตั้ง AWX Operator
helm install awx-operator awx-operator/awx-operator \
  -n awx \
  --set image.tag=2.7.2

# Deploy AWX instance
cat > awx-instance.yaml << 'EOF'
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: LoadBalancer
  postgres_storage_class: standard
  projects_persistence: true
  projects_storage_size: 8Gi
  projects_storage_class: standard
EOF

kubectl apply -f awx-instance.yaml

# รอจนพร้อม
kubectl wait \
  --for=condition=Ready pod \
  -l app.kubernetes.io/name=awx-web \
  -n awx \
  --timeout=10m

# ดึง admin password
kubectl get secret awx-admin-password -n awx \
  -o jsonpath='{.data.password}' | base64 -d

# Get AWX URL
kubectl get service awx-service -n awx
```

---

## 15. แบบฝึกหัด

### แบบฝึกหัดที่ 1: First Playbook

```yaml
# exercise-1.yaml
# สร้าง playbook ที่ configure web server:
# 1. ติดตั้ง nginx
# 2. สร้าง static website
# 3. Configure firewall
# 4. Ensure nginx started

---
- name: Configure Web Server
  hosts: localhost
  connection: local
  become: true
  
  vars:
    website_title: "My First Ansible Website"
    server_port: 80
  
  tasks:
    # TODO: ติดตั้ง nginx
    
    # TODO: สร้าง /var/www/html/index.html
    # ใช้ copy module กับ content parameter
    
    # TODO: Start nginx
    
    # TODO: Verify nginx is accessible
    - name: Test website
      uri:
        url: "http://localhost:{{ server_port }}"
        status_code: 200
      delegate_to: localhost
```

### แบบฝึกหัดที่ 2: สร้าง Role

```bash
# สร้าง role skeleton
ansible-galaxy role init exercise-2-role

# โครงสร้างที่ต้องการ:
exercise-2-role/
├── tasks/main.yaml     # ติดตั้งและ configure application
├── handlers/main.yaml  # restart services
├── templates/          # config file templates
├── files/              # static files
├── vars/main.yaml      # variables
├── defaults/main.yaml  # default variables (override ได้)
└── meta/main.yaml

# Role ต้องทำ:
# 1. ติดตั้ง nodejs
# 2. Create app user
# 3. Clone/deploy application
# 4. สร้าง systemd service
# 5. Start application
```

### แบบฝึกหัดที่ 3: Ansible Vault

```bash
# 1. สร้าง vault password
echo "my-super-secret-vault-password" > .vault-pass
chmod 600 .vault-pass

# 2. สร้าง encrypted secrets file
# vault.yaml ควรมี:
# - db_password
# - api_key
# - smtp_password
ansible-vault create --vault-password-file .vault-pass vault.yaml

# 3. สร้าง playbook ที่ใช้ vault variables
cat > exercise-3.yaml << 'EOF'
---
- name: Test Ansible Vault
  hosts: localhost
  connection: local
  
  vars_files:
    - vault.yaml
  
  tasks:
    - name: Print masked secret
      debug:
        msg: "DB Password length: {{ db_password | length }}"
    
    - name: Create config with secrets
      copy:
        content: |
          DATABASE_PASSWORD={{ db_password }}
          API_KEY={{ api_key }}
        dest: /tmp/app.env
        mode: '0600'
EOF

# 4. รัน playbook
ansible-playbook --vault-password-file .vault-pass exercise-3.yaml
```

### แบบฝึกหัดที่ 4: Dynamic Inventory + Deployment

```python
# exercise-4-inventory.py
#!/usr/bin/env python3
"""
Dynamic inventory script ที่อ่านจาก JSON file
(จำลอง server registry)
"""

import json
import argparse
import sys

SERVERS = [
    {"hostname": "web1", "ip": "192.168.1.100", "role": "webserver", "env": "production"},
    {"hostname": "web2", "ip": "192.168.1.101", "role": "webserver", "env": "production"},
    {"hostname": "db1", "ip": "192.168.1.200", "role": "database", "env": "production"},
    {"hostname": "staging-web", "ip": "192.168.2.100", "role": "webserver", "env": "staging"},
]

# TODO: Implement get_inventory() function
# ต้องคืน JSON ในรูปแบบ:
# {
#   "_meta": {"hostvars": {...}},
#   "webservers": {"hosts": [...]},
#   "databases": {"hosts": [...]},
#   "production": {"hosts": [...]},
#   "staging": {"hosts": [...]}
# }

def get_inventory():
    pass  # TODO: implement

def get_host(hostname):
    pass  # TODO: implement

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--list", action="store_true")
    parser.add_argument("--host")
    args = parser.parse_args()
    
    if args.list:
        print(json.dumps(get_inventory(), indent=2))
    elif args.host:
        print(json.dumps(get_host(args.host), indent=2))
```

### แบบฝึกหัดที่ 5: Complete Deployment Pipeline

```yaml
# exercise-5-playbook.yaml
# สร้าง playbook สำหรับ deploy web application:
# 1. Check prerequisites
# 2. Backup current version
# 3. Deploy new version
# 4. Run migrations
# 5. Restart services
# 6. Run smoke tests
# 7. Rollback if tests fail

---
- name: Deploy Application
  hosts: webservers
  become: true
  serial: 1  # deploy one server at a time
  
  vars:
    app_name: myapp
    app_version: "{{ version | default('latest') }}"
    app_dir: /opt/myapp
    backup_dir: /opt/myapp-backups
    smoke_test_url: "http://{{ ansible_host }}/health"
  
  tasks:
    # TODO: ตรวจสอบ disk space
    
    # TODO: Backup current version
    
    # TODO: Deploy new version
    
    # TODO: Run database migrations
    
    # TODO: Restart application service
    
    # TODO: Wait for service to be ready
    
    # TODO: Run smoke tests
    
    # TODO: Rollback ถ้า smoke tests fail
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:

1. **Ansible Basics**: Agentless automation ด้วย SSH
2. **Inventory**: Static และ Dynamic inventory
3. **Playbooks/Roles**: การจัดการ configuration เป็น modules
4. **Modules**: apt, yum, copy, template, service, command และอื่นๆ
5. **Variables/Facts**: การจัดการ data ใน playbooks
6. **Vault**: Encryption สำหรับ sensitive data
7. **CI/CD Integration**: GitHub Actions + Ansible

**Key Takeaways:**
- Ansible เหมาะสำหรับ configuration management และ ad-hoc automation
- ใช้ Vault เสมอสำหรับ sensitive data
- เขียน playbooks แบบ idempotent
- ใช้ roles สำหรับ reusable components
- Test playbooks ด้วย `--check` mode ก่อน apply จริง

**ใน Part 24** เราจะเรียนรู้เรื่อง Kubernetes Fundamentals

---

## แหล่งอ้างอิง

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)
- [Molecule Testing](https://molecule.readthedocs.io/)
- [AWX Documentation](https://ansible.readthedocs.io/projects/awx/en/latest/)
