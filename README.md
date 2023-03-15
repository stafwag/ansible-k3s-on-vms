# Ansible-k3s-on-vms

An ansible playbook to deploy virtual machines and deploy [K3s](https://k3s.io/).

This playbook is a wrapper around the roles:

* [https://github.com/stafwag/ansible-role-delegated_vm_install](https://github.com/stafwag/ansible-role-delegated_vm_install)

To set up the virtual machines.

* [https://github.com/PyratLabs/ansible-role-k3s](https://github.com/PyratLabs/ansible-role-k3s)

To install and configure K3s on the virtual machines.

* [https://github.com/stafwag/ansible-role-libvirt](https://github.com/stafwag/ansible-role-libvirt)

To enable libvirt on the ```vm_kvm_host```.


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
