# 🚀 Deploying E-Commerce App on AWS EKS

## 1. Install AWS CLI
***Download and install the AWS CLI v2:***
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
### Configure AWS CLI:
```bash
aws configure
```
## 2. Install kubectl
```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.29.0/2024-01-04/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/
kubectl version --client
```
## 3. Install eksctl
```bash
curl -s https://api.github.com/repos/weaveworks/eksctl/releases/latest \
| grep "browser_download_url.*linux_amd64.tar.gz" \
| cut -d '"' -f 4 \
| wget -qi -
tar -xzf eksctl_*_linux_amd64.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### 1. Prerequisites
AWS account with IAM permissions for EKS, EC2, VPC, and IAM roles

AWS CLI installed and configured (aws configure)

kubectl installed

eksctl installed (simplifies cluster creation)

Domain name configured in Route53 (optional, for ingress + TLS)

Cert-manager and Ingress Controller (Nginx or Traefik)
# 2. Create EKS Cluster
```sh
eksctl create cluster \
  --name ecom-cluster \
  --region ap-southeast-1 \
  --nodegroup-name ecom-nodes \
  --node-type t3.medium \
  --nodes 1 \
  --nodes-min 2 \
  --nodes-max 4 \
  --managed
``` 
### This provisions:

Control plane (managed by AWS)

Worker nodes in an Auto Scaling Group

VPC networking
# 3. Verify Cluster
```sh
aws eks describe-cluster --name ecom-cluster
kubectl get nodes
kubectl get pods -A
```
# 4. Deploy Application
same as k3s

Check the status of the application:
```sh
kubectl get pods
kubectl get svc
kubectl get nodes -o wide
kubectl describe pod <pod-name>
kubectl logs -f <pod-name>
```
# 5. Ingress + TLS
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecom-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - yourdomain.com
    secretName: ecom-tls
  rules:
  - host: yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ecom-service
            port:
              number: 80
```
# 7. same Install Cert-Manager
# 8. same Create ClusterIssuer (Let’s Encrypt)
# 9. same Create Ingress for your app
# 10. check your app
```sh
kubectl get ingress 
kubectl get certificate
kubectl describe certificate
kubectl describe challenge
kubectl get svc -n ingress-nginx
# scale deployment
kubectl scale deployment ecom-deployment --replicas=4

```

# 11.cleanup
```sh
eksctl delete cluster --name ecom-cluster
```