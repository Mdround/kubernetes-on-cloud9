# kubernetes-on-cloud9
Non-automated script for setting up Kubernetes within AWS on Cloud9
adapted from AWS' EKS workshop

### IN THE CONSOLE ###

Create an AWS account (already done - used existing acct, as it had priv's)

Create a Cloud9 envt, named 'eksworkshop', otherwise using the defaults

Create a new IAM role with Admin access, using...
 https://console.aws.amazon.com/iam/home#/roles$new?step=review&commonUseCase=EC2%2BEC2&selectedUseCase=EC2&policies=arn:aws:iam::aws:policy%2FAdministratorAccess
 - Confirm that 'AWS service' and 'EC2' are selected, then click `Next` to view permissions
 - Confirm that 'AdministratorAccess' is checked, then click `Next: Tags` to assign tags
 - Name the role (e.g. `eksworkshop-admin`), and click `Create role`

### IN EC2 ###

Attach the IAM role to the (Cloud9) workspace, by
- selecting the Cloud9 instance, then 'Actions > Instance Settings > Attach/Replace IAM role';
- then selecting 'eksworkshop-admin' for 'IAM-role', and clicking 'Apply'.

### IN CLOUD9 ###

# Install kubectl
```
sudo curl --silent --location -o /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/v1.13.7/bin/linux/amd64/kubectl
sudo chmod +x /usr/local/bin/kubectl
``` 

# install JQ and envsubst    ... for later in the EKS workshop, I assume...
```
sudo yum -y install jq gettext
```

Verify the binaries are in the path and executable
```
for command in kubectl jq envsubst;   do     which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND";   done
```

Clone the service repos  ... for later in the workshop!
```
cd ~/environment
git clone https://github.com/brentley/ecsdemo-frontend.git
git clone https://github.com/brentley/ecsdemo-nodejs.git
git clone https://github.com/brentley/ecsdemo-crystal.git
```

Update IAM settings for the [Cloud9] workspace
- 'Preferences' > 'AWS Settings' > 'Credentials' > 'AWS managed temporary credentials' > [OFF]
- [Close the Prefs window]

To ensure temporary credentials aren’t already in place we also remove any existing credentials file:
`rm -vf ${HOME}/.aws/credentials`

We should configure our AWS CLI with our current region as default:
```
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
echo "export ACCOUNT_ID=${ACCOUNT_ID}" >> ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" >> ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
```

Check that the right IAM role is attached to the CLOUD9 workspace. The output assumed-role name (ARN) should contain: 'eksworkshop-admin' or 'TeamRole'
`aws sts get-caller-identity`

Run this command to generate SSH Key in Cloud9. This key will be used on the worker node instances to allow ssh access if necessary.
`ssh-keygen`
- Press 'enter' 3 times, to accept defaults

 Upload the public key to your EC2 region:
`aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material file://~/.ssh/id_rsa.pub`

 Download the eksctl binary
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin
eksctl version
```

Create an EKS cluster
`eksctl create cluster --version=1.13 --name=eksworkshop-eksctl --nodes=3 --node-ami=auto --region=${AWS_REGION}`
-  Wait 10-15 mins...

 Test the cluster: confirm your Nodes
`kubectl get nodes`

 Export the Worker Role Name _ was used throughout the subsequent workshop
```
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
INSTANCE_PROFILE_ARN=$(aws cloudformation describe-stacks --stack-name $STACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InstanceProfileARN") | .OutputValue')
ROLE_NAME=$(aws cloudformation describe-stacks --stack-name $STACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InstanceRoleARN") | .OutputValue' | cut -f2 -d/)
echo "export ROLE_NAME=${ROLE_NAME}" >> ~/.bash_profile
echo "export INSTANCE_PROFILE_ARN=${INSTANCE_PROFILE_ARN}" >> ~/.bash_profile
```

# Deploy the K8s dashboard with the following command
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

# Since this is deployed to our private cluster, we need to access it via a proxy. 
# Kube-proxy is available to proxy our requests to the dashboard service. In your workspace, run the following command:
kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &
# This will start the proxy, listen on port 8080, listen on all interfaces, and will disable the filtering of non-localhost requests.
# This command will continue to run in the background of the current terminal’s session.

# To access the K8s dashboard: 
## In your Cloud9 environment, click 'Tools / Preview / Preview Running Application'
## Scroll to the end of the URL and append: 
## /api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
# Open a New Terminal Tab and enter:
## aws eks get-token --cluster-name eksworkshop-eksctl | jq -r '.status.token'
# Copy the output of this command and 
# then click the radio button next to Token 
# then in the text field below paste the output from the last command.
# Then press Sign In.

# Clean up
kubectl get svc --all-namespaces
eksctl delete cluster --name=eksworkshop-eksctl
