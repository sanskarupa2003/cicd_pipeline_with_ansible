# Ansible CI/CD Pipeline Setup

This repository contains an automated setup for a CI/CD pipeline using Ansible, Nginx, and GitHub webhooks. The setup includes automatic deployment of web applications when changes are pushed to the repository.

## Prerequisites

- Ubuntu-based system (tested on Ubuntu 20.04/22.04)
- Sudo privileges
- GitHub repository with your web application
- SSH access to your target server
  
## Setup Script (setup_ansible.sh)

```bash
#!/bin/bash

# System Updates and Dependencies
sudo apt update
sudo apt upgrade -y

# Install Python3 and pip
sudo apt install -y python3 python3-pip

# Install Ansible
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible

# Create Project Directory Structure
mkdir -p ~/ansible-cicd/{group_vars,host_vars,roles,templates,files}
cd ~/ansible-cicd

# Create Inventory File
cat > inventory.ini << 'EOL'
[web_servers]
webserver ansible_host=172.31.7.172 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/key/pro.pem

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOL

# Create Ansible Configuration
cat > ansible.cfg << 'EOL'
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = ubuntu
private_key_file = /home/ubuntu/key/pro.pem

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
EOL
```

## Ansible Playbook (site.yml)

```yaml
---
- name: Setup Web Server with CI/CD
  hosts: web_servers
  become: yes
  vars:
    github_repo: "https://github.com/sanskarupa2003/cicd_pipeline_with_ansible.git"
    website_dir: "/var/www/html"
    app_user: "ubuntu"
  
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
  
  tasks:
    - name: Remove existing website directory and its contents
      file:
        path: "{{ website_dir }}"
        state: absent
        
    - name: Recreate website directory
      file:
        path: "{{ website_dir }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0755'
    
    - name: Install required packages
      apt:
        name:
          - nginx
          - git
          - python3-pip
          - python3-venv
          - nodejs
          - npm
        state: present
        
    - name: Install required Python packages
      pip:
        name:
          - flask
          - gunicorn
          
    - name: Clone website repository
      git:
        repo: "{{ github_repo }}"
        dest: "{{ website_dir }}"
        version: main
        force: yes
        
    - name: Configure Nginx
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: Restart Nginx
      
    - name: Setup webhook service
      template:
        src: templates/webhook.service.j2
        dest: /etc/systemd/system/webhook.service
      notify: Start webhook service
      
  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
        
    - name: Start webhook service
      systemd:
        name: webhook
        enabled: yes
        state: started
        daemon_reload: yes
```

## Nginx Configuration (nginx.conf.j2)

```nginx
server {
    listen 80;
    server_name _;  # Change to server's IP or domain if needed
    
    location / {
        root /var/www/html/website;  # Replace with actual path
        index index.html;
        try_files $uri $uri/ =404;
    }
    
    location /webhook {
        proxy_pass http://172.31.7.172:5000;  # Ensure this is reachable
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### 5. Configure GitHub Webhook

1. Go to your GitHub repository settings
2. Navigate to "Webhooks" → "Add webhook"
3. Set Payload URL to: `http://YOUR_SERVER_IP/webhook`
4. Set Content type to: `application/json`
5. Set Secret to your webhook secret (default: "hello123")
6. Select "Just the push event"
7. Ensure "Active" is checked
8. Click "Add webhook"

## Running the Playbook

```bash
ansible-playbook site.yml
```

## Playbook Details

The playbook performs the following tasks:

1. Updates apt cache
2. Removes existing website directory (if any)
3. Creates new website directory
4. Installs required packages:
   - nginx
   - git
   - python3-pip
   - python3-venv
   - nodejs
   - npm
5. Installs Python packages:
   - flask
   - gunicorn
6. Clones the website repository
7. Configures Nginx
8. Sets up the webhook service

## Configuration Variables

Edit these variables in `site.yml` according to your needs:

```yaml
vars:
    github_repo: "YOUR_REPO_URL"
    website_dir: "/var/www/html"
    app_user: "ubuntu"
```

## Security Considerations

1. Change the default webhook secret in `webhook.py`
2. Use HTTPS for production environments
3. Configure proper firewall rules
4. Use environment variables for sensitive information

## Troubleshooting

1. Check Nginx logs:
```bash
sudo tail -f /var/log/nginx/error.log
```

2. Check webhook service status:
```bash
sudo systemctl status webhook
```

3. Verify webhook endpoint:
```bash
curl -X POST http://YOUR_SERVER_IP/webhook
```
