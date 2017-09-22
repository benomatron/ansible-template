# Variables

This includes facts, defaults, and other ways to specify variables.  Their precedence is important.
The closer you get to execution, they higher the precedence.

In 2.x, we have made the order of precedence more specific (with the last listed variables winning prioritization):

role defaults [1]
inventory file or script group vars [2]
inventory group_vars/all
playbook group_vars/all
inventory group_vars/*
playbook group_vars/*
inventory file or script host vars [2]
inventory host_vars/*
playbook host_vars/*
host facts
play vars
play vars_prompt
play vars_files
role vars (defined in role/vars/main.yml)
block vars (only for tasks in block)
task vars (only for the task)
role (and include_role) params
include params
include_vars
set_facts / registered vars
extra vars (always win precedence)


# Facts

Facts are used to get information about a host.

Facts can be gathered using:

`ansible -m setup hostname`

For example, to retreive the eth0 ip address for my db01 host:

```
ansible -m setup db01

db01 | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "172.31.53.134"
        ],
        "ansible_all_ipv6_addresses": [
            "fe80::472:6eff:fecd:85ca"
        ],
        "ansible_apparmor": {
            "status": "enabled"
        },
        "ansible_architecture": "x86_64",
        # LOTS OF OTHER STUFF.....
        "ansible_eth0": {
            "active": true,
            "device": "eth0",
            "features": {
                "busy_poll": "off [fixed]",
                "fcoe_mtu": "off [fixed]",
                "generic_receive_offload": "on",
                "generic_segmentation_offload": "on",
                "highdma": "off [fixed]",
                "hw_tc_offload": "off [fixed]",
                "l2_fwd_offload": "off [fixed]",
                "large_receive_offload": "off [fixed]",
                "loopback": "off [fixed]",
                "netns_local": "off [fixed]",
                "ntuple_filters": "off [fixed]",
                "receive_hashing": "off [fixed]",
                "rx_all": "off [fixed]",
                "rx_checksumming": "on [fixed]",
                "rx_fcs": "off [fixed]",
                "rx_vlan_filter": "off [fixed]",
                "rx_vlan_offload": "off [fixed]",
                "rx_vlan_stag_filter": "off [fixed]",
                "rx_vlan_stag_hw_parse": "off [fixed]",
                "scatter_gather": "on",
                "tcp_segmentation_offload": "on",
                "tx_checksum_fcoe_crc": "off [fixed]",
                "tx_checksum_ip_generic": "off [fixed]",
                "tx_checksum_ipv4": "on [fixed]",
                "tx_checksum_ipv6": "off [requested on]",
                "tx_checksum_sctp": "off [fixed]",
                "tx_checksumming": "on",
                "tx_fcoe_segmentation": "off [fixed]",
                "tx_gre_segmentation": "off [fixed]",
                "tx_gso_robust": "on [fixed]",
                "tx_ipip_segmentation": "off [fixed]",
                "tx_lockless": "off [fixed]",
                "tx_nocache_copy": "off",
                "tx_scatter_gather": "on",
                "tx_scatter_gather_fraglist": "off [fixed]",
                "tx_sit_segmentation": "off [fixed]",
                "tx_tcp6_segmentation": "off [requested on]",
                "tx_tcp_ecn_segmentation": "off [fixed]",
                "tx_tcp_segmentation": "on",
                "tx_udp_tnl_segmentation": "off [fixed]",
                "tx_vlan_offload": "off [fixed]",
                "tx_vlan_stag_hw_insert": "off [fixed]",
                "udp_fragmentation_offload": "off [fixed]",
                "vlan_challenged": "off [fixed]"
            },
            "ipv4": {
                "address": "172.31.53.134",
                "broadcast": "172.31.63.255",
                "netmask": "255.255.240.0",
                "network": "172.31.48.0"
            },
            "ipv6": [
                {
                    "address": "fe80::472:6eff:fecd:85ca",
                    "prefix": "64",
                    "scope": "link"
                }
            ],
            "macaddress": "06:72:6e:cd:85:ca",
            "module": "xen_netfront",
            "mtu": 9001,
            "pciid": "vif-0",
            "promisc": false,
            "type": "ether"
        },
        # LOTS OF OTHER STUFF.....
}
```

Now we can use these **facts** in our yaml files.

```
lineinfile:
  dest: /etc/mysql/my.cnf
  regexp: "{{ item.regexp }}"
  line: "{{ item.line }}"
with_items:
  - { regexp: '^[mysqld]', line: '[mysqld]' }
  - { regexp: '^bind-address', line: 'bind-address = {{ ansible_eth0.ipv4.address }}' }
```

# Defaults

Use jinja2 style to specify variables.

```
- name: create database
  mysql_db: name={{ db_name }} state=present

- name: create demo user
  mysql_user: name={{ db_user_name }} password={{ db_user_pass }} priv={{ db_name}}.*:ALL
              host='{{ db_user_host }}' state=present
```

Specify these values in the roles/mysql/defaults/main.yaml

```
---
db_name: myapp
db_user_name: dbuser
db_user_pass: dbpass
db_user_host: localhost
```

# Vars

Can be injected in many places.  One is in roles/name/vars/main.yaml.  This is a bad idea.
You can also pass them as key:value pairs in many places along the way. For example, in the playbook:

```
---
- hosts: loadbalancer
  vars:
    favcolor: blue
```

The same can be part of **includes** and **roles**.

```
- hosts: database
  become: true
  roles:
    - { role: mysql, db_name: demo, db_user_name: demo, db_user_pass: demo, db_user_host: '%' }
```

Where should we put them?

Pick a scheme and stick to it.  Use a flat hierarchy.  Ansible uses a global scope, so no matter where you put them, they are entering the scope.
Put them in one place or a few very simple places.

### Example:

Using a dict in nginx/defaults/main.yaml and then calling those variables from tasks/main.yaml and templates/nginx.conf.j2.  

roles/nginx/defaults/main.yaml

```
---
sites:
  myapp:
    frontend: 80
    backend: 80
```

roles/nginx/tasks/main.yaml

```
---
- name: install tools
  apt: name={{item}} state=present update_cache=yes
  with_items:
    - curl
    - python-httplib2

- name: configure nginx site
  template: src=nginx.conf.j2 dest=/etc/nginx/sites-available/{{ item.key }} mode=0644
  with_dict: "{{ sites }}"
  notify: restart nginx

- name: activate demo nginx site
  file: src=/etc/nginx/sites-available/{{ item.key }} dest=/etc/nginx/sites-enabled/{{ item.key }} state=link
  with_dict: "{{ sites }}"
  notify: restart nginx
```

roles/nginx/templates/nginx.conf.j2

```
upstream {{ item.key }} {
{% for server in groups.webserver %}
  server {{ server }}:{{ item.value.backend }};
{% endfor %}
}

server {
  listen {{ item.value.frontend }};

  location / {
    proxy_pass http://{{ item.key }};
  }
}
```

It is better to put all your vars in a single place though.  You can do this in:

+ Inventory
+ vars_file
+ group_vars

We are going to use group_vars b/c it can be pulled in automatically by all of the roles.

In ansible root:

```
mkdir group_vars
cd group_vars
touch all
```

> Using **all** means it will be used for all of the groups.  Alternatively, you can make an individual file for each group.

```
---
db_name: demo
db_user: demo
db_pass: demo
```

You can then use variable routing if you want to specify the value of variables used elsewhere if named differently.

```
---
- hosts: database
  become: true
  roles:
    - role: mysql
      db_user_name: "{{ db_user }}"
      db_user_pass: "{{ db_pass }}",
      db_user_host: '%'
```

Or better, just use the same names and only specify them once in the all.yml file.

# Vault

A vault is a way to encrypt a secrets file.  You can then pair the vault with a vars file so you don't lose track of the var names, but at the same time encrypt their values.

```
cd group_vars
mv all vars
mkdir mkdir all
mv vars all/
cd all
ansible-vault create vault
> Vault password: ansible_pass
```

Now you can add key:value passwords to the vault.

```
vault_db_pass: some_pass
```

If you look at the vault it is encrypted:

$ANSIBLE_VAULT;1.1;AES256
33386435376262313664376233656437333532383334613065643332393439653965303436316235
3731643135393238613438623530363736383666653330300a333661343635343239643662653638
62353137626363313566656137383662386463656439383661313830356562366438623736383035
6466663662353432620a363263633166303738383938396665663735626435643434313330336535
30633831316237336266613331396332386666393663363266393338313663303734

You need to use vault to edit.

`ansible-vault edit vault`

Now keep your vars file in plain text but reference the vault for the actual values.

```
---
db_name: demo
db_user: demo
db_pass: "{{ vault_db_pass }}"
```

Now you will need to specify the vault password in one of two ways when executing playbooks.

1. Use `ansible-playbook --ask-vault-pass` which is the worst arg name I have ever seen.
2. Or create a vault password file and put it somewhere safe and lock it down.  We will do this.

```
echo ansible_pass > ~/.vault_pass
chmod 0600 ~/.vault_pass
```

Then add this file to your ansible.cfg.

```
[defaults]
inventory = ./dev
roles_path = ./roles
retry_files_enabled = False
vault_password_file = ~/.vault_pass
```
