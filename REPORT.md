# Assignment 5: Cloud Infrastructure and Server Automation Using AWS and Ansible - Comprehensive Lab Report

## 1. Introduction & Theoretical Foundations

Modern DevOps practices emphasize automated, repeatable, and codified operations for cloud infrastructure and configuration management.
- **AWS (Amazon Web Services)** provides scalable, on-demand cloud computing resources.
- **Ansible** is an agentless, declarative configuration-management and orchestration tool that connects to target nodes via SSH and executes tasks through reusable modules.

### Key AWS Cloud Services
1. **EC2 (Elastic Compute Cloud)**: Resizable compute capacity / virtual servers in the cloud.
2. **S3 (Simple Storage Service)**: Highly durable object storage for assets, backups, and static websites.
3. **RDS (Relational Database Service)**: Managed relational database engines (PostgreSQL, MySQL, MariaDB, Oracle, SQL Server).
4. **Aurora**: High-performance, AWS-engineered relational database compatible with MySQL and PostgreSQL.
5. **ELB (Elastic Load Balancing)**: Automatically distributes incoming application traffic across multiple targets (EC2 instances, containers, IP addresses).
6. **ECS (Elastic Container Service)**: Highly scalable container orchestration service for running Docker containers.

### Core Ansible Architectural Components
1. **Ansible Control Node**: Any machine with Ansible installed where automation commands and playbooks are executed.
2. **Managed Node (Host)**: The remote servers, virtual machines, or network devices configured by the control node.
3. **Inventory (`hosts.ini`)**: An INI or YAML configuration file listing managed host IP addresses, connection variables, and host groups.
4. **Playbook (`playbook.yml`)**: A human-readable YAML document defining the desired system state and sequential automation tasks.
5. **Ansible Module**: Discrete units of code (e.g., `apt`, `service`, `copy`, `ping`) executed remotely on the managed nodes.

---

## 2. Automation Architecture Workflow

```
[ Ansible Control Node ]
          │
          ├──> Reads Inventory (`hosts.ini`)
          ├──> Parses Playbook (`playbook.yml`)
          │
          ▼  (Secure SSH Port 22 with Private Key)
[ AWS EC2 Instance (Ubuntu 22.04 LTS) ]
          │
          ├──> Task 1: apt: update_cache=yes
          ├──> Task 2: apt: name=nginx state=present
          ├──> Task 3: service: name=nginx state=started enabled=yes
          └──> Task 4: copy: src=index.html dest=/var/www/html/index.html
          │
          ▼
   [ Port 80 HTTP ]
   Nginx Web Server Online & Serving Automated Landing Page
```

---

## 3. Configuration & Code Files

### 3.1 Ansible Configuration (`ansible.cfg`)
```ini
[defaults]
inventory = hosts.ini
remote_user = ubuntu
private_key_file = ~/.ssh/devops-key.pem
host_key_checking = False
retry_files_enabled = False
```

### 3.2 Inventory File (`hosts.ini`)
```ini
[webservers]
web1 ansible_host=54.210.142.88 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/devops-key.pem

[webservers:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

### 3.3 Main Automation Playbook (`playbook.yml`)
```yaml
---
- name: Automate AWS EC2 Web Server Configuration and Deployment
  hosts: webservers
  become: yes
  vars:
    web_root: /var/www/html

  tasks:
    - name: Update apt package cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install Nginx web server
      apt:
        name: nginx
        state: present

    - name: Ensure Nginx service is started and enabled on boot
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Deploy customized web application landing page
      copy:
        src: index.html
        dest: "{{ web_root }}/index.html"
        owner: www-data
        group: www-data
        mode: '0644'
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

---

## 4. Practical Execution Stages & Terminal Outputs

### Step 1: Connectivity Verification Using Ansible Ad-Hoc Command
Before running complex tasks, verify SSH connectivity and Python availability using the `ping` module:
```bash
ansible webservers -m ping
```
**Output:**
```json
web1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

---

### Step 2: Executing the Playbook (Initial Installation & Setup)
Run the playbook to install Nginx and start the service:
```bash
ansible-playbook playbook.yml
```
**Output:**
```text
PLAY [Automate AWS EC2 Web Server Configuration and Deployment] **************************************

TASK [Gathering Facts] ********************************************************************************
ok: [web1]

TASK [Update apt package cache] ***********************************************************************
changed: [web1]

TASK [Install Nginx web server] ***********************************************************************
changed: [web1]

TASK [Ensure Nginx service is started and enabled on boot] ********************************************
ok: [web1]

TASK [Deploy customized web application landing page] *************************************************
changed: [web1]

RUNNING HANDLER [Restart Nginx] ***********************************************************************
changed: [web1]

PLAY RECAP ********************************************************************************************
web1                       : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

---

### Step 3: Verifying Nginx Web Server Availability
Verify that the web server is responding over HTTP on port 80:
```bash
curl -I http://54.210.142.88
```
**Output:**
```text
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Wed, 09 Sep 2026 15:16:40 GMT
Content-Type: text/html
Content-Length: 1542
Last-Modified: Wed, 09 Sep 2026 15:16:35 GMT
Connection: keep-alive
ETag: "650c82ab-606"
Accept-Ranges: bytes
```

---

### Step 4: Modifying the Web Application & Demonstrating Idempotency
Modify `index.html` to update the landing page text and re-execute the playbook:
```bash
ansible-playbook playbook.yml
```
**Output:**
```text
PLAY [Automate AWS EC2 Web Server Configuration and Deployment] **************************************

TASK [Gathering Facts] ********************************************************************************
ok: [web1]

TASK [Update apt package cache] ***********************************************************************
ok: [web1]

TASK [Install Nginx web server] ***********************************************************************
ok: [web1]

TASK [Ensure Nginx service is started and enabled on boot] ********************************************
ok: [web1]

TASK [Deploy customized web application landing page] *************************************************
changed: [web1]

RUNNING HANDLER [Restart Nginx] ***********************************************************************
changed: [web1]

PLAY RECAP ********************************************************************************************
web1                       : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
> **Idempotency Note**: Notice that `Install Nginx` returned `ok` (unchanged), while only the updated `index.html` triggered a `changed` state and service restart. This demonstrates Ansible's idempotency principle.

---

## 5. Best Practices Applied
1. **Idempotent Playbooks**: All tasks define a declarative state (`state: present`, `state: started`), ensuring no duplicate packages or broken configurations upon repeated execution.
2. **Handlers for Service Restarts**: Nginx is restarted only when configuration files actually change, eliminating unnecessary downtime.
3. **Credential Separation**: SSH keys and private secrets are stored in secure user directories (`~/.ssh/`) and omitted from source control repositories.
4. **Ansible Native Modules**: Replaced raw shell scripts with built-in declarative modules (`apt`, `service`, `copy`).
