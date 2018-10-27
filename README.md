# hoolah_challenge
Git repo for Hoolah Code Challenge

## Pre-requisites:
* an AWS account for deploying the environment
* an PC or server environment with AWS CLI (Command Line Interface) installed and configured

## How to get the sources:
```sh
git clone git://github.com/logicgate1978/hoolah_challenge.git
```

## How it works
I use AWS CloudFormation template & shell scripts to provision the environment.

## Steps:
1. since key pair cannot be created using CloudFormation, we need manually create it.
```sh
echo -e `aws ec2 create-key-pair --key-name MyKeyPair --profile={your-aws-profile} --region=ap-southeast-1 | grep KeyMaterial | awk -F\" '{print $4}'` > ~/.ssh/mykeypair.pem
```

2. for security, we store the private key to AWS Secret Manager, and remove the local key file.
```sh
aws secretsmanager create-secret --name MyPrivateKey --secret-string file://~/.ssh/mykeypair.pem --profile={your-aws-profile} --region=ap-southeast-1
rm -f ~/.ssh/mykeypair.pem
```

3. use AWS CLI to create the stack
```sh
aws cloudformation create-stack --stack-name bastionStack --template-body file://bastionStack.yml --profile={your-aws-profile} --region=ap-southeast-1 
```

4. use AWS CLI to retrieve output of the CloudFormation stack. The output contains EIP of the bastion host, and internal IP of the private host. You may need to wait for a while until the CloudFormation stack is created successfully. 
```sh
aws cloudformation describe-stacks --stack-name bastionStack --query 'Stacks[0].Outputs' --profile={your-aws-profile} --region=ap-southeast-1
```

5. test SSH to bastion host, the private key is restrieved from AWS Secret Manager.
```sh
eval `ssh-agent -s`

PRIVATE_KEY=`aws secretsmanager get-secret-value --secret-id MyPrivateKey --profile={your-aws-profile} --region=ap-southeast-1 | grep SecretString | awk -F\" '{print $4}'`

ssh-add -k <(echo -e "$PRIVATE_KEY")

ssh ec2-user@{private-host-ip} -o "proxycommand ssh -W %h:%p ec2-user@{bation-host-ip}"
```
