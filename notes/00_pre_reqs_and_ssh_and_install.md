# Mastering Ansible Course

## Pre-reqs:

- 4 Hosts running ubuntu trusty (Controller, LB, Web, DB)
- Setup SSH between the control machine and other hosts
- Ansible user has sudo access w/o password (add ansible user to sudoers file)
- Python 2.7 installed on hosts.  If you are running Xenial this is a problem.  You can install manually or run somthing like this from the controller once everything is setup:

`ansible --sudo -m raw -a "apt-get install -y python2.7 python-simplejson" group_name`

Setup Notes:

1.  launch instance in terraform
2.  login to host and add personal ssh key to authorized keys (optional, but easier to login)
3.  login to host and gen new ssh key and add to authorized keys
4.  add control machine to known hosts
5.  in aws console launch 3 more instances like this one and make sure to use demo.pem

## Current environment

Controller: ec2-54-209-216-247.compute-1.amazonaws.com / ip-172-31-58-141.ec2.internal / 172.31.58.141  
LB: ec2-34-234-204-72.compute-1.amazonaws.com / ip-172-31-58-41.ec2.internal / 172.31.58.141  
APP: ec2-54-174-32-42.compute-1.amazonaws.com / ip-172-31-55-152.ec2.internal / 172.31.55.152  
DB: ec2-54-86-84-122.compute-1.amazonaws.com / ip-172-31-60-171.ec2.internal / 172.31.60.171  

Login to each with pem and then add authorized key

ssh -i ~/.ssh/demo.pem -l ubuntu ec2-54-209-216-247.compute-1.amazonaws.com  
ssh -i ~/.ssh/demo.pem -l ubuntu ec2-34-234-204-72.compute-1.amazonaws.com  
ssh -i ~/.ssh/demo.pem -l ubuntu ec2-54-174-32-42.compute-1.amazonaws.com  
ssh -i ~/.ssh/demo.pem -l ubuntu ec2-54-86-84-122.compute-1.amazonaws.com

Update hosts file on controller with names / ips of other servers

127.0.0.1 localhost  
127.0.0.1 controller  
172.31.58.41 lb  
172.31.55.152 app01  
172.31.61.15 app02  
172.31.53.134 db01

Create new security group allowing ssh, http, and icmp all from source < this-security-group-id \>

Add all hosts to this security group

## SSH

`ssh-keygen -t rsa`

Setting a passphrase locks the private key so someone needs a password to use it, if they gain access to it.

`ssh hostname`

Need to add ssh key to ssh hosts file on current host specifying which key to use for which host, etc.

`ssh -i path/to/private_key.rsa username@hostname.com`

Specify path to private key and user.  default key is ~/.ssh/id_rsa

By default ansible will use the default key.  but you can pass specific keys for specific hosts as well.

> authorized_keys - has a public ssh key on each line.  when a user attempts to ssh in, it checks the private key against each of those keys, and if there is a match, login succeeds.  you will need to add the public key from the controller to the authorized_keys file on each of the other nodes.

## Installation

Login to the controller host and follow these:

<http://docs.ansible.com/ansible/latest/intro_installation.html#latest-releases-via-apt-ubuntu>
