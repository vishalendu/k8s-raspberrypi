Setting up K8s cluster on a Raspberry Pi 4
============
----

### **Objective**
In this document, I am trying to document my experience of setting up a 2 node K8s cluster on a single Raspberry Pi 4. Previous to this I had setup microk8s and k3s on the same device. Both microk8s and k3s are pretty straight forwards and dont require multiple steps for installation, so I am not going to be covering them here.

### **Declaration**
I have just started on my journey to learn Kubernetes and I am in no way an expert. I am writing this documemt, since I couldnt find a single article/tutorial that covered the setup end to end. It was perhaps too painful to setup on a single device. I am sure you will realize the same below.😉 

If you have any suggestions or any improvements in the steps please contact me on my linkedin profile. I would also like to pose this as a challenge to the Ops folks out there to automate, so that many more people can have a portable lab on their disposal. Lets get to it then..

*(I will try to cover major steps in this document, however its possible that some of the links provided may change or cease to exist.)*

### **Pre-requisites**
1. Raspberry Pi4: suggested configuration: 4 core, 8GB RAM, 64-128GB sdcard
2. Ubuntu 20.04 server image (64bit) for RPi 
   (https://ubuntu.com/download/raspberry-pi) 
3. An sd card reader/writer to flash the RPi OS.
4. A RPi OS Flashing Utility (RPi folks have come up with their own)
    (https://www.raspberrypi.org/software/)
5. Wired mouse, keyboard and a monitor to setup the RPi.

### **Setup Ubuntu on RPi**
>**Note**: Please follow the instructions on the imager download page or search youtube on how to flash the OS. Once the sd card is flashed, pop it in the sd card slot and boot up your RPi.
> If you dont have an ethernet connection to your RPi already, you will need to setup a wifi connection. Here is an article on the configurations: https://bit.ly/38R4hz5

> Please continue to the next section once you are logged into your ubuntu OS and have internet connection. This login will be local as you have not setup ssh to connect remotely.
----
&nbsp;
### **Pre-Requisite Software**
#### > Snap packages: lxd, multipass
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
#### > XFCE4 for basic GUI
#### > VNCServer to be able to view browser on RPi
~~~bash
sudo apt install xfce4 xfce4-goodies
sudo apt install tightvncserver
~~~
> Follow this link to setup VNC Server: https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-vnc-on-ubuntu-18-04

>Note: For Ease of use, please setup a key based login to your RPi. Simple google search should be enough to find articles on this. One suggesting is to use ***"ed25519"*** based key instead of default ***rsa***, since its a good practice. 
>For Mac/Linux users its easy enough but for Windows users, I would suggest installing either WSL2 or atleast git bash to connect to your Pi easily.
---
&nbsp;

### **Creating VMs/Containers using Multipass and LXD**
We will use cloud-init for setting up ssh connectivity from RPi with the created container.

### Steps:
#### 1. Create a ssh key pair on your RPi.
~~~
ssh-keygen -t ed25519 -C "k8s" -f ~/.ssh/k8s
~~~
this will generate two files *"~/.ssh/k8s"* and *"~/.ssh/k8s.pub"*. Copy the text inside k8s.pub file, this will be used in the next step.
#### 2. Create a file names *cloud-init.yaml*
~~~
#cloud-config
ssh_authorized_keys:
    - <Paste the text from your k8s.pub file>

package_update: true
~~~
#### 3. Create two Containers using Multipass:
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
>
>You have to either 
> * login to your Rpi and then ssh to the ***"ci1"/"ci2"*** machine. 
> * Or you can configure your RPi to be a jump host to connect to the Containers directly. You can find more about jump host setup here: https://www.tecmint.com/access-linux-server-using-a-jump-host/
&nbsp;
---
### **Milestone: We are ready to start with K8s installation from this point onwards** 👍
---
&nbsp;
### **Pre-requisites software for Kubernetes**
>**Note**: Please repeat the following setup on both ***ci1/ci2*** containers
#### 1. Docker Installation:
~~~
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
~~~

>Note: change your docker's cgroupdriver to cgroupfs, (*"I have not tried with systemd"*)
Please ensure your *"/etc/docker/daemon.json"* has the following
~~~
{"exec-opts": ["native.cgroupdriver=cgroupfs"]}
~~~
>Note: Restart docker and check if the changes have taken effect, run the following command:
> `docker info | grep cgroupfs`
#### 2. kubectl/kubelet/kubeadm installation:
>Note: The ***ci1/ci2*** containers do not have swap turned on. So you dont need to do anything to turn it off.

~~~
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update
apt-get install -y kubelet kubeadm kubectl
~~~
> **Additional Changes**: Need to update kubelet service configuration. Please add the following configuration in the ***/etc/systemd/system/kubelet.service.d/10-kubeadm.conf*** file:
~~~
Environment=”cgroup-driver=cgroupfs”
~~~
>After the above changes are done run:
>systemctl daemon-reload
>systemctl restart kubelet
>
>Note: After this step, it is very important to check if kubelet service is not throwing any unnecessary errors. One problem that I have found so far is the cgroupfs setting on docker, if it is missing, the kubelet service will not start when you do `"kubeadm init"`
> To validate if there are no unusual errors, please check:
> * systemctl status kubelet
> * dmesg
> * tail -f /var/log/syslog
>
>Check if any of the above have any unusual errors. Ideally, the kubelet service ***will not run*** untill the `kubeadm init` command is triggered. and you will see the service has exited in the logs at this point.


&nbsp;
### **Lets start the engine**
Next steps are now related to configuring and setting up the cluster and nodes.
*I am afraid that I have followed online tutorials from this point onwards. I dont completely understand why these steps are required and what they are doing exactly.*

#### > kubeadm init 
##### (*all commands mentioned below to be run on k8s master node, until mentioned otherwise*)
~~~
sudo kubeadm init --apiserver-advertise-address=<ip-address-of-kmaster> --pod-network-cidr=192.168.0.0/16
~~~
From the ***ci1/ci2*** containers, you have to select one to become the master of k8s cluster. We have to supply the ip of the same in the above command. This command will initiate the kubernetes cluster along with basic services and apis.
>***Important***: Please copy the last line from the previous command's output. This will be required to add nodes to the cluster. The line should look something like the following:
> `kubeadm join 10.126.42.170:6443 --token 9yrkjf.2i6fnd9c1to6arxt 
    --discovery-token-ca-cert-hash sha256:61f6c82c781365fe40c62cd4f233f3c8d67461c40799009588f9c0a40e266899`

After the above command has been executed, please do the following steps as normal user:
~~~
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~
> **Note**: At this point, you should be able to check that the kubernetes cluster is up and running. Try running this command `kubectl get pods -o wide --all-namespaces`

You will notice from the previous command (kubectl get pods), that all the pods are running except the ones named like: ‘kube-dns’. For resolving this we will install a pod network. To install the CALICO pod network, run the following command:
~~~
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
~~~
> Now if you try `kubectl get pods -o wide --all-namespaces` command again, you will see that those 'kube-dns' pods have started. (It might take a few minutes for all of thes pods to come up)

#### > Setup Kubernetes dashboard
~~~
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml
~~~
> If you run `kubectl get pods --all-namespaces`, you should now be able to see a new namespace ***"kubernetes-dashboard"*** would have come up.

#### ~ How to access Kubernetes Dashboard ~
Setting up kubernetes to proxy the dashboard. run the following command on k8s master node:
~~~
kubectl proxy --address='0.0.0.0' --accept-hosts='^*$'
~~~
On your RaspberryPi machine (after doing ssh), you can do a local port forwarding to look at the Kubernetes Dashboard. The reason why we are doing port forwarding is that Kubernetes dashboard does not enable login for non "localhost" connections.
Example of local port forwarding:
~~~
ssh -L 8001:10.126.42.170:8001 localhost
~~~
Now, if you remember we have setup the VNC Server, we can open the VNC Server and use the browser to open the following URL:
`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`

##### ~ How to login to Kubernetes Dashboard ~
* Creating a service account and attaching it
~~~
kubectl create serviceaccount dashboard-admin-sa
kubectl create clusterrolebinding dashboard-admin-sa  --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa
~~~

* Getting the secret name:
~~~
kubectl get secrets
~~~
> Sample output:
~~~
NAME                             TYPE                                  DATA   AGE
dashboard-admin-sa-token-7tcdw   kubernetes.io/service-account-token   3      46s
default-token-wslbg              kubernetes.io/service-account-token   3      63m
~~~
* Getting the token for the secret name:
~~~
kubectl describe secret dashboard-admin-sa-token-7tcdw
~~~
***This command will return you the token that can be filled on the login page for the Kubernetes Dashboard.***

---
### **Milestone: We have got the Kubernetes cluster up and running with DNS and Kubernetes Dashboard** 👍
---
&nbsp;
### **Adding node to K8s Cluster**
##### (*Following command has to be run on Kubernetes Cluster nodes*)
Do you remember the command we had saved after running `kubeadm init`?
We have to run it now, to add node to the cluster
~~~
# sample command, your command will differ
sudo kubeadm join 10.126.42.170:6443 --token 9yrkjf.2i6fnd9c1to6arxt --discovery-token-ca-cert-hash sha256:61f6c82c781365fe40c62cd4f233f3c8d67461c40799009588f9c0a40e266899
~~~

You can validate that the node has been added to the cluster by:
* Checking the Kubernetes dashboard under nodes
* Running command on master: `kubectl get nodes`

---
### **Milestone: We have completed the 2 node Kubernetes cluster on a single RPi4** 👍
---
## YAY!! Hope everything turned out well

 [[params.social]]
    icon = "linkedin"
    icon_pack = "fa"
    link = "//linkedin.com/in/vishalendupandey"

