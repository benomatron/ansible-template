# Roles

## Roles make it easy to reuse code for different environments.

Create a roles directory and create a skeleton for our roles

```
mkdir roles
cd roles
ansible-galaxy init control
ansible-galaxy init nginx
ansible-galaxy init apache2
ansible-galaxy init demo_app
ansible-galaxy init mysql
```

1. Migrate all of the playbook tasks, handlers, etc to their respective role directory.
2. For example, webserver.yml tasks get moved to roles/apache2/tasks/main.yml.
3. In the case of the webserver, it has two parts apache and demo_app.  Create separate roles for each and include them in the playbook.
4. Move files to their directory under the role.  The `demo` directory was moved to roles/demo_app/files.  nginx.conf.j2 was moved to roles/nginx/templates.

```
---
- hosts: webserver
  become: true
  roles:
    - apache2
    - demo_app
```

## site.yml

> **site.yml** is a top level playbook that executes all of the other playbooks using **include**.  

```
---
- include: control.yml
- include: database.yml
- include: webserver.yml
- include: loadbalancer.yml
```

## External Roles / Galaxy

Galaxy is an online repo for roles that other people contribute and you can use.  Similar to Chef supermarket or puppet forge.
There is likely a role that already exists if you are installing some common components are applications.

`ansible-galaxy install username.rolename`

A few considerations:

1. You wan't roles that have been around for a long time and that it has a high rating.
2. Also, check how much coverage it has for the application.  You want to do as little customization as possible if you are using galaxy roles
3. How well is it maintained?  How often is it updated?
