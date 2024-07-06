## 1.make ec2 instance with t2.micro and ubuntu

```bash
git clone https://github.com/LondheShubham153/TWSThreeTierAppChallenge
```

```bash
cd frontend 
nano dockerfile
```

```json
FROM node:14
WORKDIR app
COPY package*.json ./
RUN npm install 
COPY . .
CMD ["npm","start"]
```

```bash
sudo apt install docker.io -y
```
```bash
sudo chown $USER /var/run/docker.sock
```
```bash
docker build -t frontend .
```
```bash
docker run -d -p 3000:3000 frontend
```
```bash
docker rm -f <container_name>
```

```bash
cd ~
```
## download aws cli 

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
```
### aws configure

## 2.create a iam user called eks-user
### attach "administrator_access" policy
### create access key

## aws configure:

## 3.create ecr repo frontend "1st"
### push image on ecr 

```bash
cd backend
nano dockerfile
```

```json
FROM node:14
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["node","index.js"]
```

## create new ecr repo for backend "2nd"

```bash
docker build -t backend .
```
### push docker image on ecr

```bash
docker run -d -p 8080:8080 backend
```

```bash
docker logs <container_id>
```
(could not connect to databse because it is not connected to mongodb)

```bash
cd ~
```
## install kubectl:

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

## install eksctl:

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

## setup eks cluster:

```bash
eksctl create cluster --name three-tier-cluster --region us-west-2 --node-type t2.medium --nodes-min 2 --nodes-max 2
aws eks update-kubeconfig --region us-west-2 --name three-tier-cluster
kubectl get nodes
```

## -> go to cloudformation we can a stack 

```bash
cd 3tierrepo

cd mongo/
```

```bash
nano deploy.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: 
  namespace: three-tier
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
      - name: mon
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
        volumeMounts:
          - name: mongo-volume
            mountPath: /data/db
      volumes: 
      - name: mongo-volume
        persistentVolumeClaim:
          claimName: mongo-volume-claim

```
```bash
kubectl create namespace workshop
kubectl apply -f deploy.yml 
```

```bash
nano secrets.yml
```

```yaml
apiVersion: v1
kind: Secret
metadata: 
  namespace: three-tier
  name: mongo-sec
type: Opaque
data:  
  password: cGFzc3dvcmQxMjM=   #Three-Tier-Project
  username: YWRtaW4= #admin
```

```python
kubectl apply -f secrets.yml
kubectl get pods -n workshop
``` 
##(mongodb is now running)

```bash
nano service.yml

```
```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: three-tier
  name: mongodb-svc
spec:
  selector:
    app: mongodb
  ports:
  - name: mongodb-svc
    protocol: TCP
    port: 27017
    targetPort: 27017
```

```bash
kubectl apply -f service.yml (mongo is ready to communicate internally)
kubectl get svc -n workshop
```

```bash
cd ..
cd backend
```

```bash
nano deploy.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: api
  namespace: three-tier
  labels: 
    role: api
    env: demo
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
      imagePullSecrets:
      - name: ecr-registry-secret
      containers:
      - name: api
        image: 407622020962.dkr.ecr.us-east-1.amazonaws.com/backend:latest
        imagePullPolicy: Always
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
        ports:
        - containerPort: 3500
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
          successThreshold: 1
```

```bash
kubectl apply -f deploy.yml
``` 

```bash
nano svc.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: three-tier
spec: 
  ports:
  - port: 3500
    protocol: TCP
  type: ClusterIP
  selector:
    role: api
```

```bash
kubectl apply -f svc.yml (backend)
kubectl get pods -n workshop
kubectl logs <svc_id> 
``` 
### (now it is connectd with mongo)

```bash
cd frontend

nano deploy.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: three-tier
  labels:
    role: frontend
    env: demo
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
      imagePullSecrets:
      - name: ecr-registry-secret
      containers:
      - name: frontend
        image: 407622020962.dkr.ecr.us-east-1.amazonaws.com/frontend:latest
        imagePullPolicy: Always
        env:
          - name: REACT_APP_BACKEND_URL
            value: "http://backend.amanpathakdevops.study/api/tasks"
        ports:
        - containerPort: 3000
```

```bash
kubectl apply -f deploy.yml
```

```bash
nano svc.yml
```

```yaml
apiVersion: v1
kind: Service
metadata: 
  name: frontend
  namespace: three-tier
spec:
  ports:
  - port: 3000
    protocol: TCP
  type: ClusterIP
  selector:
    role: frontend
```

```bash
kubectl apply -f svc.yml
kubectl get pods -n workshop

cd ~
```

## 1. install aws loadbalancer :
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
eksctl utils associate-iam-oidc-provider --region=us-west-2 --cluster=three-tier-cluster --approve
eksctl create iamserviceaccount --cluster=three-tier-cluster --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::626072240565:policy/AWSLoadBalancerControllerIAMPolicy --approve --region=us-west-2

```
## 2. deploy aws load balancer controller:

```bash
sudo snap install helm --classic
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=my-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl apply -f full_stack_lb.yaml
```

    (2.13.30)

cd 3tierrepo/k8s repo
