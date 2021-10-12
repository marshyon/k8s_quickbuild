# Ansible Kubernetes VM environment

This is an example of how ansible can be used to configure K8s to run on a couple of VMs in a typical 'cloud' environment that could be in AWS, Azure, GCP or locally hosted on say VMWare, Proxmox or similar, even Hyper-V

It is a quick stand up and not intended for use in large infra builds. It does not closely follow ansibles [suggestioed best practice](https://docs.ansible.com/ansible/2.4/playbooks_best_practices.html), rather it uses a simple directory structure and variable definition just to get a quick demo running. Transposing this to the former would not take a lot of effort but would increase the complexity slightly of this project.

Ansible tower is not used here, rather it uses a 'workstation' from which the ansible playbooks are run and then ansible reaches out to its target hosts in order to configure them. The playbooks and scripts that this uses may be used to configure ansible tower to do this work for us in a centralised and coordinated way but this use case will not do that and that is left for another project


Ubuntu 1804 LTS is the target to be configured. Other flavours of Linux operating systems are available.

for the implatient like me the working code for this is to be found at [this repo on Github](https://github.com/marshyon/k8s_quickbuild)

install pre-requistites with 

```
ansible-galaxy install -r requirements.yml --force
```

run it against the vms specified in `hosts` file something like :

```
ansible-playbook -e "ansible_sudo_pass=${ansible_sudo}" site.yml 
```


# Prep Linux hosts for ansible

Ansible for linux typically uses ssh, to communicate and configure hosts. 

Kubernetes runs on Linux and this is at least required for its management plain. This project is not intended for use with Windows worker nodes.

As this use case uses ssh, this needs to be configured to work properly for ansible to work.

## ssh configuration

1. from the host that is used to control the host, ssh keys are generated and stored
2. ssh to the host to be controled by ansible and copy the key to the host
3. test ansible can actually ping the host

### generate ssh keys

if we dont already have an ssh key we want to use for building the infra, we can generate a new one 

**note** that I chose to name the key `build_key` as the name for the new key which does __does not exist__ yet, so I'm creating a new one.

with :

```
$ ssh-keygen -t rsa -f ~/.ssh/build_key
```
and the output looks something like this :
```
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/jon/.ssh/build_key
Your public key has been saved in /home/jon/.ssh/build_key.pub
The key fingerprint is:
SHA256:07KnFoMWUbKE3To/ER4Odgu+19u5qT/qUzJ87v5LcIA jon@LAPTOP-HFGFI09D
The key's randomart image is:
+---[RSA 3072]----+
|     ooo.        |
|    ..*o=  .     |
|     o.O +E .    |
|      = =.   .   |
|       *S+. . .  |
|      + *+= oo   |
|     . ..+.O ..  |
|        .oo =o   |
|       ...+B*+o. |
+----[SHA256]-----+
```

### copy the key to the build hosts

first we load the key we just created (above) with


then copy the key to each host ( and we are using 2 hosts for this example ):

```
$ ssh-copy-id 192.168.88.71
```
```
$ ssh-copy-id 192.168.88.72
```

### test ansible ping

create an ansible config file

>ansible.cfg

```
[defaults]
inventory = hosts
roles_path = roles
```

the section heading of [defaults](https://docs.ansible.com/ansible/2.3/intro_configuration.html#general-defaults) needs to be present for this value of inventory to work

create a local hosts file called `hosts` and update the ip addresses to match those we just copied the keys to (above)

> hosts

```
[k8s]
192.168.88.71
192.168.88.72

[k8s:vars]
ansible_connection=ssh
ansible_user=jon
ansible_python_interpreter=/usr/bin/python3

[k8smaster]
192.168.88.71

[k8smaster:vars]
hostname=cks-master

[k8sworker01]
192.168.88.72

[k8sworker01:vars]
hostname=cks-worker
```
there are several ways to format this hosts file, this is the 'ini' style. Ansible supports other formats, YAML being one of them and one which many people use but ini format for simplicity is used here

notable YAML format can store object data and configuration data within it for groups of hosts and can be usefull in more complicated scenarios.

example `ansible -i hosts k8s -m ping`

where `-i hosts` tells ansible to use this local hosts file and `-m ping` says to use the ping command in ansible to reach out to the host

equaly, the following will also work, so long as a config file (above) has been created in the current working directory


example `ansible k8s -m ping`

the following will also work if you have a bunch of groups in your inventory / hosts file and want to ping all of them

example `ansible all -m ping`

the output will look something like :

```
$ ansible -i hosts k8s -m ping
192.168.88.71 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
192.168.88.72 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

the above example output demonstrates a successfull ping pong - all is good with ansible connectivity for this host so the build with ansible may commence

## configure an env file for later use with ansible to establish sudo authentication

create a `.env` file in the current directory that looks like this, replacing the values for password:

```
ansible_sudo=password
```

and then apply this to the current shell with

```
source .env
```

# create and run a playbook that does something 

Typically, when dealing with newly created vms we find that it is necessary to do some initial configuration common to template driven builds where a server has been created from some kine of template and as such it has default values where these were not known or by error, whatever. So first is to establish for this vm and this project some basic things.

Lets look at hostname, which may or may not be set correctly but we are going to ensure that this is the case.

Ansible can edit text files, such as `/etc/hostnama` and it may be comforting to use ansibles text manipuation to change the contents of this file to be that of the intended hostname but it has a built in function for hostname so lets use that instead

the following snipet has the hostname stanza within it - it wont work on its own and has to be a part of a working playbook but this is what it looks like :

```
    - name: Set hostname
      ansible.builtin.hostname:
        name: "{{ hostname }}"
```

the weird looking **hostname** value is a way Ansible can substitute a value with a variable where this is specified elsewhere and we will get to that later.

to use this without a variable, you could replace the whole value with the name of your host, so it would read on the 3rd line 

`name: mynewhost`

For this to fully work however, it still needs to be a part of a working playbook which might look something like this and within a file called `site.yml` :

> site.yml

```
---
- hosts: all
  become: true
  vars_files:
    - vars/default.yml

  tasks:

    - name: Set hostname
      ansible.builtin.hostname:
        name: "{{ hostname }}"
```

a `vars/default.yml` file would reside in a `vars` directory in the current directory and may look something like this :

> vars/default.yml

```
---
some_value_or_other: 42
```

This is not at all used (yet) but it will be at some point in the future so its here as a placeholder. The `vars_files` section (above) could be missed out but its here as a part of this template as illustration of how we can store configuration values outside of our playbook

This could be used to store our hostname but we wont use this method for now, rather we'll use the inventory file, our local `hosts` file to do that for us so here is an updated version of that file :

> hosts

```
[k8s]
192.168.88.62

[k8s:vars]
ansible_connection=ssh
ansible_user=jon
ansible_python_interpreter=/usr/bin/python3

[k8smaster]
192.168.88.62

[k8smaster:vars]
hostname=k8s-master
```

the above works but how can we add another host and this be configured with the correct hostname ?

it would not work, as the `all` keyword in this playbook would apply the hostname variable to every host and every time ansible runs so the playbook can be re-factored to look something like this :

> hosts

```
---
- hosts: all
  become: true
  vars_files:
    - vars/default.yml
  tasks:
    - name: Install common packages
      apt:
        pkg:
          - curl
          - software-properties-common
          - python3-pip
          - net-tools
          - vim
        state: latest
        update_cache: true
- hosts: k8smaster
  become: true
  vars_files:
    - vars/default.yml
  tasks:
    - name: Set k8s master hostname
      ansible.builtin.hostname:
        name: "{{ hostname }}"
```        
This more complete playbook should now will work and will apply a hostname to just the named host `k8smaster` and we can add hosts to it named worker01 ... worker10 for instance and each will recieve a correct hostname. They will all have common packages added to them.


we run the playbook with 

```
ansible-playbook --limit k8smaster -e "ansible_sudo_pass=${ansible_sudo}" site.yml 
```

# k8s cluster config

```
ansible-galaxy install geerlingguy.swap
```

# References

the following resources were used to buld this script from the killer-sh repo :

```
https://raw.githubusercontent.com/killer-sh/cks-course-environment/master/cluster-setup/latest/install_master.sh
https://raw.githubusercontent.com/killer-sh/cks-course-environment/master/cluster-setup/latest/install_worker.sh
```
