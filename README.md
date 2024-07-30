# Documentation for Automated Deployment and Configuration with Ansible


## Overview
This guide covers automating the deployment and configuration of a boilerplate application using Ansible. It includes setting up dependencies, PostgreSQL, Redis, Nginx, and logging.

### Prerequisites
Fresh Ubuntu 22.04 Server with Python 3.12 installed.

Ansible installed on the control node.


### Project Structure

project/
│
├── main.yaml              # The Ansible playbook
├── inventory.cfg          # Ansible inventory file


### Ansible Playbook Breakdown

- Common Tasks
  
Create User with Sudo Privileges:

```bash
- name: Create user hng with sudo privileges
  user:
    name: hng
    groups: sudo
    state: present
    createhome: yes
    shell: /bin/bash
    password: '*'

- name: Allow hng user to run sudo without password
  lineinfile:
    path: /etc/sudoers.d/hng
    create: yes
    state: present
    line: 'hng ALL=(ALL) NOPASSWD:ALL'
```


### Create Directories:

```bash
- name: Create /var/secrets and /var/log/stage_5b directories
  file:
    path: "{{ item }}"
    state: directory
    owner: hng
    group: hng
    mode: '0750'
  loop:
    - /var/secrets
    - /var/log/stage_5b
```


PostgreSQL Setup

Install and Configure PostgreSQL

- name: Install PostgreSQL
  apt:
    name: postgresql
    state: present

- name: Start and enable PostgreSQL service
  service:
    name: postgresql
    state: started
    enabled: true

- name: Save PostgreSQL admin credentials
  copy:
    content: |
      POSTGRES_USER=admin
      POSTGRES_PASSWORD=secure_password
    dest: /var/secrets/pg_pw.txt
    owner: hng
    group: hng
    mode: '0600'

- name: Create PostgreSQL database and user
  become_user: postgres
  postgresql_db:
    name: myappdb

- name: Create PostgreSQL user
  become_user: postgres
  postgresql_user:
    name: admin
    password: secure_password
    priv: "ALL"

Redis Setup

Install and Configure Redis:

- name: Install Redis
  apt:
    name: redis-server
    state: present

- name: Start and enable Redis service
  service:
    name: redis-server
    state: started
    enabled: true


Node.js and Yarn Installation
Install Node.js 20.x and Yarn:

- name: Download Node.js 20.x setup script
  get_url:
    url: "https://deb.nodesource.com/setup_20.x"
    dest: /tmp/nodesource_setup.sh

- name: Run Node.js setup script
  command: bash /tmp/nodesource_setup.sh

- name: Install Node.js 20
  apt:
    name: nodejs
    state: present

- name: Add Yarn APT repository and install Yarn
  apt_repository:
    repo: 'deb https://dl.yarnpkg.com/debian/ stable main'
    state: present

- name: Install Yarn
  apt:
    name: yarn
    state: present

Application Deployment
Clone Repository and Install Dependencies:

- name: Clone repository
  git:
    repo: 'https://github.com/yourusername/yourrepo.git'
    dest: /opt/stage_5b
    version: devops
    force: yes

- name: Change ownership of /opt/stage_5b
  file:
    path: /opt/stage_5b
    state: directory
    owner: hng
    group: hng
    recurse: yes

- name: Install application dependencies
  command: yarn install
  args:
    chdir: /opt/stage_5b


Configure Environment Variables and Start Application:

- name: Configure environment variables
  copy:
    content: |
      PORT=3000
      DATABASE_URL=postgresql://admin:secure_password@localhost/myappdb
      REDIS_URL=redis://localhost:6379
    dest: /opt/stage_5b/.env
    owner: hng
    group: hng
    mode: '0600'

- name: Ensure log files exist and are owned by hng user
  file:
    path: /var/log/stage_5b
    state: directory
    owner: hng
    group: hng
    mode: '0750'

- name: Ensure stdout and stderr log files exist
  file:
    path: "{{ item }}"
    state: touch
    owner: hng
    group: hng
    mode: '0644'
  loop:
    - /var/log/stage_5b/out.log
    - /var/log/stage_5b/error.log

- name: Start application
  shell: "yarn start > /var/log/stage_5b/out.log 2> /var/log/stage_5b/error.log &"
  args:
    chdir: /opt/stage_5b
  environment:
    NODE_ENV: production

Nginx Configuration
Install and Configure Nginx:

- name: Install Nginx
  apt:
    name: nginx
    state: present

- name: Start and enable Nginx service
  service:
    name: nginx
    state: started
    enabled: true

- name: Configure Nginx to reverse proxy
  copy:
    content: |
      server {
        listen 80;
        server_name localhost;

        location / {
          proxy_pass http://127.0.0.1:3000;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection 'upgrade';
          proxy_set_header Host $host;
          proxy_cache_bypass $http_upgrade;
        }
      }
    dest: /etc/nginx/sites-available/default
    owner: root
    group: root
    mode: '0644'

- name: Test Nginx configuration
  command: nginx -t

- name: Reload Nginx
  service:
    name: nginx
    state: reloaded

Running the Playbook
Syntax Check:

ansible-playbook main.yaml --syntax-check

Dry Run:

ansible-playbook main.yaml --check -b

Execute:

ansible-playbook main.yaml -b



