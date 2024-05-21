## Three tier app deployment on AWS EKS cluster
This project is help to deploy the the end to end three tier app based on node, MongoDB. Deployment to docker containerize the app and build on AWS EKS cluster. To give the cluster services to outside world user weused Ingress service which helps to connect outside the cluster. For balacing the load of the ELB(elastic load balancer) in our deployment.


## Prerequisites
- Basic knowledge of Docker, and AWS services.
- An AWS account with necessary permissions.

  
### Step 1: IAM Configuration in AWS/ Microsoft extra ID in azure
- In IAM Create a user `eks-admin` with `AdministratorAccess` and 'Database' rights.
- Generate Security Credentials: Access Key and Secret Access Key.

### Step 2: Create a server
- Launch an Ubuntu instance in your favourite region (eg. region `us-west-2`). and used t2-micro type for free tier.
- SSH into the instance from your local machine.
- check the video https://www.youtube.com/watch?v=0Gz-PUnEUF0
- switch the user that we create
```shell
sudo su <user_name>
```

### Step 3: Install AWS CLI v2
``` shell
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
aws configure       #Enter here Access Key and Secret Access Key after run the command.
```

### Step 4: Install Docker and create docker image
``` shell
sudo apt-get update
sudo apt install docker.io
docker ps
sudo chown $USER /var/run/docker.sock
```
- Go to the backend directory you can find a docker file and run
``` shell
docker build -t <repo_url>:my-tag <path_to_Dockerfile>
docker tag my-image-name <repository_uri>:my-tag
```
- push the created image in aws ECR(elastic container registry) for that you need to create repo in AWS ECR first.
``` shell
aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker push <repository_uri>:my-tag
docker run -p 8080:80 <repository_uri>:my-tag
```
- Do the same with frontend

### Step 5: Install kubectl
Note :- AWS EKS is not in aws free tier.
``` shell
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

### Step 6: Install eksctl
``` shell
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

## Create nasmespame
kubectl create Namespace to isolate the resources and after that switch to the myapp namespace after that all resource create in this namespace
```shell
kubectl create ns myapp                                         
kubectl config set-context --current --namespace myapp
```

### Step 7: Setup EKS Cluster
``` shell
eksctl create cluster --name three-tier-cluster --region us-west-2 --node-type t2.medium --nodes-min 2 --nodes-max 2
aws eks update-kubeconfig --region us-west-2 --name three-tier-cluster
kubectl get nodes
```

### Step 8: Run Manifests
- Go to the kubernetses manifest files directory.
- Run the commands to create the nodes
``` shell
kubectl create namespace myapp
kubectl apply -f . -n myapp
```

### Step 9: Install AWS Load Balancer
``` shell
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
eksctl utils associate-iam-oidc-provider --region=us-west-2 --cluster=three-tier-cluster --approve
eksctl create iamserviceaccount --cluster=three-tier-cluster --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::626072240565:policy/AWSLoadBalancerControllerIAMPolicy --approve --region=us-west-2
```

### Step 10: Deploy AWS Load Balancer Controller
``` shell
sudo snap install helm --classic
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=my-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl apply -f full_stack_lb.yaml         #apply file for ingress routiung
```

### Cleanup
- To delete the EKS cluster:
``` shell
eksctl delete cluster --name three-tier-cluster --region us-west-2
```

