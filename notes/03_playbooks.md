# Four Pillars of a Linux Application
### Use these when planning out your ansible playbooks.

+ Packages
    - requirements
+ Service handler
    - upstart / initd / supervisor
+ System configuration
    - directories / user permissions / firewall rules
+ Application configuration
    - config files
    - customizations for this environment

## Overall process

+ Pick a [module](http://docs.ansible.com/ansible/latest/modules_by_category.html)
+ Figure out the arguments we need
+ Implement and run
+ Confirm it does what it should

## Installing Packages and Services

Using apt to install packages.

Create loadbalancer.yaml

```
---
- hosts: loadbalancer
  become: true
  tasks:
    - name: install tools
      apt: name={{item}} state=present update_cache=yes
      with_items:
        - curl
        - python-httplib2

    - name: install nginx
      apt: name=nginx state=present update_cache=yes

    - name: configure nginx site
      template: src=templates/nginx.conf.j2 dest=/etc/nginx/sites-available/demo mode=0644
      notify: restart nginx

   - name: get active sites
     shell: ls -1 /etc/nginx/sites-enabled
     register: active

    - name: deactivate default nginx site
      file: path=/etc/nginx/sites-enabled/{{ item }} state=absent
      with_items: "{{ active.stdoue_lines }}"
      when: item not in sites
      notify: restart nginx

    - name: activate demo nginx site
      file: src=/etc/nginx/sites-available/demo dest=/etc/nginx/sites-enabled/demo state=link
      notify: restart nginx

      - name: ensure nginx started
        service: name=nginx state=started enabled=yes

  handlers:
    - name: restart nginx
      service: name=nginx state=restarted

```

> **become** - Equivalent of sudo  
> **apt** - Install packaces using apt
> **name** - Name of apt package  
> **state** - Confirm this is the current state of the package (present, absent, latest, etc)  
> **update_cache** - Equivalent to apt-get update before installing the package  
> **service**: Add a service tasks using state=started.  Other states are in the docs.  
> **enabled** - Run on system startup.  

Create nginx.conf.j2 using jinja2 to create our loadbalancer config.

```
upstream demo {
{% for server in groups.webserver %}
  server {{ server }};
{% endfor %}
}

server {
  listen 80;

  location / {
    proxy_pass http://demo;
  }
}
```

Create database.yaml

```
---
- hosts: database
  become: true
  tasks:
    - name: install tools
      apt: name={{item}} state=present update_cache=yes
      with_items:
        - python-mysqldb

    - name: install mysql-server
      apt: name=mysql-server state=present update_cache=yes

    - name: ensure mysql started
      service: name=mysql state=started enabled=yes

    - name: ensure mysql listening on all interfaces
      lineinfile:
          dest: /etc/mysql/my.cnf
          regexp: "{{ item.regexp }}"
          line: "{{ item.line }}"
      with_items:
        - { regexp: '^[mysqld]', line: '[mysqld]' }
        - { regexp: '^bind-address', line: 'bind-address = 0.0.0.0' }
      notify: restart mysql

    - name: create demo database
      mysql_db: name=demo state=present

    - name: create demo user
      mysql_user: name=demo password=demo priv=demo.*:ALL host='%' state=present

  handlers:
    - name: restart mysql
      service: name=mysql state=restarted
```

> **lineinfile** - Change one or more lines in a file.  

Create webserver.yaml

```
---
- hosts: webserver
  become: true
  tasks:
    - name: install web components
      apt: name={{item}} state=present update_cache=yes

      with_items:
        - apache2
        - libapache2-mod-wsgi
        - python-pip
        - python-virtualenv
        - python-mysqldb

    - name: ensure apache2 started
      service: name=apache2 state=started enabled=yes

    - name: ensure mod_wsgi enabled
      apache2_module: state=present name=wsgi
      notify: restart apache2

    - name: copy demo app source
      copy: src=demo/app/ dest=/var/www/demo mode=0755
      notify: restart apache

    - name: copy apache virtual host config
      copy: src=demo/demo.conf dest=/etc/apache2/sites-available mode=0755
      notify: restart apache2

    - name: setup python virtualenv
      pip: requirements=/var/www/demo/requirements.txt virtualenv=/var/www/demo/.venv
      notify: restart apache2

    - name: deactivate default apache sites
      file: path=/etc/apache2/sites-enabled/000-default.conf state=absent
      notify: restart apache2

    - name: activate demo apache site
      file: src=/etc/apache2/sites-available/demo.conf dest=/etc/apache2/sites-enabled/demo.conf state=link
      notify: restart apache2

    handlers:
      - name: restart apache2
        service: name=apache2 state=restarted

```

> **with_items** - Iterates over the list and installs each {{item}} (jinja template).
> > The installation on multiple hosts runs in parallel when there are multiple hosts in a group.  

> **apache2_module** - The [ansible module][ams].  
> **notify** - If a change is made, then call the restart apache2 handler.  
> **handlers** - Like a stored task that does not run automatically, only when called with notify. If it is called 5 times within a playbook, it will only run one restart at the end.   
> **copy** - The [ansible module][ams].
> **file** - Change properties of the file directly on the host.  

## Create control playbook for our controller

```
---
- hosts: control
  become: true
  tasks:
    - name: install tools
      apt: name={{item}} state=present update_cache=yes
      with_items:
        - curl
        - python-httplib2
```

## Create stack-restart playbook

Create stack-restart.yaml

```
---
- hosts: loadbalancer
  become: true
  tasks:
    - service: name=nginx state=stopped
    - wait_for: port=80 state=drained

- hosts: webserver
  become: true
  tasks:
    - service: name=apache2 state=stopped
    - wait_for: port=80 state=stopped

- hosts: database
  become: true
  tasks:
    - service: name=mysql state=restarted
    - wait_for: port=3306 state=started

- hosts: loadbalancer
  become: true
  tasks:
    - service: name=nginx state=started
    - wait_for: port=80

- hosts: webserver
  become: true
  tasks:
    - service: name=apache2 state=started
    - wait_for: port=80
```

> We are just executing the commands to restart the entire environment
> **wait_for** - Check the status of the port, and wait for that status to become true before continue.  

## Create stack-status playbook

```
---
- hosts: loadbalancer
  become: true
  tasks:
    - name: verify nginx service
      command: service nginx status

    - name: verify nginx is listening on 80
      wait_for: port=80 timeout=1

- hosts: webserver
  become: true
  tasks:
    - name: verify apache2 service
      command: service apache2 status

    - name: verify apache is listening on 80
      wait_for: port=80 timeout=1

- hosts: database
  become: true
  tasks:
    - name: verify mysql service
      command: service mysql status

    - name: verify mysql is listening on 3306
      wait_for: port=3306 timeout=1

- hosts: control
  tasks:
    - name: verify end-to-end response
      uri: url=http://{{item}} return_content=yes
      with_items: "{{groups.loadbalancer}}"
      register: lb_index

    - fail: msg="index failed to return content"
      when: "'Hello, from sunny' not in item.content"
      with_items: "{{lb_index.results}}"

    - name: verify db response
      uri: url=http://{{item}}/db return_content=yes
      with_items: "{{groups.webserver}}"
      register: app_db

    - fail: msg="dbfailed to return content"
      when: "'Database Connected from' not in item.content"
      with_items: "{{app_db.results}}"

- hosts: loadbalancer
  tasks:
    - name: verify backend response
      uri: url=http://{{item}} return_content=yes
      with_items: "{{groups.webserver}}"
      register: app_index

    - fail: msg="index failed to return content"
      when: "'Hello, from sunny' not in item.content"
      with_items: "{{app_index.results}}"

    - name: verify db response
      uri: url=http://{{item}}/db return_content=yes
      with_items: "{{groups.webserver}}"
      register: app_db

    - fail: msg="dbfailed to return content"
      when: "'Database Connected from' not in item.content"
      with_items: "{{app_db.results}}"

```

> **uri** - Interact with endpoints and return content




[ams]:(http://docs.ansible.com/ansible/latest/list_of_all_modules.html).
