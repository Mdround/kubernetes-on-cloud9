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
1. **Microk8s for Kubeflow** - possibly 'the easiest way to provision a *single node* Kubernetes cluster', but no idea whether Kubeflow would also be possible.
2. **MiniKF** - 'A fast and easy way to deploy Kubeflow on your laptop ... [or ] ... get started with Kubeflow'. Supports moving to a Kubeflow cloud deployment 'with one click, without having to rewrite anything'.
3. **Minikube for Kubeflow** - would support following the O'Reilly tutorial, and *also* experimenting with Kubeflow.

My choice at this stage is #3.

## Steps ##

0. Determine EC2 instance type required.
0. Create Cloud9 environment.
0. Install kubectl.
0. Install minikube.
0. Test and experiment.
