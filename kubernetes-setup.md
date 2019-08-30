# kubernetes-on-cloud9
A record of one process for setting up Kubernetes within AWS on Cloud9, adapted (and quoting almost verbatim) from [AWS' EKS workshop](https://eksworkshop.com/).

## IN THE CONSOLE ##

- Create an AWS account (or use an existing acct with 'AdministratorAccess' privileges).
- Create a Cloud9 envt, 
  - e.g. for eu-west-1,  https://eu-west-1.console.aws.amazon.com/cloud9/home?region=eu-west-1
  - name it (e.g. `eksworkshop`), sticking with the defaults otherwise.
- Create a new IAM role with Admin access
  - e.g. using the link: https://console.aws.amazon.com/iam/home#/roles$new?step=review&commonUseCase=EC2%2BEC2&selectedUseCase=EC2&policies=arn:aws:iam::aws:policy%2FAdministratorAccess
  - Confirm that '**AWS service**' and '**EC2**' are selected, then click '**Next**' to view permissions.
  - Confirm that '**AdministratorAccess**' is checked, then click '**Next: Tags**' to assign tags.
  - Name the role (e.g. `eksworkshop-admin`), and click '**Create role**'.

## IN EC2 ##

Attach the IAM role to the (Cloud9) workspace:
- e.g. by finding your instance here: https://console.aws.amazon.com/ec2/v2/home?#Instances:tag:Name=aws-cloud9-eksworkshop*;sort=desc:launchTime 
- select the Cloud9 instance, then '**Actions > Instance Settings > Attach/Replace IAM role**'; then
- select  '**eksworkshop-admin**' for '**IAM-role**', and click '**Apply**'.

## IN CLOUD9 ##

### Install Kubernetes tools ###
```
sudo curl --silent --location \-o /usr/local/bin/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.13.7/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl
``` 

### Optional: Install JQ and envsubst ##
N.B. while they are not required for the **KubeFlow** part of the workshop, they're useful - e.g. in querying AWS details to set some convenient variables recording AWS region, account_id, etc.
```
sudo yum -y install jq gettext
```

### Optional: clone the service repos ##
N.B. these **aren't** essential to the Kubernetes part of the workshop
```
cd ~/environment
git clone https://github.com/brentley/ecsdemo-frontend.git
git clone https://github.com/brentley/ecsdemo-nodejs.git
git clone https://github.com/brentley/ecsdemo-crystal.git
```

### Verify that the downloaded binaries are in the path and executable ###
```
for command in kubectl jq envsubst
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done
```

### Update IAM settings for the Cloud9 workspace ###
According to the workshop, 
> 'Cloud9 normally manages IAM credentials dynamically. This isn’t currently compatible with the EKS IAM authentication, so we will disable it and rely on the IAM role instead'.

- **'Preferences' > 'AWS Settings' > 'Credentials' > 'AWS managed temporary credentials' > [OFF]**
- **Close** the Prefs window, to apply the changes.

To ensure temporary credentials aren’t already in place we also remove any existing credentials file:
```
rm -vf ${HOME}/.aws/credentials
```

Configure the AWS CLI with our current region as default:
```
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')

echo "export ACCOUNT_ID=${ACCOUNT_ID}" >> ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" >> ~/.bash_profile

aws configure set default.region ${AWS_REGION}
aws configure get default.region
```

**Check** that the right IAM role is attached to the Cloud9 workspace. The output assumed-role name (ARN) should contain the IAM role defined above (e.g `eksworkshop-admin`):
```
aws sts get-caller-identity
```

### Generate an ssh key in Cloud9 ###
This key will be used on the worker node instances to allow ssh access if necessary.
```
ssh-keygen
```
- Press 'enter' 3 times, to accept the defaults.

Upload the public key to your EC2 region:
```
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material file://~/.ssh/id_rsa.pub
```

## EKS ##

Download the eksctl binary
```
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
```

Verify that the eksctl binary is in the path, and executable:
```
which eksctl &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
eksctl version
```
And you're **ready to start** an EKS cluster. 

*N.B.*
It's not clear, from the workshop instructions, whether their process for Kubeflow deployment assumed the step below ('If you're planning to run Machine Learning workloads...') as a requirement.

TODO: try the Kubeflow step without first following the step below.

## Create an EKS cluster ##
This takes 15 mins to execute: enough time to suck a mint, buy a sledge, have a fast bath. 
**N.B.** you may get an error if the stack (e.g. 'eksctl-eksworkshop-eksctl-cluster') already exists from a previous session.
```
date
eksctl create cluster --version=1.13 --name=eksworkshop-eksctl --nodes=3 --node-ami=auto --region=${AWS_REGION}

# N.B. If you're planning to run Machine Learning workloads, then use the following command INSTEAD
curl -OL https://raw.githubusercontent.com/aws-samples/eks-workshop/master/content/eksctl/launcheks.files/eksworkshop-kubeflow.yml.template
export AWS_AZS=$(aws ec2 describe-availability-zones --region=${AWS_REGION} --query 'AvailabilityZones[*].ZoneName' --output json | tr '\n' ' ' | sed 's/[][]//g')
export AWS_AZ=$(aws ec2 describe-availability-zones --region=${AWS_REGION} --query 'AvailabilityZones[0].ZoneName' --output json)
echo "export AWS_AZS=${AWS_AZS}" >> ~/.bash_profile
export "AWS_AZ=${AWS_AZ}" >> ~/.bash_profile
envsubst <eksworkshop-kubeflow.yml.template >eksworkshop-kubeflow.yml

eksctl create cluster -f eksworkshop-kubeflow.yml
```

Test the cluster: confirm your Nodes
```
kubectl get nodes
```






## Optional: Export useful details as reusable variables ##
```
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
INSTANCE_PROFILE_ARN=$(aws cloudformation describe-stacks --stack-name $STACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InstanceProfileARN") | .OutputValue')
ROLE_NAME=$(aws cloudformation describe-stacks --stack-name $STACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InstanceRoleARN") | .OutputValue' | cut -f2 -d/)
echo "export ROLE_NAME=${ROLE_NAME}" >> ~/.bash_profile
echo "export INSTANCE_PROFILE_ARN=${INSTANCE_PROFILE_ARN}" >> ~/.bash_profile
```




## Optional: Deploy the K8s dashboard ##

... with the following command (n.b. might might to update version numbers, in the future)
```
kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

Since this is deployed to our private cluster, we need to access it via a proxy. 
Kube-proxy is available to proxy our requests to the dashboard service. In your workspace, run the following command:
```
kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &
```
This will start the proxy, listen on port 8080, listen on all interfaces, and will disable the filtering of non-localhost requests.
This command will continue to run in the background of the current terminal’s session.

To access the K8s dashboard: 
- in your Cloud9 environment, click '**Tools / Preview / Preview Running Application**';
- scroll to the end of the URL and append: 
- `/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`

Open a new Terminal tab and enter:
```
aws eks get-token --cluster-name eksworkshop-eksctl | jq -r '.status.token'
```
- Copy the output of this command,
- click the radio button next to '**Token**', 
- in the text field below paste the output from the last command, then
- press '**Sign In**'.

## Delete the cluster ##
```
kubectl get svc --all-namespaces
eksctl delete cluster --name=eksworkshop-eksctl
```
