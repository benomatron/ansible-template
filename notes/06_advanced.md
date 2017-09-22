# Optimizing Ansible

## Getting duration of playbooks

Execute your playbooks using **time** to get the duration.

```
time ansible-playbook playbooks/site.yml
*************
real	0m19.676s
user	0m4.220s
sys	0m1.328s
```

```
time ansible-playbook playbooks/stack-restart.yml
*************
real	0m9.191s
user	0m1.744s
sys	0m0.512s
```

```
time ansible-playbook playbooks/stack-status.yml
*************
real	0m5.339s
user	0m2.076s
sys	0m0.532s
```

## Remove unnecessary steps

If you notice the first step of a playbook is always gathering facts.  In many cases, you will not use these facts.

Look for where don't need facts and don't collect them.  For example:

```
---
- hosts: loadbalancer
  become: true
  gather_facts: false
  tasks:
    - name: verify nginx service
      command: service nginx status
```

Then run **time** again to see the improvement.

## Removing unnecessary updates

In the case of **apt** and using update_cache, we are waiting for apt update to run even though we know we won't be doing an upgrade.

You can use **cache_valid_time** skip cache updates if an update has been run within x seconds.

roles/apache2/tasks/main.yml:

```
---
- name: install web components
  apt: name={{item}} state=present update_cache=yes cache_valid_time=86400
  with_items:
    - apache2
    - libapache2-mod-wsgi
```

You can also run the cache update into one step instead of running it one at a time.

site.yml:

```
---
- hosts: all
  become: true
  gather_facts: false
  tasks:
    - name: update apt cache
      apt: update_cache=yes cache_valid_time=86400
- include: control.yml
- include: database.yml
- include: webserver.yml
- include: loadbalancer.yml
```

You can then remove update cache from our other playbooks.

## Limit scope to a host or group of hosts

`ansible-playbook playbooks/site.yml --limit app01`

This will only run the playbook(s) on the app01 host.

## Limit execution by tags

Apply tags to tasks.

roles/nginx/tasks/main.yml

```
---
- name: install tools
  apt: name={{ item }} state=present
  with_items:
    - curl
    - python-httplib2
  tags: [ 'packages' ]

- name: install nginx
  apt: name=nginx state=present
```

Now you can retrieve the tags with:

```
ansible-playbook playbooks/site.yml --list-tags

playbook: playbooks/site.yml

  play #1 (all): all	TAGS: []
      TASK TAGS: []

  play #2 (control): control	TAGS: []
      TASK TAGS: []

  play #3 (database): database	TAGS: []
      TASK TAGS: []

  play #4 (webserver): webserver	TAGS: []
      TASK TAGS: []

  play #5 (loadbalancer): loadbalancer	TAGS: []
      TASK TAGS: [packages]
```

You can then execute filtering by tags.

`ansible-playbook playbooks/site.yml --tags "packages"`

And also skip tags:

`ansible-playbook playbooks/site.yml --skip-tags "packages"`


## Not reporting changes

You can skip the 'changed' status when running playbooks and only report on pass/fail using **changed_when**.
This is helpful for when using **command** that pretty much always reports a change.

playbooks/stack-status.yml

```
---
- hosts: loadbalancer
  become: true
  tasks:
    - name: verify nginx service
      command: service nginx status
      changed_when: false

    - name: verify nginx is listening on 80
      wait_for: port=80 timeout=1

- hosts: webserver
  become: true
  tasks:
    - name: verify apache2 service
      command: service apache2 status
      changed_when: false

    - name: verify apache is listening on 80
      wait
```

It will now only report if the step passed or failed.  None of this change nonsense.

You can also have change reported only when some conditions are met.

roles/nginx/tasks/main.yml

```
---
- name: install tools
  apt: name={{ item }} state=present
  with_items:
    - curl
    - python-httplib2
  tags: [ 'packages' ]

- name: install nginx
  apt: name=nginx state=present

- name: configure nginx site
  template: src=nginx.conf.j2 dest=/etc/nginx/sites-available/{{ item.key }} mode=0644
  with_dict: "{{ sites }}"
  notify: restart nginx

- name: get active sites
  shell: ls -1 /etc/nginx/sites-enabled
  register: active
  changed_when: "active.stdout_lines != sites.keys()"
```

You will get an 'ok' instead of a 'changed' if this condition is not met.

You can also use **failed_when** to customize when a playbook reports failures using the same method.

## Accelerated Mode

Just be aware that accelerated mode and pipelining exists.  If you use this in a big enough environment check it out.

http://docs.ansible.com/ansible/latest/playbooks_acceleration.html
