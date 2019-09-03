# Installing minikube on AWS Cloud9 #

Incorporating material from:
- an O'Reilly course on K8s
- [Minikube for Kubeflow](https://www.kubeflow.org/docs/other-guides/virtual-dev/getting-started-minikube/) in the Kubeflow tutorial
- RadishLogic's https://www.radishlogic.com/kubernetes/running-minikube-in-aws-ec2-ubuntu/ 

## Introduction ##

### Motivation ###

I'm learning K8s *and* Kubeflow, but I want to limit the costs (and potential clean-up activities required) of running these systems in AWS.
So I need a dev, or sandbox, environment in which I can experiment.

I'd also like to make the environment accessible from the various locations from which I work. 
Work pay for AWS access, so AWS is the suite of services within which I want to develop this dev environment.

### Options ###

A. Run small K8s+Kubeflow nodegroups from a Cloud9 master node - already proved; too expensive and complex for a sandbox; too much tidying up to do.

B. Run a virtual developer environment (VDE), such as:
1. **Microk8s for Kubeflow** - possibly 'the easiest way to provision a *single node* Kubernetes cluster'.
2. **MiniKF** - 'A fast and easy way to deploy Kubeflow on your laptop ... [or ] ... get started with Kubeflow'. Supports moving to a Kubeflow cloud deployment 'with one click, without having to rewrite anything'.
3. **Minikube for Kubeflow** - would support experimenting with Kubeflow, and *also* following the O'Reilly K8s tutorial (if I'm careful).

This script addresses #3. But [this](https://www.kubeflow.org/docs/other-guides/virtual-dev/getting-started-minikube/) approach requires VirtualBox - which is prohibited within Cloud9, and so we'll have to create an SSH workspace from Cloud9 to another machine.


## Steps - TODO: establish whether AWS allows VirtualBox ##

0. Determine EC2 instance type required.
0. Create Cloud9 ssh-workspace.
0. Install a Hypervisor (e.g. VirtualBox).
0. Install kubectl.
0. Install minikube.
0. Test and experiment.

## Determine EC2 instance type required ###

### System requirements ###

According to the [Kubeflow docs](https://www.kubeflow.org/docs/other-guides/virtual-dev/getting-started-minikube/)...

**Prerequisites**
- Laptop, Desktop or a Workstation
  - >= 12GB RAM
  - >= 8 CPU Cores (N.B. MiniKF requires only 2)
  - ~100GB or more Disk Capacity (MiniKF: 50GB)
  - Optional: GPU card
- Mac OS X or Linux (Ubuntu/RedHat/CentOS)
- sudo or admin access on the local machine
- Access to an Internet connection with reasonable bandwidth
- A hypervisor such as VirtualBox, Vmware Fusion, KVM etc. (though installation is covered in the instructions).

## Create Cloud9 ssh-workspace ##

Cloud9 options include only t-, m-, and c- instances (no p-, hence no GPU-compute options), but this will be fine for our purposes.
## Create the EC2 instance ##

For minikube, this means (using the guide [here](https://www.ec2instances.info/?min_memory=12&min_vcpus=2&min_storage=50&region=eu-west-1) the cheaper Cloud9 options meeting this spec include:
- t3.2xlarge (32GiB, 8 vCPU)
- m5.2xlarge (32GiB, 8 vCPU)
- **c5.2xlarge** (16GiB, 8 vCPU) 
- t2.2xlarge (32GiB, 8 vCPU)
- m4.2xlarge (32GiB, 8 vCPU)
- m5.4xlarge (64GiB, 8 vCPU)
- c5.4xlarge (32GiB, 16 vCPU)
- m4.4xlarge (64GiB, 16 vCPU)
- c4.4xlarge (30GiB, 16 vCPU)

Working off local advice, it seems that compute-optimised ('c') instances would be the best option. 
I used a **c5.2xlarge**.

### Operating systems ###
I had issues trying to get minikube to run within an Amazon Linux OS. Using the Ubuntu 18.04 server OS worked well, for me.

## Install a Hypervisor (e.g. Virtual Box) ##
Apparently, this step isn't possible on an EC2 instance. However, some instructions point out that it isn't necessary to run minikube within VMs.
e.g. https://www.radishlogic.com/kubernetes/running-minikube-in-aws-ec2-ubuntu/
Let's try them out...

**in Cloud9...*

## Install AWS CLI ##

- may not be necessary, but useful (and does not come as standard in the Cloud9 **Ubuntu** image)
```
pip install awscli --upgrade --user
aws --version
```

## Install kubectl ##

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

**N.B. Minikube requires Docker - but it's already installed in the Ubuntu 18 environment**

## Install minikube ##
```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/
minikube version
```

## Test and experiment ##

Become a root user.
```
sudo -i
```

If you are not comfortable running commands as root, you must always add sudo before the commands minikube and kubectl: 
"the vm-driver "none" requires sudo".
```
sudo minikube start --vm-driver=none
```

For me, this generated some useful, colourful output:
```
😄  minikube v1.3.1 on Ubuntu 18.04
🤹  Running on localhost (CPUs=8, Memory=15464MB, Disk=9861MB) ...
ℹ️   OS release is Ubuntu 18.04.3 LTS
🐳  Preparing Kubernetes v1.15.2 on Docker 19.03.1 ...
    ▪ kubelet.resolv-conf=/run/systemd/resolve/resolv.conf
💾  Downloading kubeadm v1.15.2
💾  Downloading kubelet v1.15.2
🚜  Pulling images ...
🚀  Launching Kubernetes ... 
🤹  Configuring local host environment ...

⚠️  The 'none' driver provides limited isolation and may reduce system security and reliability.
⚠️  For more information, see:
👉  https://minikube.sigs.k8s.io/docs/reference/drivers/none/

⚠️  kubectl and minikube configuration will be stored in /home/ubuntu
⚠️  To use kubectl or minikube commands as your own user, you may
⚠️  need to relocate them. For example, to overwrite your own settings:

    ▪ sudo mv /home/ubuntu/.kube /home/ubuntu/.minikube $HOME
    ▪ sudo chown -R $USER $HOME/.kube $HOME/.minikube

💡  This can also be done automatically by setting the env var CHANGE_MINIKUBE_NONE_USER=true
⌛  Waiting for: apiserver proxy etcd scheduler controller dns
🏄  Done! kubectl is now configured to use "minikube"
```

### Commands that work ###

- sudo minikube status
- sudo kubectl get services
- which kubectl
- which minikube
- sudo minikube status
- sudo kubectl version
- sudo kubectl get nodes
- sudo minikube

### Commands that don't ###

- sudo minikube ssh
  - `💡  'none' driver does not support 'minikube ssh' command`
- minikube dashboard (yet!)

## Learning with minikube ##


  
  
  
  
  
  
  
  
  
## Installing Kubeflow ##

Install Kubeflow on Amazon EKS
Download 0.6.1+ release of kfctl. This binary will allow you to install Kubeflow on Amazon EKS:
```
curl --silent --location "https://github.com/kubeflow/kubeflow/releases/download/v0.6.1/kfctl_v0.6.1_$(uname -s).tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/kfctl /usr/local/bin
```

Download Kubeflow configuration file:
```
CONFIG=~/environment/kfctl_aws.yaml
curl -Lo ${CONFIG} [....where from?.....]
# ???
```

Set Kubeflow application name:
```
export KFAPP=${CLUSTER_NAME} 
```

Initialize the cluster:
```
kfctl init ${KFAPP} --config=${CONFIG} -V
```

Create and apply AWS and Kubernetes resources in the cluster:
```
cd ${KFAPP}

kfctl generate all -V
kfctl apply all -V
```


