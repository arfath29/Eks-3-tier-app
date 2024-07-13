## 1. Setting up EC2 Instance
### 1.Create an EC2 Instance:

### Type: t2.micro
### OS: Ubuntu
## 2.Install Docker:

```bash
sudo apt update
sudo apt install docker.io -y
sudo chown $USER /var/run/docker.sock
```
## 2. Cloning the Repository and Setting Up Frontend
### 1.Clone the Repository:

```bash
git clone https://github.com/arfath29/Eks-3-tier-app
cd Eks-3-tier-app
```
### 2.Create a Dockerfile for Frontend:

```bash
nano Dockerfile
```
```bash
dockerfile
Copy code
FROM node:14
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "start"]
```
```bash
docker build -t your_dockerhub_username/frontend .
```

## Deploy Backend in Kubernetes
### Create Backend Dockerfile and Push to Docker Hub:

```bash
cd ../backend
nano Dockerfile
```
### dockerfile:

```bash
FROM node:14
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["node", "index.js"]
```
### build docker image 
```bash
docker build -t your_dockerhub_username/backend .
```
```bash
docker login
docker push your_dockerhub_username/frontend
docker push your_dockerhub_username/backend
```
### 3.Build and Push Frontend Docker Image to Docker Hub:

### 4.Run Frontend Container Locally (Optional):

```bash
docker run -d -p 3000:3000 your_dockerhub_username/frontend
```
## 3. Install AWS CLI
### 1.Download and Install AWS CLI:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
```
```bash
aws configure
```
## 4. Set Up IAM User for EKS
### 1.Create IAM User:
### Name: eks-user
### Policy: AdministratorAccess
### Create Access Key: Use the key for AWS CLI configuration.

## 5. Install kubectl and eksctl
### Install kubectl:

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```
```bash
sudo chmod +x /usr/local/bin/kubectl

```
### Install eksctl:

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
## 7. Set Up EKS Cluster
### Create EKS Cluster:

```bash
eksctl create cluster --name three-tier-cluster --region ap-south-1 --node-type t2.medium --nodes-min 2 --nodes-max 2
aws eks update-kubeconfig --region ap-south-1 --name three-tier-cluster
kubectl get nodes
```
## 8. Deploy MongoDB in Kubernetes
### Create Namespace:

```bash
kubectl create namespace workshop
```
Create Secrets:

```bash
nano secrets.yml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  namespace: workshop
  name: mongo-sec
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # Three-Tier-Project
  username: YWRtaW4=  # admin
```
### Apply Secrets:

```bash
kubectl apply -f secrets.yml
```
### Deploy MongoDB:

```bash
nano mongo-deploy.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: workshop
  name: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongo
        image: mongo:4.4.6
        command:
          - "numactl"
          - "--interleave=all"
          - "mongod"
          - "--wiredTigerCacheSizeGB"
          - "0.1"
          - "--bind_ip"
          - "0.0.0.0"
        ports:
        - containerPort: 27017
        env:
          - name: MONGO_INITDB_ROOT_USERNAME
            valueFrom:
              secretKeyRef:
                name: mongo-sec
                key: username
          - name: MONGO_INITDB_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mongo-sec
                key: password
```
## Apply MongoDB Deployment:

```bash
kubectl apply -f mongo-deploy.yml
```
## Create MongoDB Service:

```bash
nano mongo-service.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: workshop
  name: mongodb-svc
spec:
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```
### Apply MongoDB Service:

```bash
kubectl apply -f mongo-service.yml
```

## Create Backend Deployment:

```bash
nano backend-deploy.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: workshop
  name: api
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 25%
  selector:
    matchLabels:
      role: api
  template:
    metadata:
      labels:
        role: api
    spec:
      containers:
        - name: api
          image: arfath29/3-tier-app-backend
          ports:
            - containerPort: 3500
          env:
            - name: MONGO_CONN_STR
              value: mongodb://mongodb-svc:27017/todo?directConnection=true
            - name: MONGO_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-sec
                  key: username
            - name: MONGO_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-sec
                  key: password
          livenessProbe:
            httpGet:
              path: /ok
              port: 3500
            initialDelaySeconds: 2
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /ok
              port: 3500
            initialDelaySeconds: 5
            periodSeconds: 5
```
### Apply Backend Deployment:

```bash
kubectl apply -f backend-deploy.yml
```
### Create Backend Service:

```bash
nano backend-service.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: workshop
  name: api
spec:
  selector:
    role: api
  ports:
    - protocol: TCP
      port: 3500
      targetPort: 3500
```
### Apply Backend Service:

```bash
kubectl apply -f backend-service.yml
```
## 9. Deploy Frontend in Kubernetes
### Create Frontend Deployment:

```bash
nano frontend-deploy.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: workshop
  name: frontend
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 25%
  selector:
    matchLabels:
      role: frontend
  template:
    metadata:
      labels:
        role: frontend
    spec:
      containers:
        - name: frontend
          image: arfath29/3-tier-app-frontend
          ports:
            - containerPort: 3000
          env:
            - name: REACT_APP_BACKEND_URL
              value: "http://api:3500"
### apply frontend-deploymrnt:  
```
```bash
kubectl apply -f frontend-deployment
```
### Create Frontend Service:

```bash
nano frontend-service.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: workshop
  name: frontend
spec:
  selector:
    role: frontend
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
```
### Apply Frontend Service:

```bash
kubectl apply -f frontend-service.yml
```

## install aws loadbalancer :
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```
```bash
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicyForEKS --policy-document file://iam_policy.json
```
```bash
eksctl utils associate-iam-oidc-provider --region=ap-south-1 --cluster=three-tier-cluster --approve
```
```bash
eksctl create iamserviceaccount --cluster=three-tier-cluster --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRoleForEKS --attach-policy-arn=arn:aws:iam::626072240565:policy/AWSLoadBalancerControllerIAMPolicyForEKS --approve --region=ap-south-1
```

## deploy aws load balancer controller:

```bash
sudo snap install helm --classic
```
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=three-tier-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```
## 11. Create Ingress for Routing
### Create Ingress Resource:

```bash
nano ingress.yml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mainlb
  namespace: workshop
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
spec:
  ingressClassName: alb
  rules:
    - host: <public_ip>.nip.io
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 3500
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 3000
```
### Apply Ingress Resource:

```bash
kubectl apply -f ingress.yml
```
## 12. Verify MongoDB Connection
### Get MongoDB Pod:

```bash
kubectl get pods -n workshop
```
### Connect to MongoDB Pod:

```bash
kubectl exec -it <mongo-pod-name> -n workshop -- /bin/bash
```
```bash
mongo
show dbs
use todo
db.tasks.find()
```
