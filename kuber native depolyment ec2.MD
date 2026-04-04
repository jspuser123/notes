## Kubernetes Native Deployment on EC2
Create 2 instances:

1 × Master Node

1 × Worker Node

### Recommended:

OS: Ubuntu 22.04

Instance: t3.medium (minimum)

Open ports (Security Group):

22 (SSH)

6443 (Kubernetes API)

30000–32767 (NodePort range)
## IAM Roles:
    Attach roles with permissions for EBS (AmazonEBSCSIDriverPolicy) 
    and EC2 (AmazonEC2FullAccess)
    and ELB (ElasticLoadBalancingFullAccess).
## 2. COMMON SETUP (Run on BOTH Master & Worker)🧱 
```sh
# Update system
sudo apt update && sudo apt upgrade -y

# Disable swap (required)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter


cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

# Install Docker
```sh
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo docker login -u <username> -p <password>
```
# Install Kubernetes
```sh
sudo apt install -y apt-transport-https ca-certificates curl

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
## 3. MASTER NODE SETUP 🧠
```sh
#Run ONLY on master:
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
#👉 SAVE THIS — needed for worker
kubeadm join <MASTER-IP>:6443 --token xxxx --discovery-token-ca-cert-hash sha256:xxxx
```
## 4.Install Network Plugin (Calico)
```sh
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

## 5. WORKER NODE SETUP 🧠
```sh
#Run ONLY on worker:
sudo kubeadm join <MASTER-IP>:6443 --token xxxx --discovery-token-ca-cert-hash sha256:xxxx
```

## 6.check status
```sh
kubectl get nodes

#NAME       STATUS   ROLES    AGE   VERSION
#master     Ready    master   10m   v1.24.0
#worker     Ready    <none>   10m   v1.24.0
```
## 7.storage
```sh
# create folder use local storage
sudo mkdir -p /data
sudo chmod 777 /data
```
## 8.deploy my app
```sh
kubectl apply -f ecom.yaml
kubectl get pods
kubectl get svc
kubectl get nodes -o wide
kubectl describe pod <pod-name>
kubectl logs -f <pod-name>

```
## SUMMARY ✅
***

Master: kubeadm init

Worker: kubeadm join

Fix path → /data

Deploy YAML

Access via NodePort

***

## If you want next level 🚀
***
✅ Convert this to PersistentVolume (EBS)

✅ Setup LoadBalancer (AWS ELB)

✅ Add Ingress + domain + HTTPS
***
## 1 Convert this to PersistentVolume (EBS → PV)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3          # Volume type (gp2, gp3, io1, io2)
  fsType: ext4       # File system type (optional, default is ext4)
  encrypted: "true"
```
***kubectl apply -f storageclass.yaml***

## 2. Create PersistentVolumeClaim (PVC) 
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ecom-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 10Gi
```
**kubectl apply -f pvc.yaml**
## 3.Update Deployment (IMPORTANT)
```sh
volumeMounts:
- name: data-volume
  mountPath: /data

volumes:
- name: data-volume
  persistentVolumeClaim:
    claimName: ecom-pvc
```
***kubectl apply -f ecom.yaml***

## PART 1 — Install Ingress Controller (NGINX)
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```
***kubectl get pods -n ingress-nginx***
## PART 2 Extenal ip
```sh
kubectl get svc -n ingress-nginx
#NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
#ingress-nginx-controller   LoadBalancer   10.100.20.13   <pending>     80:31035/TCP,443:30874/TCP   2m
```
## PART 3 — Install Cert-Manager
```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

``` 
***kubectl get pods -n cert-manager***
# Create Certificate
```sh
openssl req -x509 -newkey rsa:4096 -keyout my-key.key -out my-cert.crt -days 365 -nodes -subj "/CN=your-ip-address"
kubectl create secret tls ecom-tls --cert=my-cert.crt --key=my-key.key
# delete keys
# kubectl delete secret ecom-tls 
```

## part 4 — Create ClusterIssuer (Let’s Encrypt)
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```
***kubectl apply -f issuer.yaml***

## part 5 Create Ingress for your app
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecom-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
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
**kubectl apply -f ingress.yaml**

## part 6 Check your app
```sh
kubectl get ingress
kubectl get certificate
## comman error
kubectl describe certificate
kubectl describe challenge
```

### pricing
***TOTAL COST (Your Setup)📊
🔻 Option A: Cheapest (Not recommended)

2 × t3.micro = ₹1200

Storage = ₹130

IP = ₹300
👉 Total ≈ ₹1600/month

⚠️ Will likely crash / unstable

✅ Option B: Practical Setup (Recommended)

1 × t3.medium (master)

1 × t3.small (worker)

Cost:

EC2 = ₹2400 + ₹1200 = ₹3600

Storage = ₹130

IP = ₹300

👉 Total ≈ ₹4000/month***