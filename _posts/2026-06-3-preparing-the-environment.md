---
layout: post
title: "Preparing K8s Lab Environment"
date: 2026-06-3
categories: [k8s]
---

## Introduction

We use VirtualBox and Vagrant for preparing our K8s lab environment.

We will have 3 virtual machines with **Ubuntu Server 22.04** for each node. One for control-plane node and two for worker nodes.

```bash
controlplane
node01
node02
```

Instead of running each virtual machine manually, we will use **vagrantfile** to automate our virtual machines.

Download the **lab-setup** folder from the Github repository.

```bash
https://github.com/techiescamp/cka-certification-guide/tree/main/lab-setup
```

Enter the folder with your operating system name, in my case, I will enter the **windows** folder.

Together with other necessary setup files, you will find **Vagrantfile** and **settings.yaml** files that help us run our virtual machines on virtualbox.

```bash
# Vargrantfile

require 'yaml'
settings = YAML.load_file(File.join(File.dirname(__FILE__), 'settings.yaml'))

Vagrant.configure("2") do |config|
  config.vm.box = settings['box_name']
  config.vm.box_check_update = true
  # config.ssh.private_key_path = "/Users/apple/.ssh/id_rsa"

  settings['vm'].each do |vm_config|
    config.vm.define vm_config['name'] do |vm|
      vm.vm.hostname = vm_config['name']
      vm.vm.network "private_network", ip: vm_config['ip']
      vm.vm.synced_folder ".", "/vagrant"

      vm.vm.provider "virtualbox" do |vb|
        vb.memory = vm_config['memory']
        vb.cpus = vm_config['cpus']
        vb.gui = false
      end

      vm.vm.provision "shell", inline: <<-SHELL
        apt-get update -y
        echo "192.168.201.10 controlplane" >> /etc/hosts
        echo "192.168.201.11 node01" >> /etc/hosts
        echo "192.168.201.12 node02" >> /etc/hosts
      SHELL
    end
  end
end
```

```bash
# settings.yaml

# settings.yaml
box_name: "bento/ubuntu-22.04"
vm:
- name: "controlplane"
  ip: "192.168.201.10"
  memory: "2048"
  cpus: "2"
- name: "node01"
  ip: "192.168.201.11"
  memory: "2048"
  cpus: "1"
- name: "node02"
  ip: "192.168.201.12"
  memory: "2048"
  cpus: "1"
```

As you can see in the settings.yaml file, vagrant will download and run 3 coppies of Ubuntu version **bentu/ubuntu-22.04** on virtual box. The required resource for each machine is defined in the settings.yaml file. You may change the parameters according to your needs.

## Extra Configurations for Smooth Running of Virtual Machines

You have to configure the following in order to run virtual machiens smoothly with your operating system:

- Make sure, **VirtualBox Host-only Ethernet Adapter** is configured: Go to **File → Tools → Network → Host-only Networks**, make sure IPv4 address maches the IPs assigned to each virtual machine in the **settings.yaml** file. Also, disable the DHCP Server for the Network Adapter. Once Ethernet Adapter is configured, you may confirm it from **Control Panel → Network & Internet → Network Connections**. This allows you to **ping** your virtual machine from your Windows terminal.

- **Enable Virtualization in BIOS/UEFI**: Enter BIOS/UEFI setup and enable virtualization.
  - Intel Virtualization Technology (VT‑x)
  - AMD‑V (if you have AMD CPU)

- Check Windows Features: Open Control Panel → Programs → Turn Windows features on or off.
  - Make sure Hyper‑V is disabled (VirtualBox conflicts with Hyper‑V).
  - Also disable Windows Hypervisor Platform and Virtual Machine Platform.

Once all configurations are done, opne your windows terminal and go to **lab-setup → windows** folder. Run **vagrant up** command. This will download and install specified virtual machines in your virtualbox.

Once completed, your check your nodes by **vagrant status** command.

```bash
controlplane              running (virtualbox)
node01                    running (virtualbox)
node02                    running (virtualbox)
```

Now that your virtual machines are up and running, you may ssh into each virtual machine from your windows terminal.

```bash
vagrant ssh node01
```

---
