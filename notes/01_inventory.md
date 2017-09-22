# inventory

With Ansible, you need to specify potential targets in advance.  This can be done through an inventory.

The inventory can take a few forms:

1.  Static file
2.  Dynamic (usually a script that calls and API or service to populate a list)

An inventory can list the hosts and additional information about that host.  Usually you specify any special connection information here.

Inventory allows you to group hosts by role, which allows you to deploy the same configuration to multiple hosts.

We are going to use the static file for this course.  By default it lives in:

`/etc/ansible/hosts`

To list the current inventory:

`ansible --list-hosts all`

But we want to create a local inventory file and keep that in your git repo.  Create a git repo for ansible config then create a dev file and add:

```
[loadbalancer]
lb01

[webserver]
app01
app02

[database]
db01

[control]
control01 ansible_connection=local
```

Now you can access these hosts using:

`ansible -i dev --list-hosts all`

You will need to use -i every time to specify the config file unless you override the ansible config file.  By default it is at /etc/ansible/ansible.cfg.

Create an ansible.cfg file in the root of our repo and add:

```
[defaults]
inventory = ./dev
```

Now, as long as you execute ansbile from this directory, this will override the defaults from /etc/ansible/ansible.cfg.

### Selecting Inventory

```
ansible --list-hosts all
ansible --list-hosts "*"  # quote * b/c we are running in bash
ansible --list-hosts webserver  # use group
ansible --list-hosts app02  # or individual hostname
ansible --list-hosts app*  # or wildcard hostnames
ansible --list-hosts app*,database  # or multiple hostnames or groups joined with a comma (or colon in previous versions)
ansible --list-hosts webserver[0]  # specific hosts in a group by index
ansible --list-hosts \!webserver  # exclude a group (select everything not in webserver)
```

Full list of patterns here: <http://docs.ansible.com/ansible/latest/intro_patterns.html>
