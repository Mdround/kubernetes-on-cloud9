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

## Create the EC2 instance ##

As GPU cards are an 'option', I chose:
- the `Deep Learning AMI (Ubuntu) Version 24.0 - ami-04b29aaed8d74f8f3` AMI,
- a `p3.2xlarge` to run it on (8 CPUs, 61GiB of RAM).

## Create Cloud9 ssh-workspace ##

## Install a Hypervisor (e.g. Virtual Box) ##

## Install kubectl ##

## Install minikube ##

## Test and experiment ##

