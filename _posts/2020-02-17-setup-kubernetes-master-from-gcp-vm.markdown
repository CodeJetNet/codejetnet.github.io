---
layout: post
title:  "Setup Kubernetes Master node on GCP VM"
date:   2020-02-17 10:41:05 -0700
categories: kubernetes
---
## Why not use Google Kubernetes Engine? 
Since Google Cloud Platform (GCP) offers the Google Kubernetes Engine (GKE) then why would we want to build a kubernetes master
node from a virtual machine? 

1. To know how to do it and have a better understanding of Kubernetes. (This is my motivation)  
2. To run a different (read:newer) version of Kubernetes than what GKE offers.  Currently GKE offers versions 1.13.12 through 1.15.9 yet the Ubuntu xenial package repo makes 1.17.3 available.

Lets get in to it. 

<figure class="video_container">
  <iframe width="740" height="435" src="https://www.youtube.com/embed/JNIuhYtz4P0" frameborder="0" allowfullscreen="true"> </iframe>
</figure>

## Create an Ubuntu 16.04 Instance
We're going to use Ubuntu 16.04 because it's concrete and the Kubernetes xenial apt repo has all the tools we need readily available.

Set the instance name to something relevant because we will be referencing it in our configurations.  In this example
I will use `master-1`.

On the GCP dashboard navigate to Compute Engine > VM Instances and create a new VM instance.  I choose machine type
n1-standard-2 as it meets the minimum requirements for a Kubernetes master node. 

Change the boot disk to Ubuntu 16.04 LTS and create the instance.

Alternatively, if you have the gcloud SDK command line tools installed you can accomplish this with the following: 
```bash
gcloud beta compute instances create master-1 --zone=us-central1-a --machine-type=n1-standard-2 --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image=ubuntu-1604-xenial-v20200129 --image-project=ubuntu-os-cloud
```

After running the above command the output will provide you with the external IP address of the newly created VM.  
SSH into the box and we'll continue from there. I tend to prefer to sudo everything but it looks cleaner to write out the 
without prefixing sudo to everything so if you are going to copy and paste your way to success following this guide
then you'll want to first run `sudo su` or `sudo -i`.  Everything else in this guide is executed as root inside the 
virtual machine instance we created above unless otherwise noted.

## Setup Apt & Install the Kubernetes tools
First, we'll want to add the Kubernetes xenial repo and add the GPG key for the packages.
```bash
cat <<EOF> /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```

Update the package list upgrade our VM.
```bash
apt-get update
apt-get upgrade -y
```

Install the Kubernete's tools
```bash
apt-get install docker.io kubeadm kubelet kubectl -y
```

## Configure kubeadm
When creating a kubeadm-config.yaml file there here are two important items to note.  

1. The `controlPlaneEndpoint` needs to reference your virtual machine's name followed by port 6443. 
You can setup a host configuration for this or use the instance name that you configured when creating
 the instance as we have done below. `master-1:6443`
2. the `podSubnet` is the address range you want to use for your pods. 

Create the kubeadm-config.yaml file.

```bash
cat <<EOF> kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "master-1:6443"
networking:
  podSubnet: 192.168.0.0/16
EOF
```

Now we can initialize our master node.

```bash
kubeadm init --config=kubeadm-config.yaml --upload-certs
```

When kubeadm is done initializing your master node it will spit out all the instructions we're about to regurgitate below.
At this point you're pretty much done.  All we need to do is setup our user's ability to access the node using kubectl 
and then setup a pod network.
 
## Configure kube config for kubectl
You will want to perform this action as your default user. This will allow you to use kubectl to interact with your kubernetes instance.
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

After executing the above commands you will be able to interface your kubernetes node and execute commands with kubectl. 
```bash
kubectl get node
``` 

## Install a pod network
We need a pod network for our container networking interface and for no particular reason we're choosing [Project Calico](https://docs.projectcalico.org/introduction/).
You should perform this action as your default (non root) user. 

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```

Here you will need to reference your podSubnet from the configure kubeadm step above.  If you went with the same defaults
I did `podSubnet: 192.168.0.0/16` then we don't need to modify anything but if you have any other address range then
we'll need to modify the calico.yaml file and make some changes.  Edit the calico.yaml file we downloaded, find the 
reference to `CALICO_IPV4POOL_CIDR` and update the value to reflect your configured address range. During the time of 
this writing the configuration item could be found on line 630 of calico.yaml.

```bash
vim calico.yaml
```

```yaml
- name: CALICO_IPV4POOL_CIDR
  value: "192.168.0.0/16"
```

Once you have configured the correct pod network pool you may apply the calico configuration file.
 
```bash
kubectl apply -f calico.yaml
```

## Next Steps
That's it, you're done!  Now you can move on to creating and adding additional kubernetes nodes.   
Once you get comfortable with this super easy setup it takes hardly takes a two minutes to get a kubernetes 
cluster up and running.
