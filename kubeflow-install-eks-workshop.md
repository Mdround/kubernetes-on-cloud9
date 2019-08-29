# Kubeflow on AWS install - EKS Workshop version #

## MDR: Create an EKS cluser ##
(from https://eksworkshop.com/eksctl/launcheks/#create-eks-cluster-1)

```
# Normally, you'd do this by: 
# eksctl create cluster --version=1.13 --name=eksworkshop-eksctl --nodes=3 --node-ami=auto --region=${AWS_REGION}`
# But, if you're planning to run Machine Learning workloads, then use the following commands instead:

curl -OL https://raw.githubusercontent.com/aws-samples/eks-workshop/master/content/eksctl/launcheks.files/eksworkshop-kubeflow.yml.template

export AWS_AZS=$(aws ec2 describe-availability-zones --region=${AWS_REGION} --query 'AvailabilityZones[*].ZoneName' --output json | tr '\n' ' ' | sed 's/[][]//g')
export AWS_AZ=$(aws ec2 describe-availability-zones --region=${AWS_REGION} --query 'AvailabilityZones[0].ZoneName' --output json)
echo "export AWS_AZS=${AWS_AZS}" >> ~/.bash_profile
export "AWS_AZ=${AWS_AZ}" >> ~/.bash_profile
envsubst <eksworkshop-kubeflow.yml.template >eksworkshop-kubeflow.yml

eksctl create cluster -f eksworkshop-kubeflow.yml
```

Confirm that the nodes are visible:
```
kubectl get nodes # if we see our node(s), we know we have authenticated correctly
```

Export the Worker Role Name for use throughout the workshop (but maybe not in the Kubeflow section?):
```
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
INSTANCE_PROFILE_ARN=$(aws cloudformation describe-stacks --stack-name $STACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InstanceProfileARN") | .OutputValue')
ROLE_NAME=$(aws cloudformation describe-stacks --stack-name $STACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InstanceRoleARN") | .OutputValue' | cut -f2 -d/)
echo "export ROLE_NAME=${ROLE_NAME}" >> ~/.bash_profile
echo "export INSTANCE_PROFILE_ARN=${INSTANCE_PROFILE_ARN}" >> ~/.bash_profile
```

## Install Kubeflow on Amazon EKS ##
Download 0.6.1+ release of kfctl. This binary will allow you to install Kubeflow on Amazon EKS:

```
curl --silent --location "https://github.com/kubeflow/kubeflow/releases/download/v0.6.1/kfctl_v0.6.1_$(uname -s).tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/kfctl /usr/local/bin
```

Download Kubeflow configuration file:
```
CONFIG=~/environment/kfctl_aws.yaml
curl -Lo ${CONFIG} https://raw.githubusercontent.com/kubeflow/kubeflow/v0.6.1/bootstrap/config/kfctl_aws.yaml
```

Customize this configuration file for AWS region and IAM role for your worker nodes:
```
sed -i "s@eksctl-kubeflow-aws-nodegroup-ng-a2-NodeInstanceRole-xxxxxxx@$ROLE_NAME@" ${CONFIG}
sed -i "s@us-west-2@$AWS_REGION@" ${CONFIG}
```

Until https://github.com/kubeflow/kubeflow/issues/3827 is fixed, install **aws-iam-authenticator**:
```
curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.13.7/2019-06-11/bin/linux/amd64/aws-iam-authenticator
chmod +x aws-iam-authenticator
sudo mv aws-iam-authenticator /usr/local/bin
``` 

Set Kubeflow application name:
```
export AWS_CLUSTER_NAME=eksworkshop-eksctl
export KFAPP=${AWS_CLUSTER_NAME}
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

Wait for all pods to be in Running state (this can take a few minutes):
```
kubectl get pods -n kubeflow
```

Validate that GPUs are available:
```
kubectl get nodes "-o=custom-columns=NAME:.metadata.name,MEMORY:.status.allocatable.memory,CPU:.status.allocatable.cpu,GPU:.status.allocatable.nvidia\.com/gpu"
# Should give results like:
# NAME                                          MEMORY        CPU   GPU
# ip-192-168-54-93.eu-west-1.compute.internal   251641628Ki   32    4
# ip-192-168-68-80.eu-west-1.compute.internal   251641628Ki   32    4
```
... although the script above actually setup only one node, for me.


## Kubeflow Dashboard ##
Get the Kubeflow service endpoint:
```
kubectl get ingress -n istio-system -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'
```
The command above will return the endpoint address. Use this address in a browser to access the Kubeflow dashboard.

Re the screen "Name your workspace. A namespace is a collection of Kubeflow services. Resources created within a namespace are isolated to that namespace. By default, a namespace will be created for you."

N.B. I initially went with the default namespace, 'anonymous-kubeflow.org', but ended up in an endless loop of splash pages.

Should have followed the EKS Workshop instructions, and used `eksworkshop`. This worked.

## Setting up Jupyter notebooks ##
... still following https://eksworkshop.com/kubeflow/jupyter/ (learned my lesson!)

- Under 'Quick Shortcuts' in the Kubeflow dashboard, click on 'Create a new Notebook Server'.
- Select the namespace created in previous step (`eksworkshop`) - this pre-populates the namespace field on the dashboard. 
- Specify a name (e.g. `myjupyter`) for the notebook and change the CPU value to `1.0`.
- Scroll to the bottom, take all other defaults, and click on LAUNCH.
- Wait **2 mins** for the Jupyter notebook to come online. 
- Click on CONNECT

## Testing TensorFlow within a notebook ##
- In Jupyter, click 'New > Python 3'
- Copy into your notebook the sample training code from https://eksworkshop.com/kubeflow/kubeflow.files/mnist-tensorflow.py
- Click on *Run* within the notebook to load this code.
- This also creates a new code block. Write the command `main()` in this new code block and click on **Run** again.





