# Ansible-k3s-on-vms

An ansible playbook to deploy virtual machines and deploy [K3s](https://k3s.io/).

This playbook is a wrapper around the roles:

* [https://github.com/stafwag/ansible-role-delegated_vm_install](https://github.com/stafwag/ansible-role-delegated_vm_install)

To set up the virtual machines.

* [https://github.com/PyratLabs/ansible-role-k3s](https://github.com/PyratLabs/ansible-role-k3s)

To install and configure K3s on the virtual machines.

* [https://github.com/stafwag/ansible-role-libvirt](https://github.com/stafwag/ansible-role-libvirt)

To enable libvirt on the ```vm_kvm_host```.

The sample inventory will install the virtual machines on ```localhost```. It's possible to install the virtual machine on multuple lbvirt/KVM hypervisors


## Requirements

### Supported GNU/Linux Distributions

It should work on most GNU/Linux distributions.
```cloud-localds``` is required. ```cloud-localds``` was available on
Centos/RedHat 7 but not on Redhat 8. You'll need to install it manually
to use this playbook on Centos/RedHat 8.

* Archlinux
* Debian
* Centos 7
* RedHat 7
* Ubuntu

### Ansible

#### Ansible

You use this Ansible playbook, You'll need Ansible install on the Ansible host.
Install the from your GNU/Linux distribution responsitory.

Or install it using ```pip`` see the official Ansible documentation:
[https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

#### Authentication

For easy usage, it's recommended to set up a Ansible account that accessible over ssh without password authentication.
You can create a local ssh key with ```ssh-keygen``` or if you really care about security use a smartcard/hsm with 
a ssh-agent.

###### ssh connection

Install and enable/start the ```sshd``` on the ansible host and the libvirt kvm hypervisor were the virtual machines/K3s will be installed.

####### Local ssh key pair (less security)

To create a local ssh keypair run the ```ssh-keygen``` command.

```
$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/staf/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/staf/.ssh/id_rsa
Your public key has been saved in /home/staf/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:L+Ptozbkj5UEv4yfoRj1werntqkhIF6mSSYeRbtsrn4 staf@manjaro
The key's randomart image is:
+---[RSA 3072]----+
|   .             |
|  . .            |
|   o    .        |
|  o .    +       |
|..+++   S =      |
|.=+* . ..B +     |
| .+.  oo* O      |
|  .E   *+X++     |
|.o.   ..BXXo     |
+----[SHA256]-----+
```

And add the public key to ```~/.ssh/authorized_keys``` on the Ansible host and the libvirt kvm hypervisor hosts.

````
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
````

####### Ssh private key on SmartCard/HSM

To use ssh keypair on a smartcard or HSM you can have a look at the 

* [https://stafwag.github.io/blog/blog/2015/11/21/starting-to-protect-my-private-keys-with-smartcard-hsm/](https://stafwag.github.io/blog/blog/2015/11/21/starting-to-protect-my-private-keys-with-smartcard-hsm/)
* [https://stafwag.github.io/blog/blog/2015/06/16/using-yubikey-neo-as-gpg-smartcard-for-ssh-authentication/](https://stafwag.github.io/blog/blog/2015/06/16/using-yubikey-neo-as-gpg-smartcard-for-ssh-authentication/)

Or another howto setup to store your ssh private key in a secure way.


####### Test the connect + exchange ssh host keys

Login to the anisble host (localhost) and libvirt/kvm hypervisors.
Please note that it's requirement (by default) to also allow ssh connection to install required packages for your GNU/Linux distribution.

```
$ ssh localhost
The authenticity of host 'localhost (::1)' can't be established.
ED25519 key fingerprint is <snip>
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'localhost' (ED25519) to the list of known hosts.
Last login: Sat Mar  4 13:01:47 2023 from 192.168.122.1
$ 
```

###### sudo

For easy usage set up sudo with authentication to gain root access or use the ```--ask-become-pass``` when you run the playbook.

```
<ansible_user> ALL=(ALL:ALL) NOPASSWD: ALL
```

Personality I use sudo authentication and use [ansible](https://docs.ansible.com/ansible/latest/vault_guide/index.html) to store the sudo password.

```
<ansible_user> ALL=(ALL:ALL) ALL
```

### Install the required roles

To install the required roles execute:

```
$ ansible-galaxy install -r requirements.yml
```

### Debian cloud image

This playbook depends on the Debian bulls-eye cloud image. The playbook will update the packages by default,
using the daily image will reduce the update time.

Download the Debian cloud image from

[https://cloud.debian.org/images/cloud/bullseye/daily/latest/](https://cloud.debian.org/images/cloud/bullseye/daily/latest/)

The playbook will use ```~/Downloads/debian-11-generic-amd64-daily.qcow2``` by default.

If you want to use another location update ```boot_disk``` ```src``` 

```
delegated_vm_install:
      vm:
        path: /var/lib/libvirt/images/k3s/
        boot_disk:
          src:
            ~/Downloads/debian-11-generic-amd64-daily.qcow2
```


# Usage

## Inventory

The included ansible.cfg set the inventory path to the ```./etc/inventory/``` directory.

There is a sample inventory included in this repository.

Copy the sample inventory

```
[staf@vicky k3s_on_vms]$ cp -r etc/sample_inventory/ etc/inventory
[staf@vicky k3s_on_vms]$ 
```

Inventory file is located at: ```etc/inventory/k3s```.

The default, parameters will install virtual machines and install k3s on them on the ```localhost``` KVM/libvirt hypervisor.
The IP range is the default subnet - 192.168.122.0/24 - used on the default network on libvirt.
This playbook will install the required libvirt packages and start/enable the default libvirt network.

```
---
k3s_cluster:
  vars:
    ansible_python_interpreter: /usr/bin/python3
    k3s_become: yes
    k3s_etcd_datastore: true
    k3s_use_experimental: true
    k3s_control_node: true
    k3s_flannel_interface: enp1s0
    k3s_server:
      disable:
        - servicelb
        - traefik
    delegated_vm_install:
      post:
        always:
          true
        update_ssh_known_hosts:
          true
      vm:
        path: /var/lib/libvirt/images/k3s/
        boot_disk:
          src:
            ~/Downloads/isos/debian/cloud/amd64/debian-11-generic-amd64-daily.qcow2
          size: 50G
        memory: 4096
        cpus: 2
        dns_nameservers: 9.9.9.9
        gateway: 192.168.122.1
        default_user:
          ssh_authorized_keys:
            - "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
#
# Set the network "default" is used by default
#        network: bridge:my-bridge
        commands:
          - echo "Installed @ $(date)" > /var/tmp/installed
  hosts:
    k3s-master001:
      ansible_host: 192.168.122.20
      vm_ip_address: "{{ ansible_host }}"
      vm_kvm_host: localhost
    k3s-master002:
      ansible_host: 192.168.122.21
      vm_ip_address: "{{ ansible_host }}"
      vm_kvm_host: localhost
    k3s-master003:
      ansible_host: 192.168.122.22
      vm_ip_address: "{{ ansible_host }}"
      vm_kvm_host: localhost
```

## Parameters

We go over some useful parameters.

The full documentation of parameters are documented at the upstream roles:

* [https://github.com/stafwag/ansible-role-delegated_vm_install](https://github.com/stafwag/ansible-role-delegated_vm_install) 
* [https://github.com/PyratLabs/ansible-role-k3s](https://github.com/PyratLabs/ansible-role-k3s)

### Deployment arget

By default the playbook will create virtual machines on the ```localhost``` hypervisor and install k3s on it.
The ```vm_kvm_host``` can be adjusted if you want to deploy the virtual machines on different hypervisor hosts.

### Network

If you want to change the host machine and the ip address of the virtual machine you can update the ```vm_ip_address``` and the ```vm_kvm_host``` parameters.
To use a bridge you can set the

```
delegated_vm_install:
  vm:
    network:
      network: bridge:my-bridge
```

the bridge needs to be already configured on the hypervisor host before running this playbook.

### Cpu & memory

You might want adjust some parameters like to the ```cpus```,  ```memory```,  settings.

### default user

#### username

The playbook will create a default user on the virtual machines - using ```cloud-int```. This default user is set to environment variable ```USER``` is not specified.
If you want to another user you can set ```ansible_user``` as part of the host inventory see:
[https://docs.ansible.com/ansible/latest/inventory_guide/connection_details.html](https://docs.ansible.com/ansible/latest/inventory_guide/connection_details.html) for more details.

#### ssh pub key

The playboot will update ```~/.ssh/authorized_keys``` in the default_user home directory with the ssh public key ````~/.ssh/id_rsa.pub```  by default.
If you want to use another ssh public key you can update the 

## Execute playbook

Make that you have access with ssh and the kvm hypervisor hosts (localhost by default).
If you need to type a sudo password add the ```---ask-become-pass``` argument.

```
$ ansible-playbook --ask-become-pass site.yml
```

***Have fun!***
