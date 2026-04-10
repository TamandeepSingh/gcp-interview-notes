# Ansible - Configuration Management & Automation

## 1. Core Concept

**Ansible** is an **agentless automation tool** that executes playbooks (YAML-based instructions) against target systems (servers, cloud resources, network devices). It's a declarative automation engine similar to Terraform but for OS-level configuration and application deployment.

**Core value:**
- **Agentless:** No agent to install, uses SSH (easy)
- **Idempotent:** Running twice produces same result
- **Simple YAML:** Easy to write and understand
- **Push model:** Control machine pushes configs to targets
- **Versatile:** Works with servers, cloud, containers, network

---

## 2. Internal Working (VERY IMPORTANT)

### Playbook Execution Flow

```
1. User: ansible-playbook site.yaml -i inventory.ini
   ↓

2. Ansible parses:
   - site.yaml (playbook with plays)
   - inventory.ini (target hosts)
   - Variables from all sources
   ↓

3. For each play in playbook:
   a. Determine target hosts (patterns: all, group, specific)
   b. For each task in play:
      - Render variables into task
      - Find module to execute
      - Push module code to target(s) via SSH
      - Execute module
      - Collect output
      - Evaluate result (changed/failed/skipped)
   c. Record result
   ↓

4. Generate report
   - How many changed
   - How many failed
   - Summary per host
```

### Inventory System

```ini
# inventory.ini
[webservers]
web-1.example.com
web-2.example.com
web-3.example.com

[dbservers]
db-1.example.com
db-2.example.com

[production:children]
webservers
dbservers

[webservers:vars]
ansible_user=ubuntu
ansible_port=22
app_port=8080
```

**Variables resolution order:**
```
1. Command-line (-e)
2. Host/Group variables
3. Playbook variables
4. Defaults
5. Facts (gathered from system)
```

### Module Execution

```yaml
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: yes
  
# Behind the scenes:
# 1. Ansible generates Python script (module)
# 2. Sftp scripts to /tmp on target
# 3. SSH execute: python /tmp/module.py
# 4. Module returns JSON:
#    {
#      "changed": true,
#      "msg": "nginx installed",
#      "rc": 0,
#      "stdout": "..."
#    }
# 5. Ansible parses JSON, evaluates state
```

### Conditional Execution

```yaml
tasks:
  - name: Install nginx if not already present
    ansible.builtin.apt:
      name: nginx
      state: present
    when: "'nginx' not in ansible_facts.packages"
  
  - name: Create db if production
    postgresql_db:
      name: appdb
    when: inventory_hostname in groups['production']
  
  - name: Configure for high memory
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    when: ansible_memtotal_mb > 8000
  
  # Handlers (run once, even if triggered multiple times)
  - name: Restart nginx
    systemd:
      name: nginx
      state: restarted
    when: config_changed | changed
```

---

## 3. Why Ansible is Needed

### Without Ansible (Manual Approach)

**Scenario: Deploy and configure 50 servers**

```bash
# Manual approach
for server in server1 server2 ... server50; do
  ssh ubuntu@$server "
    apt-get update
    apt-get install nginx
    systemctl enable nginx
    cp /local/nginx.conf /etc/nginx/nginx.conf
    systemctl restart nginx
  "
done

# Problems:
# - Error-prone (typos, inconsistency)
# - Not reproducible
# - No version control
# - Hard to roll back
# - manual efforts don't scale
```

### What Ansible Enables

✅ **Declarative Configuration:**
```yaml
- hosts: webservers
  tasks:
    - name: Install nginx
      apt: name=nginx state=present
    - name: Start nginx
      systemd: name=nginx state=started enabled=yes
```

✅ **Idempotency:**
```bash
ansible-playbook site.yaml  # Installs nginx
ansible-playbook site.yaml  # Detects already installed, no change
```

✅ **Version Control:**
```
Git repository with all playbooks
Code review before changes
Audit trail of who changed what
```

✅ **Scaling:**
```
Run same playbook against 5 or 500 servers
Parallel execution (--forks 20)
```

---

## 4. When to Use vs Not Use Ansible

### Use Ansible When:

- ✅ Server provisioning and configuration
- ✅ Application deployment
- ✅ Multi-host orchestration
- ✅ Infrastructure automation
- ✅ Need agentless configuration management

### Don't Use Ansible When:

- ❌ One-time commands (just SSH)
- ❌ Real-time system management (use tools like systemd)
- ❌ Golden image preferred (use Packer instead)
- ❌ File synchronization heavy (use rsync)

---

## 5. Real-world Ansible Usage

### Complete Playbook Example

```yaml
# site.yaml - Main playbook

---
- name: Deploy web application stack
  hosts: webservers
  become: yes  # Run with sudo
  
  vars:
    app_name: myapp
    app_version: "2.1.0"
    app_user: appuser
    app_port: 8080
    nginx_port: 80
  
  pre_tasks:
    - name: Update package cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
      
    - name: Set timezone
      timezone:
        name: UTC
    
    - name: Gather facts about system
      set_fact:
        is_prod: "{{ inventory_hostname in groups['production'] }}"
  
  roles:
    - role: base-system
      vars:
        packages:
          - git
          - curl
          - nodejs
    
    - role: docker
      docker_version: "20.10"
    
    - role: nginx
      nginx_sites:
        - name: "{{ app_name }}"
          port: "{{ nginx_port }}"
          upstream_port: "{{ app_port }}"
  
  tasks:
    - name: Create application user
      user:
        name: "{{ app_user }}"
        shell: /bin/bash
        home: "/home/{{ app_user }}"
        createhome: yes
    
    - name: Clone application repository
      git:
        repo: https://github.com/company/myapp.git
        dest: "/opt/{{ app_name }}"
        version: "v{{ app_version }}"
        force: yes
      become_user: "{{ app_user }}"
    
    - name: Install Node.js dependencies
      npm:
        path: "/opt/{{ app_name }}"
        state: present
      become_user: "{{ app_user }}"
    
    - name: Build application
      shell: npm run build
      args:
        chdir: "/opt/{{ app_name }}"
      become_user: "{{ app_user }}"
      register: build_result
    
    - name: Copy environment configuration
      template:
        src: .env.j2
        dest: "/opt/{{ app_name }}/.env"
        owner: "{{ app_user }}"
        mode: '0600'
    
    - name: Create systemd service
      template:
        src: app.service.j2
        dest: "/etc/systemd/system/{{ app_name }}.service"
      notify: restart application
    
    - name: Enable and start application service
      systemd:
        name: "{{ app_name }}"
        enabled: yes
        state: started
        daemon_reload: yes
    
    - name: Configure Nginx upstream
      template:
        src: nginx-upstream.conf.j2
        dest: "/etc/nginx/sites-available/{{ app_name }}"
      notify: reload nginx
    
    - name: Enable Nginx site
      file:
        src: "/etc/nginx/sites-available/{{ app_name }}"
        dest: "/etc/nginx/sites-enabled/{{ app_name }}"
        state: link
      notify: reload nginx
    
    - name: Set up monitoring
      block:
        - name: Install Telegraf agent
          apt:
            name: telegraf
            state: present
        
        - name: Configure Telegraf
          template:
            src: telegraf.conf.j2
            dest: /etc/telegraf/telegraf.conf
          notify: restart telegraf
        
        - name: Start Telegraf
          systemd:
            name: telegraf
            state: started
            enabled: yes
      
      when: is_prod  # Only in production
  
  handlers:
    - name: restart application
      systemd:
        name: "{{ app_name }}"
        state: restarted
    
    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
    
    - name: restart telegraf
      systemd:
        name: telegraf
        state: restarted
  
  post_tasks:
    - name: Verify application is responsive
      uri:
        url: "http://localhost:{{ app_port }}/health"
        method: GET
        status_code: 200
      retries: 5
      delay: 10
      register: health_check
    
    - name: Send deployment notification
      slack:
        channel: "#deployments"
        msg: |
          Application {{ app_name }} v{{ app_version }} deployed to {{ inventory_hostname }}
          Status: {% if health_check.status == 200 %}✅ Healthy{% else %}❌ Failed{% endif %}
      when: '"SLACK_TOKEN" in environment'

---
# Update production servers
- name: Update production infrastructure
  hosts: production_servers
  serial: 1  # One server at a time (blue-green pattern)
  
  tasks:
    - name: Remove from load balancer
      shell: /usr/local/bin/drain-server.sh {{ inventory_hostname }}
    
    - name: Run updates
      apt:
        update_cache: yes
        upgrade: safe
    
    - name: Reboot if needed
      reboot:
        test_command: uptime
      when: reboot_required | changed
    
    - name: Add back to load balancer
      shell: /usr/local/bin/undrain-server.sh {{ inventory_hostname }}
```

### Roles Structure (Reusable)

```
roles/
├── base-system/
│   ├── tasks/
│   │   └── main.yaml
│   ├── handlers/
│   │   └── main.yaml
│   ├── templates/
│   │   └── sysctl.conf.j2
│   ├── files/
│   │   └── limit.conf
│   ├── vars/
│   │   └── main.yaml
│   └── README.md
│
├── docker/
│   ├── tasks/
│   │   ├── main.yaml
│   │   ├── redhat.yaml
│   │   └── debian.yaml
│   └── defaults/
│       └── main.yaml
│
└── nginx/
    ├── tasks/
    │   └── main.yaml
    ├── templates/
    │   ├── nginx.conf.j2
    │   └── site.conf.j2
    └── handlers/
```

---

## 6. Common Mistakes

### Mistake 1: Not Making Tasks Idempotent

❌ **Wrong:**
```yaml
- name: Build application
  shell: npm run build
  # Runs every time, even if no changes
  # Hard to tell if it failed or succeeded
```

✅ **Right:**
```yaml
- name: Check if build needed
  stat:
    path: /opt/app/dist
  register: dist_exists

- name: Build application
  shell: npm run build
  when: not dist_exists.stat.exists

# Or use specialized modules:
- name: Install Node dependencies
  npm:
    path: /opt/app
    state: present
  # Module handles idempotency
```

### Mistake 2: Hardcoded Values

❌ **Wrong:**
```yaml
- name: Deploy app
  shell: deployer deploy 10.0.1.5 8080
  # IP and port hardcoded
```

✅ **Right:**
```yaml
vars:
  app_host: "{{ groups['appservers'][0] }}"
  app_port: 8080

- name: Deploy app
  shell: "deployer deploy {{ app_host }} {{ app_port }}"
```

### Mistake 3: No Error Handling

❌ **Problem:**
```yaml
- name: Start service
  systemd:
    name: myapp
    state: started
  # If fails, playbook stops (fails the whole deployment)
```

✅ **Right:**
```yaml
- name: Start service
  systemd:
    name: myapp
    state: started
  ignore_errors: yes  # Don't stop on failure
  
- name: Check if service is running
  systemd:
    name: myapp
    state: started
  failed_when: false  # Don't mark as failed
  register: service_check

- name: Alert if service failed
  debug:
    msg: "Service not running!"
  when: not service_check.is_active
```

---

## 7. Interview Questions

### Q1: Playbook Design

**Interviewer:** "Design playbook to deploy your application to 100 production servers with zero downtime."

**Good Answer:**
```yaml
# rolling-deploy.yaml
- hosts: production
  serial: 10  # 10 at a time
  
  tasks:
    - name: Remove from LB
      shell: drain-server {{ inventory_hostname }}
    
    - name: Deploy new version
      shell: deploy-app {{ app_version }}
    
    - name: Health check
      uri: url=http://localhost:8080/health
      
    - name: Add back to LB
      shell: undrain-server {{ inventory_hostname }}

Result: 10 servers at a time, gradual rollout, zero downtime
```

---

## 8. Key Takeaways

1. **Ansible = agentless configuration management** over SSH
2. **Idempotency is critical** - safe to run multiple times
3. **YAML is simple** - easy to write and understand
4. **Roles encapsulate functionality** - reusable, manageable
5. **Variables everywhere** - flexibility across environments
6. **Handlers run once** - efficient, deduplicates actions
7. **Serial deployments** - rolling updates, zero downtime
8. **Integration point** - bridges infrastructure (Terraform) and applications
