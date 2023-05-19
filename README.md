# Assessement #2

Commands in master node

Input Database Info Into env File
```
echo '
DJANGO_SETTINGS_MODULE=to_do_proj.settings
DB_NAME=todolist
DB_USER=postgres
DB_PASSWORD='PASSWORDS'
DB_HOST=db1.chzveui56egk.us-east-1.rds.amazonaws.com
DB_PORT=5432
SECRET_KEY='YOUR KEY' > env
```
Copy Pem file and change permissions

Create inventory.ini
```
[webservers]
node1 ansible_host=54.210.196.158 ansible_user=ubuntu
node2 ansible_host=18.234.62.91 ansible_user=ubuntu

[all:vars]
ansible_ssh_private_key_file=/home/ubuntu/'YOUR PEM'
repo_url=https://github.com/chandradeoarya/
repo=todo-list
home_dir=/home/ubuntu
repo_dir={{ home_dir }}/{{ repo }}
django_project=to_do_proj

[defaults]
host_key_checking=no
```

Check accessilbility 
```
ansible all -m ping -i inventory.ini
```

Run updates.yml
```
---
- hosts: all
  become: yes
  become_user: root
  gather_facts: no
  tasks:
    - name: Runing system update
      apt: update_cache=yes
        upgrade=safe
      register: result
    - debug: var=result.stdout_lines
ansible-playbook -i inventory.ini updates.yml
```

Run packages.yml
```
---
- hosts: all
  become: yes
  become_user: root
  gather_facts: no
  tasks:
    - name: Running apt update
      apt: update_cache=yes
    - name: Installing required packages
      apt: name={{item}} state=present
      with_items:
       - python3.10-venv
       - python-pip
       - nginx
ansible-playbook -i inventory.ini packages.yml
```

Install python packages and run code from GH
```
- hosts: all
  become: yes
  become_user: ubuntu
  gather_facts: no

  tasks:
    - name: pull branch master
      git:
        repo: "{{ repo_url }}/{{ repo }}.git"
        dest: "{{ repo_dir }}"
        accept_hostkey: yes

- hosts: all
  gather_facts: no
  tasks:
    - name: Create virtual environment
      command: python3 -m venv venv
      args:
        chdir: "{{ repo_dir }}"

    - name: install python requirements
      pip:
        requirements: "{{ repo_dir }}/requirements.txt"
        state: present
        executable: "{{ repo_dir }}/venv/bin
  ```
  
Copy Env
```
---
- name: Set environment variables on hosts
  hosts: all
  become: true
  become_user: ubuntu
  tasks:
    - name: Copy env file to hosts
      copy:
        src: /home/ubuntu/todolist/env
        dest: /home/ubuntu/todo-list/.env
        mode: 0644
ansible-playbook -i inventory.ini copyenv.yml
```

Run Daemon services
```
echo '
[Unit]
Description=Gunicorn instance to serve todolist

Wants=network.target
After=syslog.target network-online.target

[Service]
Type=simple
WorkingDirectory=/home/ubuntu/todo-list
ExecStart=/home/ubuntu/todo-list/venv/bin/gunicorn -c /home/ubuntu/todo-list/gunicorn_config.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target' > todolist.service
```

Daemon ansible playbook
```
---
- hosts: all
  become: yes
  become_user: root
  gather_facts: no
  tasks:
    - name: Copy Gunicorn systemd service file
      template:
        src: /home/ubuntu/todolist/todolist.service
        dest: /etc/systemd/system/todolist.service
      register: gunicorn_service

    - name: Enable and start Gunicorn service
      systemd:
        name: todolist
        state: started
        enabled: yes
      when: gunicorn_service.changed
      notify:
        - Restart Gunicorn

    - name: Restart Gunicorn
      systemd:
        name: todolist
        state: restarted
      when: gunicorn_service.changed

  handlers:
    - name: Restart Gunicorn
      systemd:
        name: todolist
        state: restarted
ansible-playbook -i inventory.ini gunicorn.yml
```

Create NginX port forwarding config file
```
server {
    listen 80;

    server_name public_ip;

    location / {
        proxy_pass http://localhost:9876;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

NGinX Playbook
```
---
- name: Configure Nginx port forwarding
  hosts: all
  become: true
  become_user: root
  gather_facts: no
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Configure Nginx
      template:
        src: ~/project-two/todolist
        dest: /etc/nginx/sites-available/todolist
        owner: root
        group: root
        mode: 0644
      notify: Restart Nginx

    - name: Change public_ip in Nginx configuration
      replace:
        path: /etc/nginx/sites-available/todolist
        regexp: 'server_name public_ip;'
        replace: 'server_name {{ ansible_host }};'

    - name: Enable Nginx site
      file:
        src: /etc/nginx/sites-available/todolist
        dest: /etc/nginx/sites-enabled/todolist
        state: link
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

![Diagram](https://github.com/cth9191/A2/assets/129889057/0041f923-cff6-4faf-a9d1-89bd144fc549)

