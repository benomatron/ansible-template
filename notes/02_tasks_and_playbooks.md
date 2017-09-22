# Tasks

In Ansible, a Task is the basic building block of all execution.  It is divided into to parts, the Module and the Arguments.

Here are some basic ad hoc tasks (commands):

```
ansible -m ping all
ansible -m command -a "hostname" all  # command is the default module so this is the same as:
ansible -a "hostname" all
ansible -a "/bin/false" # will return failed bc /bin/false always returns non-zero exit code
```

# Plays

Plays are ways to execute multiple tasks against the same targets.

Create a new playbook directory in your repo and a new playbook:

```
mkdir playbooks
vim playbooks/hostname.yaml
```

Include the following yaml:

```
---
  - hosts: all
    tasks:
      - name: get server hostname
      - command: hostname
```

You can then execute the playbook with the following:

`ansible-playbook playbooks/hostname.yaml`

Your output will look something like this:

```
PLAY [all] ***********************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************
ok: [control01]
ok: [app01]
ok: [db01]
ok: [app02]
ok: [lb01]

TASK [command get server hostname] *******************************************************************************************************************************************************************
changed: [db01]
changed: [app02]
changed: [app01]
changed: [control01]
changed: [lb01]

PLAY RECAP ***********************************************************************************************************************************************************************
app01                      : ok=2    changed=1    unreachable=0    failed=0
app02                      : ok=2    changed=1    unreachable=0    failed=0
control01                  : ok=2    changed=1    unreachable=0    failed=0
db01                       : ok=2    changed=1    unreachable=0    failed=0
lb01                       : ok=2    changed=1    unreachable=0    failed=0
```

**Important to note**: The status of the hosts appears as 'changed' even though we didn't change anything.  With some modules, for example 'command', ansible does not know whether or not they changed the host.  Ansible only sees that something was run and it did not return an error, so ansible considers that successful and there could have been a change.  Other modules can detect whether something was changed or not.  You need to be aware of this when looking at idempotence when running playbooks with ansible.
