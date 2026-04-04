# Prepare AWS Infrastructure

## Launch EC2 Instances:
    1 master node (t3.small or larger).

    1–2 worker nodes (t3.micro/t3.small).
## networking
    Configure Security Groups: allow SSH (22), K3s API (6443), and NodePort/Ingress traffic.
    allow HTTPS (443) http 
    allow local subnet (10.0.0.0/16) all traffic ping node and master
## IAM Roles:
    Attach roles with permissions for EBS (AmazonEBSCSIDriverPolicy) 
    and EC2 (AmazonEC2FullAccess)
    and ELB (ElasticLoadBalancingFullAccess).
### Attach roles with permissions for Elb  json
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:CreateLoadBalancer",
                "elasticloadbalancing:DeleteLoadBalancer",
                "elasticloadbalancing:DescribeLoadBalancers",
                "elasticloadbalancing:ModifyLoadBalancerAttributes",
                "elasticloadbalancing:CreateTargetGroup",
                "elasticloadbalancing:DescribeTargetGroups",
                "elasticloadbalancing:RegisterTargets",
                "elasticloadbalancing:DeregisterTargets"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVpcs",
                "ec2:DescribeSubnets",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeInstances",
                "ec2:DescribeNetworkInterfaces",
                "ec2:CreateSecurityGroup",
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:RevokeSecurityGroupIngress",
                "ec2:AttachNetworkInterface"
            ],
            "Resource": "*"
        }
    ]
}

```
## update
```sh
sudo apt update && sudo apt upgrade -y
```
# install docker worker node
```sh

sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo docker login -u <username> -p <password>
```
## or this way
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl start docker
```

#  Install K3s on Master Node
```sh
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes
```
# sudo not use cmd
```sh
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=$HOME/.kube/config
kubectl get nodes
# or
echo "export KUBECONFIG=\$HOME/.kube/config" >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
```

#  Retrieve the join token:
```sh
sudo cat /var/lib/rancher/k3s/server/node-token
```
#  Join the worker nodes to the cluster:
```sh
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_NODE_PUBLIC_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
sudo k3s kubectl get nodes

```
# Install the AWS EBS CSI driver:
```sh
curl -sfL https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/install-aws-ebs-csi-driver.sh | sh -
```
or 
### helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver
```sh
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver

helm repo update
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver -n kube-system
```
# Install Network Plugin (Calico)
```sh
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

# 6. storage pv
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
# Persistent manuval Volume Claim (PVC)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ecom-pv
spec:
  capacity:
    storage: 15Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ebs-sc
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-003be9a6612d1af43  # Replace with actual EBS volume ID
```
or
# Persistent Volume Claim (PVC)
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


# deploy my app (hostPath volume) config
```sh
 volumeMounts:
      - name: data-volume
        mountPath: /data
    volumes:
    - name: data-volume
      hostPath:
        path: /mnt/data
        type: Directory
  ```
  ***to change ebs-volume***
  ```sh
 volumeMounts:
      - name: ebs-volume
        mountPath: /data
    volumes:
      - name: ebs-volume
        persistentVolumeClaim:
          claimName: ecom-pvc
```
# deploy my app (yaml)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ecom-env
data:
  SECRET_KEY: "test123321"
  DATABASE_URL: "sqlite:////data/db_this_year.sqlite3"
  UPLOAD_FOLDER: "/data/uploads"
  BACKUP_FOLDER: "/data/backup"
  DATABASE_FOLDER: ""
  DB_USER: "admin"
  DB_PASSWORD: "123"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecom-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ecom-app
  template:
    metadata:
      labels:
        app: ecom-app
    spec:
      containers:
      - name: ecom-app
        image: asp6406/ecom-app:1.0
        ports:
        - containerPort: 8888
        envFrom:
        - configMapRef:
            name: ecom-env
        volumeMounts:
            - name: ebs-volume
              mountPath: /data
      volumes:
        - name: ebs-volume
          persistentVolumeClaim:
            claimName: ecom-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: ecom-service
spec:
  type: ClusterIP
  selector:
    app: ecom-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8888
```



# Set Up ELB for Ingress controller
***k3s predefined ingress controller is traefik***
```sh
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.10/docs/content/reference/dynamic/kubernetes-crd-definition.yaml
kubectl get pods -n ingress-traefik
```
### or ingress-nginx
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/aws/deploy.yaml
kubectl get pods -n ingress-nginx
```
# Install Cert-Manager
```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl get pods -n cert-manager
```
# Create Certificate
```sh
openssl req -x509 -newkey rsa:4096 -keyout my-key.key -out my-cert.crt -days 365 -nodes -subj "/CN=your-ip-address"
kubectl create secret tls ecom-tls --cert=my-cert.crt --key=my-key.key
# delete keys
# kubectl delete secret ecom-tls 
```

# Create ClusterIssuer (Let’s Encrypt)
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your@email.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik # or nginx
```
# Expose Ingress for your app (traefik) https
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecom-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/redirect-entry-point: https
    traefik.ingress.kubernetes.io/force-ssl-headers: "true"
spec:
  ingressClassName: traefik  # or nginx
  tls:
  - hosts:
    - morpho.hopto.org # your domain
    secretName: ecom-tls
  rules:
  - host: morpho.hopto.org  # your domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ecom-service # your service name
            port:
              number: 80

```
# Expose Ingress for your app (traefik) http
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecom-ingress
spec:
  ingressClassName: traefik  # or nginx
  rules:
  - host: "morpho.hopto.org"
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
# Check your cammand
```sh
# first command
kubectl apply -f ecom.yaml
kubectl delete -f ecom.yaml
kubectl get pv
kubectl get pvc
kubectl get pods
kubectl get svc
kubectl get nodes -o wide
kubectl describe pod <pod-name>
kubectl logs -f <pod-name>
# exec pod shell cmd
kubectl exec -it <pod-name> -- bash
# copy file
kubectl cp <pod-name>:/path/to/file <local-path>

# kubesystem command
kubectl get pods -n kube-system
kubectl get pods -n kube-system -l app=ebs-csi-driver
## comman error
kubectl get pods -A
kubectl get pods -n ingress-nginx
kubectl get pods -n cert-manager
kubectl get clusterissuer
kubectl get ingress
kubectl get ecom-ingress
kubectl get certificate
kubectl describe certificate
kubectl describe challenge

# restart ingress controller
kubectl rollout restart deployment cert-manager -n cert-manager
kubectl rollout restart deployment nginx-ingress-controller -n ingress-nginx
# port forward
kubectl port-forward service/ecom-service 8080:30088
kubectl port-forward pod/ecom-app-8468885d98-pqbwm 8080:30088
# romove node 
kubectl delete node ip-172-31-46-162

```


# port check
```sh
netstat -tulnp | grep 8080 #PORT CHECK
sudo lsof -i tcp:8080

```

# port kill
```sh
netstat -tulnp | grep 8080 | awk '{print $7}' | cut -d/ -f1 | xargs kill -9
awk 'NR!=1 {print $2}' | xargs kill -9
sudo kill -9 $(lsof -t -i:8080)
```
