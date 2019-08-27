# Kubeflow Install #
Based on https://aws.amazon.com/blogs/opensource/kubeflow-amazon-eks/ 

## Prerequisites met by the 'kubernetes-setup.md' script ##
- AWS CLI
- An environment where you can build Docker images. AWS recommend using AWS Cloud9 IDE, as it has Docker and AWS CLI pre-installed. Having Docker and AWS CLI installed on your local machine would work, too.
- Have the kubectl command line tool installed.

## Additional prerequisites ##
- Subscribe to the EKS-optimized AMI with GPU Support from the AWS Marketplace 
  - https://aws.amazon.com/marketplace/pp/B07GRHFXGM
  - if you don't necessarily want to start instances immediately, you can always just add it to the Service Catalog rather than launching.
- Ensure that you can launch at least two instances of GPU instances (P2 or P3); you can raise this limit through a service request via the EC2 console.

## Presumed requisites ##
init'ing kubectl failed, initially, citing 
```
Code 400 with message: Could not find command aws-iam-authenticator in PATH
```
... and so I've assumed we also need to install aws-iam-authenticator

## Installing aws-iam-authenticator ##

To install aws-iam-authenticator on Linux

1. Download the Amazon EKS-vended aws-iam-authenticator binary from Amazon S3:
```
curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.13.7/2019-06-11/bin/linux/amd64/aws-iam-authenticator
```
(Optional) Verify the downloaded binary with the SHA-256 sum provided in the same bucket prefix.

2. Download the SHA-256 sum for your system.
```
curl -o aws-iam-authenticator.sha256 https://amazon-eks.s3-us-west-2.amazonaws.com/1.13.7/2019-06-11/bin/linux/amd64/aws-iam-authenticator.sha256
```
Check the SHA-256 sum for your downloaded binary.

```
openssl sha1 -sha256 aws-iam-authenticator
```
Compare the generated SHA-256 sum in the command output against your downloaded aws-iam-authenticator.sha256 file. The two should match.

3. Apply execute permissions to the binary.
```
chmod +x ./aws-iam-authenticator
```

4. Copy the binary to a folder in your $PATH. We recommend creating a $HOME/bin/aws-iam-authenticator and ensuring that $HOME/bin comes first in your $PATH.
```
mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH
```

5. Add $HOME/bin to your PATH environment variable.
```
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
```

6. Test that the aws-iam-authenticator binary works.
```
aws-iam-authenticator help
```
If you have an existing Amazon EKS cluster, create a kubeconfig file for that cluster. 

For more information, see [Create a kubeconfig for Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html). 
Otherwise, see [Creating an Amazon EKS Cluster to create a new Amazon EKS cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html).

## Update the kubeconfig file for an EXISTING cluster ##
```
aws eks --region eu-west-1 update-kubeconfig --name eks-kubeflow
```
...which results, e.g., in ...
```
Added new context arn:aws:eks:eu-west-1:917235138212:cluster/eks-kubeflow to /home/ec2-user/.kube/config
```

Test the config file
```
kubectl get svc
```

## OR create an EKS cluster with GPU instances ##
```
eksctl create cluster eks-kubeflow --node-type=p3.2xlarge --nodes 2 --region eu-west-1 --timeout=40m
```
The final line should look something like `[✔]  EKS cluster "xxxxxxxx" in "xx-xxxx-x" region is ready`.

## Validate the EKS Cluster ##
Run this command to apply the **Nvidia Kubernetes device plugin** as a daemonset on each worker node:
```
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.10/nvidia-device-plugin.yml
```

You can issue these commands to check the status of nvidia-device-plugin **daemonsets** and the corresponding **pods**:
```
kubectl get daemonset -n kube-system
```
...should return something like:
```
NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
aws-node                         2         2         2       2            2           <none>          9m45s
kube-proxy                       2         2         2       2            2           <none>          9m45s
nvidia-device-plugin-daemonset   2         2         2       2            2           <none>          10s
```

```
kubectl get pods -n kube-system -owide |grep nvid
```
...should return:
```
nvidia-device-plugin-daemonset-46f8r   1/1     Running   0          6m24s   192.168.86.195   ip-192-168-81-74.eu-west-1.compute.internal    <none>           <none>
nvidia-device-plugin-daemonset-wzjzp   1/1     Running   0          6m24s   192.168.39.50    ip-192-168-43-175.eu-west-1.compute.internal   <none>           <none>
```

Once the nvidia-device-plugin daemonsets are running, the next command confirms the **number of GPUs** in each worker node:
```
kubectl get nodes \
  "-o=custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu,EC2:.metadata.labels.beta\.kubernetes\.io/instance-type,AZ:.metadata.labels.failure-domain\.beta\.kubernetes\.io/zone"
```

## Storage Class for Persistent Volume ##

EDIT(MR): When I ran the command below, AWS responded:
```
Error from server (AlreadyExists): error when creating "STDIN": storageclasses.storage.k8s.io "gp2" already exists
```
It's not clear at what stage of some previous process this had been created.

Kubeflow requires a default storage class to spawn Jupyter notebooks with attached persistent volumes. A StorageClass in Kubernetes provides a way to describe the type of storage (e.g., types of EBS volume: io1, gp2, sc1, st1) that an application can request for its persistent storage. 

The following command creates a Kubernetes default storage class for dynamic provisioning of persistent volumes backed by Amazon Elastic Block Store (EBS) with the general-purpose SSD volume type (gp2).
```
cat <<EOF | kubectl create -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp2
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Delete
mountOptions:
  - debug
EOF
```

Validate that the default StorageClass is created using the command below:
```
kubectl get storageclass
```
Good output looks like:
```
NAME            PROVISIONER             AGE
gp2 (default)   kubernetes.io/aws-ebs   2d
```

## Install Kubeflow ##
Kubeflow uses **ksonnet**, a command line tool that simplifies the configuration and deployment of applications in multiple Kubernetes environments. Ksonnet abstracts Kubernetes resources as **Prototypes**. Ksonnet uses these prototypes to generate **Components** as Kubernetes YAML files, which are tuned for specific implementations by filling in the parameters of the prototypes. A different set of parameters can be used for each Kubernetes environment.

### Install ksonnet CLI ###
...from: https://ksonnet.io/get-started/

(on MacOS, you can also use `brew install ksonnet/tap/ks`).

Download and unpack the ks CLI binary, move it into the binaries directory, make it executable.
```
mkdir ~/environment/temp
sudo curl --location \
  -o ~/environment/temp/ks_0.13.1_linux_amd64.tar.gz \
  https://github.com/ksonnet/ksonnet/releases/download/v0.13.1/ks_0.13.1_linux_amd64.tar.gz
sudo tar xvf ~/environment/temp/ks_0.13.1_linux_amd64.tar.gz -C ~/environment/temp/
sudo cp temp/ks_0.13.1_linux_amd64/ks /usr/local/bin
sudo rm ~/environment/temp/ks_0.13.1_linux_amd64.tar.gz 
sudo chmod +x /usr/local/bin/ks
```

Verify the ks binary is in the path, and that it is executable:
```
command -v ks
```

Validate that you have version >=0.12.0 of ksonnet:
```
ks version
```
e.g.
```
ksonnet version: 0.12.0
jsonnet version: v0.11.2
client-go version: kubernetes-1.10.4
```

## Install Kubeflow on Amazon EKS ##

### Did this, but may be from out-of-date instructions ###
First, create a new Kubernetes namespace for the Kubeflow deployment:
```
export NAMESPACE=kubeflow
kubectl create namespace ${NAMESPACE}
```

### Kubeflow's own instructions ###
In order to deploy Kubeflow on your existing Amazon EKS cluster, you need to provide AWS_CLUSTER_NAME, cluster region and worker roles.

Download the latest `kfctl` golang binary from Kubeflow release page and unpack it.
```

VERSION="0.6.2"
sudo curl --location \
  -o ~/environment/temp/kfctl_v${VERSION}_linux.tar.gz \
  https://github.com/kubeflow/kubeflow/releases/download/v${VERSION}/kfctl_v${VERSION}_linux.tar.gz
  
sudo tar xvf ~/environment/temp/kfctl_v${VERSION}_linux.tar.gz -C ~/environment/temp/
sudo cp ~/environment/temp/kfctl /usr/local/bin
sudo rm ~/environment/temp/kfctl
sudo rm ~/environment/temp/kfctl/v${VERSION}_linux.tar.gz 
sudo chmod +x /usr/local/bin/kftcl
```

Download config file
```
export CONFIG="/tmp/kfctl_aws.yaml"
wget https://raw.githubusercontent.com/kubeflow/kubeflow/v${VERSION}/bootstrap/config/kfctl_aws.yaml -O ${CONFIG}
```
kfctl_aws.yaml is one of setup manifests, please check kfctl_aws_cognito.yaml for the template to enable authentication.

Customize your config file. Retrieve the Amazon EKS cluster name, AWS Region, and IAM role name for your worker nodes.
```
export AWS_CLUSTER_NAME='eks-kubeflow'
export KFAPP=${AWS_CLUSTER_NAME}
```
Note: To get your Amazon EKS worker node IAM role name, you can check IAM setting by running the following commands. This command assumes that you used eksctl to create your cluster. If you use other provisioning tools to create your worker node groups, please find the role that is associated with your worker nodes in the Amazon EC2 console.
```
aws iam list-roles \
    | jq -r ".Roles[] \
    | select(.RoleName \
    | startswith(\"eksctl-$AWS_CLUSTER_NAME\") and contains(\"NodeInstanceRole\")) \
    .RoleName"
```
...which outputs, e.g., `eksctl-eks-kubeflow-nodegroup-ng-NodeInstanceRole-12C63K9TZDC3J`

Change cluster region and worker roles names in your kfctl_aws.yaml, e.g.
```
region: eu-west-1
  roles:
    - eksctl-eks-kubeflow-nodegroup-ng-NodeInstanceRole-12C63K9TZDC3J
```
If you have multiple node groups, you will see corresponding number of node group roles. In that case, please provide the role names as an array.

Run the following commands to set up your environment and initialize the cluster - n.b. ...
- KFAPP - (set above) Use a relative directory name here rather than absolute path, such as kfapp. It will be used as eks cluster name.
- CONFIG - (set above) Path to the configuration file 
```
kfctl init ${KFAPP} --config=${CONFIG} -V
cd ${KFAPP}

kfctl generate all -V
kfctl apply all -V
```

KFAPP - Use a relative directory name here rather than absolute path, such as kfapp. It will be used as eks cluster name.
CONFIG - Path to the configuration file
Important!!! By default, these scripts create an AWS Application Load Balancer for Kubeflow that is open to public. This is good for development testing and for short term use, but we do not recommend that you use this configuration for production workloads.

To secure your installation, Follow the instructions to add authentication.


