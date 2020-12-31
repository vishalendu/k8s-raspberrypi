Setting up K8s cluster on a Raspberry Pi 4
============
----

**Objective**: 
In this document, I am trying to document my experience of setting up a 2 node K8s cluster on a single Raspberry Pi 4. Previous to this I had setup microk8s and k3s on the same device. Both microk8s and k3s are pretty straight forwards and dont require multiple steps for installation, so I am not going to be covering them here.

**Declaration**:
I have just started on my journey to learn Kubernetes and I am in no way an expert. I am writing this documemt, since I couldnt find a single article/tutorial that covered the setup end to end. It was perhaps too painful to setup on a single device. I am sure you will realize the same below.😉 

If you have any suggestions or any improvements in the steps please contact me on my linkedin profile. I would also like to pose this as a challenge to the Ops folks out there to automate, so that many more people can have a portable lab on their disposal. Lets get to it then..

*(I will try to cover major steps in this document, however its possible that some of the links provided may change or cease to exist.)*

**Pre-requisite**:
1. Raspberry Pi4: suggested configuration: 4 core, 8GB RAM, 64-128GB sdcard
2. Ubuntu 20.04 server image (64bit) for RPi 
   (https://ubuntu.com/download/raspberry-pi) 
3. An sd card reader/writer to flash the RPi OS.
4. A RPi OS Flashing Utility (RPi folks have come up with their own)
    (https://www.raspberrypi.org/software/)
5. Wired mouse, keyboard and a monitor to setup the RPi.

**Setup Ubuntu on RPi**:
>**Note**: Please follow the instructions on the imager download page or search youtube on how to flash the OS. Once the sd card is flashed, pop it in the sd card slot and boot up your RPi.

> Please continue to the next section once you are logged into your ubuntu OS, this login will be local as you have not setup ssh to connect remotely.

&nbsp;
**Additional Software**:
* Snap packages: lxd, multipass
~~~bash
# Good practice to update apt cache and install any updates before we start
sudo apt update && sudo apt dist-upgrade
sudo snap install lxd
sudo lxd init --auto
sudo snap install multipass --candidate
# setup multipass to use lxd
sudo snap connect multipass:lxd lxd
sudo multipass set local.driver=lxd
~~~
-- Following are my personal preferences
* XFCE4 for basic GUI
* VNCServer to be able to view browser on RPi
~~~bash
sudo apt install xfce4 xfce4-goodies
sudo apt install tightvncserver
~~~
> Follow this link to setup VNC Server: https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-vnc-on-ubuntu-18-04





