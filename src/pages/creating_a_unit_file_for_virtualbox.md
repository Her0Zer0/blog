---
layout: '../layouts/BlogPost.astro'
title: 'Creating a Unit File for VirtualBox VMs'
description: 'Have you ever used VirtualBox to create VMs that need to start in a headless state? I have a home lab where I host various tools for projects'
poster: 
    url: '/blog/assets/tux_with_unit_header.png'
    alt: 'Tux penguin pointing to a unit file'
tags: ['scripting','linux','virtualbox']
published: '02/02/2025'
author: 'Robert Smith'
imageCredit: ''
---

Have you ever used VirtualBox to create VMs that need to start in a headless state? I have a home lab where I host various tools for projects that I want to work on, and I find myself doing this a lot. This usually involves creating a Unit file to start the VMs on the boot of my home server, and today, I'm going to talk about the process a little bit. 

I hope you enjoy it!


## About the Server
I have a home server running Linux Mint with a Ryzen processor and around 64GB of memory. If you have a home server or want to build your home server lab, any Linux distro can work for this use case. Some distros provide more power and features that are more appropriate for certain needs, however, I prefer to use a Debian-based distro. So the rest of this post will follow setting up systemd using the Mint distro. It should also work on Ubuntu or any of those flavors that are available. 

Your home server doesn't have to be a very powerful machine but it will need to support what you want to run on it. If you are just getting started and have an old laptop/desktop lying around that doesn't support Windows very well anymore. You can breathe new life into this machine by installing a Linux distro on it and playing around with it to see if this is a new hobby you are willing to put money into and use. 

To be honest, I didn't do this for a long time in my career but once I did, I wished I had done it sooner. I've been able to test all kinds of products and really expand my knowledge for my career and other hobbies using a home server. 

## Tools Needed
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads?data1=dwndml04&data2=abmurlvbV2) - Virtualization software that allows you to create instances of machines with various os installed. 
- [VirtualBox Extension Pack](https://www.virtualbox.org/wiki/Downloads?data1=dwndml04&data2=abmurlvbV2) - Command line tools that allow you to manipulate the VMs. 
- Terminal - Any terminal application will work for this and should be installed already on your distro, the important piece is just being comfortable with issuing commands in a Linux shell. Commands we will be using mostly will be [_systemctl_](https://www.man7.org/linux/man-pages/man1/systemctl.1.html) and [_vboxmanage_](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/vboxmanage-cmd-overview.html).

> **Note**: It is important to get the same version of the extension pack that you have for your VirtualBox installation. Otherwise, they will not play well together. 

## The Virtual Machine
The VM we'll be working with today has Ubuntu 20.04 with 1 CPU and 4GB of memory. The application installed on the OS layer is the SQL Server that I am currently using for personal development on a project for the family. When I start up my home server, of course, I want this to start up as well in the background so my application that I am building will communicate with it as needed. With that said, let's get started. 

## Creating a Custom systemd Unit File
If you are not familiar with systemd, here is a very simplified overview. 
- systemd is a system and service manager for Linux operating systems that manage system processes and services. This allows for unified service management for administrators to start, stop, reload, and check the status of services using simple commands like [systemctl](https://www.man7.org/linux/man-pages/man1/systemctl.1.html).

We are going to use systemctl to enable to VM to start automatically in this post. 

### Permissions
When you installed VirtualBox, there should be a user group that came with the installation named _vboxusers_.
To view these users type groups in the terminal: 
```sh
groups
# output: <your_logged_in_user> adm cdrom sudo dip plugdev kvm lpadmin sambashare vboxusers
```

The output may be different on your machine depending on what you have installed, however, the vboxusers groups should be visible. We are going to add the user group to our user. Type the below in the terminal: 
```sh
sudo usermod -a -G vboxusers $USER
```

### Creating the Unit File
Now that we are a part of the cool kid's group (vboxusers) we can create the systemd template file. Feel free to use any editor you like. Some users prefer to use _vi_ but I just use _nano_ on most days. The directory location is the important piece here. 

```sh
    sudo nano /etc/systemd/system/start_vm@.service
```

With the file open in your terminal, add the following configurations. 
```sh
[Unit]
Description= Guest VM %I
After=network.target vboxdrv.service
Before=runlevel2.target shutdown.target

[Service]
User=host_username
Group=vboxusers
Type=forking
TimeoutSec=5min
KillMode=process
GuessMainPID=no
RemainAfterExit=yes
ExecStart=/usr/bin/VBoxManage startvm %i --type headless
ExecStop=/usr/bin/VBoxManage controlvm %i acpipowerbutton

[Install]
WantedBy=multi-user.target
```

Wow, wait a minute!... What is all of this? 

Well, let's talk about it a bit. 

Anything wrapped in angle brackets ([SectionNameHere]), represents a section in the file. Then each line contains predefined words that contain specific instructions to be read at boot. To learn more about Unit files please [check out this handy link](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html). 

Section: Unit
- Description= Guest VM %I 
    - Describes the service. The %I, in this case, is a placeholder that will be replaced with the instance name when the unit is instantiated. 
- After=network.target vboxdrv.service 
    - Specifies that this service should start after the _network.target_ and _vboxdrv.service_. This ensures that the network and VirtualBox driver are up and running before this service is started. 
- Before=runlevel2.target shutdown.target
    - Ensures that this service starts before reaching runlevel 2 and before the system starts shutting down. This helps in correctly ordering the start and stop services. 

Section: Service
- User=host_username 
    - Specifies the user under which the service will run. Replace _host_username_ with the actual username. 
- Group=vboxusers
    - Specifies the group under which the service will run. _vboxusers_ is typically the group that has permission to use VirtualBox. 
- Type=forking
    - Indicates that the service's process will fork into a background process. This is suitable for services that daemonize themselves. 
- TimeoutSec=5min
    - Sets a timeout of 5 minutes for the service to start or stop. If it isn't complete within this time, it will be considered a failure. 
- KillMode=process
    - Specifies how to kill the service. _process_ means only the main process of the service will be killed. 
- GuessMainPID=no
    - This tells the systemd not to guess the main process ID of the service. 
- RemainAfterExit=yes
    - Ensures the service is considered active even after the main process exits. This is useful for services that manage external processes like virtual machines. 
- ExecStart=/usr/bin/VBoxManage startvm %i --type headless
    - Specifies the command to start the virtual machine. The %i is a placeholder for the instance name, and _--type headless_ starts the VM without a graphical interface. 
- ExecStop=/usr/bin/VBoxManage controlvm %i acpipowerbutton
    - Specifies the command to stop the virtual machine gracefully by sending an ACPI power button event. Much like it is done using the graphical user interface. 

Section: Install
- WantedBy=multi-user.target
    - This ensures that this service is started at the multi-user runlevel, making it part of the default system startup. 

Again, a good resource for understanding a Unit file is in the reference link previously. Very neat stuff!

### Review Available VMs and Enable the Service
To view a list of available VMs we can use _vboxmanage_. 

```sh
vboxmanage list vms
# My vm output: "DBServer" {55f6b3b8-4919-40bc-9ceb-0f411d808508}
```

In my case, my VM is named DBServer. This will be what is used when enabling and starting the service. That command looks something like the one below. 

```sh
sudo systemctl enable start_vm@DBServer
# output: Created symlink /etc/systemd/system/multi-user.target.wants/start_vm@DBServer.service → /etc/systemd/system/start_vm@.service
```

Now we can check the status of the running service. 
```sh
sudo systemctl status start_vm@DBServer
# output: 
# start_vm@DBServer.service - Guest VM DBServer
# Loaded: located (/etc/systemd/system/start_vm@.service; enabled; preset)
# Active: inactive (dead)
```

We need to confirm the configuration file is working properly, so we should run the below commands to reload the daemon services and start our service again. Then we'll check the status.

```sh
sudo systemctl daemon-reload
sudo systemctl start start_vm@DBServer
sudo systemctl status start_vm@DBServer
```
The start option activates the unit file and status checks the runtime state of the unit file. We should see the file is now in an active state. To further test everything is going as planned we should restart the server at this time and confirm our VM is running. 

To restart from the terminal you can use: 
```sh
sudo shutdown -r now
```
Once the server is back up and running. Your VM should be running as expected. 

## Summary: 
If you're running a home lab with VirtualBox and need your virtual machines (VMs) to start in a headless state, creating a systemd unit file is a powerful way to automate this process on a Linux server. This post explains how to set up a unit file on a Debian-based distro like Linux Mint, ensuring VMs start automatically at boot. It covers adding your user to the vboxusers group, configuring the unit file with sections detailing service description, dependencies, user and group settings, commands to start and stop the VM, and enabling the service using systemctl. By following these steps, you can efficiently manage your VMs, making your home server setup both reliable and powerful.

### References: 
Ahao, N., & Ahao, N. (2024, July 31). Automatically start VirtualBox Virtual Machines on Boot | Baeldung on Linux. Baeldung on Linux. https://www.baeldung.com/linux/virtualbox-vm-start-on-boot

Downloads – Oracle VirtualBox. (n.d.). https://www.virtualbox.org/wiki/Downloads?data1=dwndml04&data2=abmurlvbV2

systemctl(1) - Linux manual page. (n.d.). https://www.man7.org/linux/man-pages/man1/systemctl.1.html

Oracle VM VirtualBox User Manual. (2020, February 4). https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/vboxmanage-cmd-overview.html

Systemd.unit. (n.d.). https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html