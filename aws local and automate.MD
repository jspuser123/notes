#  ***aws local and automate***
# 1.Install tools
```bash
sudo apt update
sudo apt install -y unzip curl
```
# 2. Install aws
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

# 3. Install eksctl
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin

eksctl version
eksctl create cluster --name my-cluster --region ap-south-1

```

# 4. Install kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/

kubectl version --client
# update kubeconfig
aws eks --region ap-south-1 update-kubeconfig --name my-cluster

kubectl get nodes
```

# 5 Configure the aws cli not using aws ec2 credentials only local
```bash
aws configure

AWS Access Key ID: XXXXX
AWS Secret Access Key: XXXXX
Default region name: ap-south-1
Default output format: json

aws sts get-caller-identity
```


# EKSWorkerNodeRole upload json
## 1. rolle create
```bash
aws iam create-role \
  --role-name EKSWorkerNodeRole \
  --assume-role-policy-document file://trust-policy.json
```
## 2. trust policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```
## 3. attach policy
```bash
aws iam attach-role-policy --role-name EKSWorkerNodeRole --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name EKSWorkerNodeRole --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam attach-role-policy --role-name EKSWorkerNodeRole --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
```
# 1. Region Set
```bash
export AWS_REGION=ap-south-1
```
# 2. Staus Check
```bash
eksctl get cluster
aws iam get-role --role-name EKSWorkerNodeRole
aws iam get-role --role-name EKSClusterRole
aws sts get-caller-identity
aws sts get-session-token
aws configure get region
aws sts assume-role --role-arn <role-arn> --role-session-name "eks-cluster-session"
#example
aws sts assume-role --role-arn "arn:aws:iam::188612031961:role/EKSClusterRole" --role-session-name "eks-cluster-session"
aws eks --region ap-south-1 list-clusters
```
# 3. Create Cluster
```bash

eksctl create cluster \
  --name cluster1 \
  --region ap-south-1 \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3
```
# 4. Update Kubeconfig
```bash
aws eks --region ap-south-1 update-kubeconfig --name cluster1
```


## or
# Already Crateded Vpc Add 
## Create vpc
```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region ap-south-1
```
## Create subnet
```bash
aws ec2 create-subnet --vpc-id vpc-xxxxxxxx --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a --region ap-south-1
aws ec2 create-subnet --vpc-id vpc-xxxxxxxx --cidr-block 10.0.2.0/24 --availability-zone ap-south-1b --region ap-south-1

# example
aws ec2 create-subnet --vpc-id vpc-074705e35fa467a15 --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a --region ap-south-1
aws ec2 create-subnet --vpc-id vpc-074705e35fa467a15 --cidr-block 10.0.2.0/24 --availability-zone ap-south-1b --region ap-south-1
```

## Create secrity groups
```bash
aws ec2 create-security-group --group-name eks-security-group --description "Security group for EKS" --vpc-id vpc-xxxxxxxx --region ap-south-1

# example

aws ec2 create-security-group --group-name eks-security-group --description "Security group for EKS" --vpc-id vpc-074705e35fa467a15 --region ap-south-1

```
## Configure Security Group Ingress Rules
```bash
aws ec2 authorize-security-group-ingress --group-id sg-xxxxxxxx --protocol tcp --port 443 --cidr 0.0.0.0/0 --region ap-south-1
aws ec2 authorize-security-group-ingress --group-id sg-xxxxxxxx --protocol tcp --port 10250 --cidr 0.0.0.0/0 --region ap-south-1
aws ec2 authorize-security-group-ingress --group-id sg-xxxxxxxx --protocol tcp --port 10255 --cidr 0.0.0.0/0 --region ap-south-1

# example

aws ec2 authorize-security-group-ingress --group-id sg-0ec9d6e1210db122c --protocol tcp --port 443 --cidr 0.0.0.0/0 --region ap-south-1
aws ec2 authorize-security-group-ingress --group-id sg-0ec9d6e1210db122c --protocol tcp --port 10250 --cidr 0.0.0.0/0 --region ap-south-1
aws ec2 authorize-security-group-ingress --group-id sg-0ec9d6e1210db122c --protocol tcp --port 10255 --cidr 0.0.0.0/0 --region ap-south-1

```
## Create cluster
```bash
aws eks --region ap-south-1 create-cluster \
  --name cluster1 \
  --role-arn arn:aws:iam::188612031961:role/EKSClusterRole \
  --resources-vpc-config subnetIds=subnet-05f5247a60130f2af,subnet-02bf7f7e05858a16b,securityGroupIds=sg-0ec9d6e1210db122c

```
## Create cluster and nodegroup
```bash
aws eks --region ap-south-1 create-nodegroup \
--cluster-name cluster1 \   
--nodegroup-name nodegroup1 \
--node-role arn:aws:iam::188612031961:role/EKSWorkerNodeRole \
--subnets subnet-05f5247a60130f2af subnet-02bf7f7e05858a16b \
--scaling-config minSize=1,maxSize=2,desiredSize=1 \
--instance-types t3.micro \
--ami-type BOTTLEROCKET_x86_64
```
## or 
## Create nodegroup with different instance type kube-version 1.32
```bash
aws eks --region ap-south-1 create-nodegroup \
  --cluster-name cluster1 \
  --nodegroup-name nodegroup1 \
  --node-role arn:aws:iam::188612031961:role/EKSWorkerNodeRole \
  --subnets subnet-02cab96ff2a04431d subnet-0c0c45aaadab8bdce subnet-07ddcabafe967b7e6 \
  --scaling-config minSize=1,maxSize=2,desiredSize=1 \
  --instance-types t3.micro \
  --ami-type AL2_x86_64 \
  --kubernetes-version 1.32
```
# Status Check
```bash
aws eks --region ap-south-1 update-kubeconfig --name cluster1

aws ec2 describe-route-tables --region ap-south-1

aws eks list-nodegroups --cluster-name cluster1 --region ap-south-1

```

# Reboot Node
```bash
aws ec2 reboot-instances --instance-ids i-0d95659915ca16544
```
# Scal Down nodegroup
```bash
aws eks update-nodegroup-config \
  --region ap-south-1 \
  --cluster-name cluster1 \
  --nodegroup-name nodegroup1 \
  --scaling-config minSize=1,maxSize=1,desiredSize=1
```
# delete nodegroup
```bash
# node
aws eks delete-nodegroup \
  --region ap-south-1 \
  --cluster-name cluster1 \
  --nodegroup-name nodegroup1

#delete node
aws ec2 release-address --allocation-id <eip-allocation-id>
# delete volume
aws ec2 delete-volume --volume-id <volume-id>
# 
aws elbv2 delete-load-balancer --load-balancer-arn <load-balancer-arn>
```
# Delete cluster
```bash
eksctl delete cluster --name cluster1 --region ap-south-1
(OR)
aws eks delete-cluster --region ap-south-1 --name cluster1
```

***This is test my aws cli on my local machine***
