Setting up K8s cluster on a Raspberry Pi 4
============
----

**Objective**
In this document, I am trying to document my experience of setting up a 2 node K8s cluster on a single Raspberry Pi 4. Previous to this I had setup microk8s and k3s on the same device. Both microk8s and k3s are pretty straight forwards and dont require multiple steps for installation, so I am not going to be covering them here.

**Declaration**
I have just started on my journey to learn Kubernetes and I am in no way an expert. I am writing this documemt, since I couldnt find a single article/tutorial that covered the setup end to end. It was perhaps too painful to setup on a single device. I am sure you will realize the same below.😉 

If you have any suggestions or any improvements in the steps please contact me on my linkedin profile. I would also like to pose this as a challenge to the Ops folks out there to automate, so that many more people can have a portable lab on their disposal. Lets get to it then..

*(I will try to cover major steps in this document, however its possible that some of the links provided may change or cease to exist.)*

**Pre-requisites**
1. Raspberry Pi4: suggested configuration: 4 core, 8GB RAM, 64-128GB sdcard
2. Ubuntu 20.04 server image (64bit) for RPi 
   (https://ubuntu.com/download/raspberry-pi) 
3. An sd card reader/writer to flash the RPi OS.
4. A RPi OS Flashing Utility (RPi folks have come up with their own)
    (https://www.raspberrypi.org/software/)
5. Wired mouse, keyboard and a monitor to setup the RPi.

**Setup Ubuntu on RPi**
>**Note**: Please follow the instructions on the imager download page or search youtube on how to flash the OS. Once the sd card is flashed, pop it in the sd card slot and boot up your RPi.

> Please continue to the next section once you are logged into your ubuntu OS. This login will be local as you have not setup ssh to connect remotely.
----
&nbsp;
**Pre-Requisite Software**
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

>Note: For Ease of use, please setup a key based login to your RPi. Simple google search should be enough to find articles on this. One suggesting is to use ***"ed25519"*** based key instead of default ***rsa***, since its a good practice. 
>For Mac/Linux users its easy enough but for Windows users, I would suggest installing either WSL2 or atleast git bash to connect to your Pi easily.
---
&nbsp;

**Creating VMs/Containers using Multipass and LXD**
We will use cloud-init for setting up ssh connectivity from RPi with the created container.

Steps:
1. Create a ssh key pair on your RPi.
~~~
ssh-keygen -t ed25519 -C "k8s" -f ~/.ssh/k8s
~~~
this will generate two files *"~/.ssh/k8s"* and *"~/.ssh/k8s.pub"*. Copy the text inside k8s.pub file, this will be used in the next step.
2. Create a file names *cloud-init.yaml*
~~~
#cloud-config
ssh_authorized_keys:
    - <Paste the text from your k8s.pub file>

package_update: true
~~~
3. Create two Containers using Multipass:
~~~
multipass launch --cpus 2 --mem 2048M --disk 10G --name ci1 --cloud-init ./cloud-init.yaml
multipass launch --cpus 1 --mem 2048M --disk 10G --name ci2 --cloud-init ./cloud-init.yaml
~~~

>**Note**: At this point, you should have 2 working LXD containers. You can check that they are running using *"multipass ls"* command. Sample output below:
~~~
Name                    State             IPv4             Image
ci1                     Running           10.126.42.170    Ubuntu 20.04 LTS
ci2                     Running           10.126.42.171    Ubuntu 20.04 LTS
~~~
>***Note***: The 10.x.x.x IPs are local to your RPi so you will not be able to connect to them directly from your laptop.
>You have to either 
> * login to your Rpi and then ssh to the ***"ci1"/"ci2"*** machine. 
> * Or you can configure your RPi to be a jump host to connect to the Containers directly. You can find more about jump host setup here: https://www.tecmint.com/access-linux-server-using-a-jump-host/
---










