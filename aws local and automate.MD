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
# 3 Configure the aws cli not using aws ec2 credentials only local
```bash
aws configure

AWS Access Key ID: XXXXX
AWS Secret Access Key: XXXXX
Default region name: ap-south-1
Default output format: json

aws sts get-caller-identity
```
# 4. Install eksctl
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin

eksctl version

eksctl create cluster --name my-cluster --region ap-south-1
```

# 5. Install kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/

kubectl version --client

aws eks --region ap-south-1 update-kubeconfig --name my-cluster

kubectl get nodes
```

# 6. Test and delete
```bash
eksctl get cluster
eksctl delete cluster --name my-cluster --region ap-south-1
```
