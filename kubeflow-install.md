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

## Create an EKS cluster with GPU instances ##
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

### Download ksonnet CLI ###
...from: https://ksonnet.io/get-started/

(on MacOS, you can also use `brew install ksonnet/tap/ks`).


ks_0.13.1_linux_amd64.tar.gz





Validate that you have version 0.12.0 of ksonnet:
```
ks version
```
e.g.
```
ksonnet version: 0.12.0
jsonnet version: v0.11.2
client-go version: kubernetes-1.10.4
```
