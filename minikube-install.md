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


## Steps - TODO: revise these in the light of ssh-workspace reqt ##

0. Determine EC2 instance type required.
0. Create Cloud9 ssh-workspace.
0. Install a Hypervisor (e.g. Virtual Box).
0. Install kubectl.
0. Install minikube.
0. Test and experiment.

## System requirements ##

According to the [Kubeflow docs](https://www.kubeflow.org/docs/other-guides/virtual-dev/getting-started-minikube/):

### 0. Determine EC2 instance type required ###

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

Cloud9 options include only t-, m-, and c- instances (no p-, hence no GPU-compute options), but this will be fine for our purposes.

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
I used a c5.2xlarge.

** N.B.** For MiniKF alone, we only need 2 CPUs, so could get also away with:
- m3.xlarge (16GiB, 4 vCPU)
- m4.xlarge (16GiB, 4 vCPU)
- m5.xlarge (16GiB, 4 vCPU)
- t2.xlarge (16GiB, 4 vCPU)
- t3.xlarge (16GiB, 4 vCPU)



