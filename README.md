### 1. Create a developer server, add ssh key to the github account, clone project files, push files to the repository.

 ### 2. Create Access-Secret key for AWS policies Go to repository settings > secrets and variables > actions >

![image](https://github.com/user-attachments/assets/14c59c5f-26e3-4698-b7bd-6afb124dcd8c)


### 3. Create new ecr repository in aws : git-repo (you can give any name)

### 4. Create eks-server and install kubectl and eksctl
## 5. *EKS-Cluster*

###  *Create an EC2 instance EKS-mgr   Ubuntu t2.micro All traffic 1x10*
### *Then create a role and give full access to Elasticcontainer full access ,amazonekscluster policy and IAM full*
### *unzip file*
 - apt install unzip -y
 - curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
 - unzip awscliv2.zip
### *install aws*
 - sudo ./aws/install
### *configure aws*
 - aws configure

(*Region without a,b and in output format enter table*)
   
### *Install EKS Tool*
 - curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
 - sudo mv /tmp/eksctl /usr/local/bin
 - eksctl version
### *Install Kubectl*
 
 -  curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
 -  sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
 -  kubectl version --client
### *Create EKS Cluster*
 -  eksctl create cluster --name my-cluster --region region-code --version 1.29 --vpc-public-subnets subnet-ExampleID1,subnet-ExampleID2 --without-nodegroup

 (*Here in this case just add the region us-west-1 instead of region code, two empty subnets from vpc < subnet instead of ExampleID1 and ID2*)

   ### *Creating node3 group*
   
  - ssh-keygen
    
  eksctl create nodegroup \
  --cluster my-cluster \
  --region us-east-2 \
  --name my-node-group \
  --node-ami-family Ubuntu2004 \
  --node-type t2.small \
  --subnet-ids subnet-086ced1a84c94a342,subnet-01695faa5e0e61d97 \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 4 \
  --ssh-access \
  --ssh-public-key /root/.ssh/id_rsa.pub
    
    (*Change the subnet here as previously done*)
    (*Change the region here*)

```yml
vim deployment.yml
```

 ### For deployment.yml paste:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app1
  namespace: default
  labels:
    app: my-app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app1
  template:
    metadata:
      labels:
        app: my-app1
    spec:
      containers:
      - name: my-app1
        image: 288761750357.dkr.ecr.ap-southeast-1.amazonaws.com/mansi-30:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```
```yml
# vim service.yml
```
### 6. For service.yml paste:
```yml
apiVersion: v1
kind: Service
metadata:
  name: regapp-service1
  labels:
    app: my-app1
spec:
  selector:
    app: my-app1

  ports:
    - port: 8080
      targetPort: 8080

  type: LoadBalancer
```
### 7.In github repository create .github/workflows/deploy.yml and paste:
```yml

name: Deploy to ECR

on:
 
  push:
    branches: [ main ]

env:
  ECR_REPOSITORY: git-repo
  EKS_CLUSTER_NAME: jitendra-cluster 
  AWS_REGION: ap-south-1

jobs:
  
  build:
    
    name: Deployment
    runs-on: ubuntu-latest

    steps:

    - name: Set short git commit SHA
      id: commit
      uses: prompt/actions-commit-hash@v2

    - name: Check out code
      uses: actions/checkout@v2
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven
    - name: Build with Maven
      run: mvn -B package --file pom.xml
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{env.AWS_REGION}}

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build, tag, and push image to Amazon ECR
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}        
        IMAGE_TAG: ${{ steps.commit.outputs.short }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:latest -f Dockerfile .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

    - name: Update kube config
      run: aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_REGION

    - name: Deploy to EKS
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}        
        IMAGE_TAG: ${{ steps.commit.outputs.short }}
      run: |
    
        kubectl apply -f regapp-deploy.yml
        kubectl apply -f regapp-service.yml
```
and run the file
