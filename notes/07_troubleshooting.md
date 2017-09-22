# Troubleshooting


## Ordering problems

Sometimes you will code yourself into a corner and get screwed by ordering or dependency issues.

For example, we initially put a service started check before all the necessary changes have been made.

Here we have mysql starting before we have made changes to the config.

```
---
- name: install tools
  apt: name={{item}} state=present
  with_items:
    - python-mysqldb

- name: install mysql-server
  apt: name=mysql-server state=present

- name: ensure mysql started
  service: name=mysql state=started enabled=yes

- name: ensure mysql listening on all interfaces
  lineinfile:
    dest: /etc/mysql/my.cnf
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
  with_items:
    - { regexp: '^[mysqld]', line: '[mysqld]' }
    - { regexp: '^bind-address', line: 'bind-address = {{ ansible_eth0.ipv4.address }}' }
  notify: restart mysql
```

This could be bad b/c the config could break mysql and prevent the service from starting.  The next time through the playbook will get stuck at 'ensure mysql started' but can't make it further to potentially correc the change.

You can resolve this by re-ordering.

You can also use **ignore_errors** to allow it to continue.

```
---
- name: install tools
  apt: name={{item}} state=present
  with_items:
    - python-mysqldb

- name: install mysql-server
  apt: name=mysql-server state=present
  ignore_errors: true
```

## Jumping to Specific Tasks

Sometimes you will just want to run one task over and over.  Instead of commenting out most of your playbook you can use some command line switches.

The first is **step**.

`ansbile-playbook playbooks/site.yml --step`

It execute one step at a time and prompt to see if you want to run it.  A little like a debug step through.

You can also pick a place you want to start using **list_tasks** and **start-at-task** to find a starting point.

`ansbile-playbook playbooks/site.yml --list-tasks`

Will give you a list of all tasks.  You can then start at a task using:

`ansbile-playbook playbooks/site.yml --start-at-task "mytask"`


## Retrying failed hosts

If there are a few hosts out of many that are failing.  Ansible tracks these in a **.retry** file.

`ansbile-playbook playbooks/site.yml --limit @/path/to/site.retry`

## Syntax-Check and dry runs

Good precursor tools when running in live environment and can be used as unit tests.

Syntax check just looks at syntax and doesn't look at hosts or anything like that.

`ansible-playbook --syntax-check playbooks/site.yml`

This will parse all of the underlying playbooks called my site.yml.

You can do a dry run using:

`ansible-playbook --check playbooks/site.yml`

This is helpful and much faster but may not be valid for all modules but it will go in and check states on hosts, etc.

It also helps not break things in a live environment.


## Debugging

When you need to get at the data in transient states during playbooks.

For example, you are going to set values by registering 'active':

```
- name: get active sites
  shell: ls -1 /etc/nginx/sites-enabled
  register: active
  changed_when: "active.stdout_lines != sites.keys()"

- debug: var=active.stdout_lines

- name: deactivate sites
  file: path=/etc/nginx/sites-enabled/{{ item }} state=absent
  with_items: "{{ active.stdout_lines }}"
  when: item not in sites
  notify: restart nginx
```

You can use debug to output to the shell by using debug statements.

Or you could see the values for variables that are being used by your playbooks.

```
debug: var=db_name
debug: var=vars
``
