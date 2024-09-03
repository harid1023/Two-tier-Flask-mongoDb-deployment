# "Two-tier Flask MySql deployment"

# 1. Our first task is to dockerize both tiers using docker-compose:

1.1 First install Docker and clone the repository required for execution on the AWS EC2 instance:
```
sudo apt update
sudo apt install docker.io -y
sudo chown $USER /var/run/docker.sock
git clone https://github.com/ghulk123/two-tier-flask-app.git
```
![Screenshot (325)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/900e82c4-799b-4d64-898b-1ef38c58849d)

1.2 Build the flask image with the help of Dockerfile and pull the MySQL image from DockerHub, also push flaskapp image
   Note: Edit inbound rules on port 5000 in EC2 instance as our flask application runs on that port.  
```
docker build -t flaskapp .
docker pull mysql:5.7
docker images
docker login
docker tag flaskapp:latest ghulk047/flaskapp:latest
docker push ghulk047/flaskapp:latest
```
![Screenshot 2024-02-15 011044](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/ea229dea-ed5e-4203-96f3-29ac691740ed)

1.3 Now we have to install docker-compose on instance:
```
sudo apt install docker-compose -y
vi docker-compose.yml
docker-compose up -d
```
![Screenshot (326)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/4dc45b53-47c3-46d3-a90b-ec32e25c6fca)

![Screenshot (327)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/9e019008-9683-4adb-a279-f735bee06733)

1.4 Our application is accessible on port <public-ip>:5000
![Screenshot (328)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/595e6ff3-5cf5-4619-b4fb-d277f59a3d53)

# 2. Our second task is to setup Kubernetes Cluster (Kubeadm):
Create two instances one for control and other as a worker node on AWS EC2

2.1 Run the following commands on both the master and worker nodes to prepare them for kubeadm.
```
# using 'sudo su' is not a good practice.
sudo apt update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo apt install docker.io -y

sudo systemctl enable --now docker # enable and start in single command.

# Adding GPG keys.
curl -fsSL "https://packages.cloud.google.com/apt/doc/apt-key.gpg" | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/kubernetes-archive-keyring.gpg

# Add the repository to the sourcelist.
echo 'deb https://packages.cloud.google.com/apt kubernetes-xenial main' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update 
sudo snap install kubeadm --classic
sudo snap install kubectl --classic
sudo snap install kubelet --classic
```
![Screenshot (330)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/f69f6905-60ca-406a-87b0-7bc59ac73fd8)

2.2 Initialize the Kubernetes master node, After successfully running, our Kubernetes control plane will be initialized successfully with all the required services for the control plane:
```
sudo kubeadm init
```
![Screenshot (331)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/45ca84c9-94f2-4e1a-8bd6-cef1de708e38)

2.3 Set up local kubeconfig (both for root user and normal user):
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
![Screenshot (333)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/725362d7-2efe-4bbb-a4fa-fd148eabc53b)

2.4 Apply Weave network:
```
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```
![Screenshot (334)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/61795cc5-5630-4f42-ace7-f8b22d32f090)

2.5 Generate a token for worker nodes to join:
```
sudo kubeadm token create --print-join-command
```
![Screenshot (335)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/b5b1af98-854a-4801-94f9-88ca38578808)

2.6 Expose port 6443 in the Security group for the Worker to connect to Master Node:
![Screenshot (336)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/38db6af0-a09f-4eaf-95ff-60202225423f)

2.7 Run the following commands on the worker node.
```
sudo kubeadm reset pre-flight checks
```
![Screenshot (337)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/d205dcc2-97ba-41f5-a21a-2253eddfeb53)

2.8 Paste the join command you got from the master node and append --v=5 at the end. Make sure either you are working as sudo user or use sudo before the command:
```
sudo kubeadm join 172.31.94.225:6443 --token dbn6ea.szatcr3dc9kkju61     --discovery-token-ca-cert-hash sha256:c2e5226a7558fae4bc9a904820ab04fe9acecb40ece1c19ac474a89e09916731 --v=5
```
![Screenshot (338)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/08423f7a-dfc5-4b75-ac39-286edfed0514)

# 3. Our third task is to deploy our two-tier flask application on Kubernetes Cluster (Kubeadm) prevoisuly created:

3.1 First clone the flask app repository on master node
```
git clone https://github.com/ghulk123/two-tier-flask-app.git
```

3.2 Now we have to apply deployment and service yaml for the flask application also have to apply deployment and service file for MySql along with its PersitentVolume and PerisitentVolumeClaim
Note : Open port 30007 on master noder
```
cd two-tier-flask-app/k8s
kubectl apply -f two-tier-app-deployment.yml
```
![Screenshot (342)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/7e062942-3d5f-49b3-8a8c-eaa06256e86a)

```
kubectl apply -f two-tier-app-svc.yml
```
![Screenshot (341)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/945761cb-4ee4-44f7-af54-a6647e0a966d)

```
kubectl apply -f mysql-deployment.yml
```
![Screenshot (343)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/8dfd18f4-8d46-45b7-80b6-622355d1bf12)

```
kubectl apply -f mysql-svc.yml
```
![Screenshot (346)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/d6fe954b-c3b7-4035-b283-e3c01c947b84)

```
kubectl apply -f mysql-pv.yml
```
![Screenshot (345)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/ea03bb54-c79f-40ed-bbb3-187cd135eaa5)

```
kubectl apply -f mysql-pvc.yml
```
![Screenshot (344)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/b9c74798-8b6f-4633-805b-f7809c270ef3)

3.3 We can access the application on <public-ip of worker node>:30004 but we get error msg "Table 'mydb.messages' doesn't exist" , we can solve it by entering into the container of mysql and create table there 

```
sudo docker exec -it <container id of mysql pod running in worker node> bash
mysql -u root -p # Enter password
show databases;
use mydb;
CREATE TABLE messages (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message TEXT
);
```
![Screenshot (347)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/8683295b-00cf-4da0-8285-3ef3c80f3099)

![Screenshot (348)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/c36a61aa-1c07-4954-a3be-d8b4e8005690)

3.4 Finally our application is accessible
![Screenshot (349)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/ae2bffd2-dc0c-4b1c-a671-79e1cc56d4a9)

# 4. Our fourth task is to package the Kubernetes manifest files using HELM and deployed the application on K8s cluster.

4.1 How to Install helm in Ubuntu
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```
![Screenshot (350)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/67839c63-92ce-4a31-a144-7ee6e7f7dbe4)

4.2 Create "mysql-chart" and modify templates and values in that
```
helm create mysql-chart
vi values.yml
```
![Screenshot (353)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/d9f478ff-4562-4fac-a426-0ebe4ad4b151)

```
vi templates/deployment.yaml
```
![Screenshot (354)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/7349ec6c-9a52-4cd7-ad7c-46acf3585da0)

4.3 Now we have to package the mysql chart and install mysql chart
```
helm package mysql-chart
helm install mysql-chart ./mysql-chart
```

# 5. Our fifth task is to deploy our two tier flask application on AWS managed Elastic Kubernetes Services(EKS)

5.1 Install and configure the AWS Command Line Interface (CLI) on your local.
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
```
![Screenshot (356)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/a2888a72-e255-44a4-b191-f11eb0f0bff5)

5.2 Create an IAM User:
 1. Go to the AWS IAM console.
 2. Create a new IAM user named "eks-admin."
 3. Attach the "AdministratorAccess" policy to this user.

5.3 Create Security Credentials:
 1. After creating the user, generate an Access Key and Secret Access Key for this user.

5.4 Configure the AWS CLI with the Access Key and Secret Access Key:
```
aws configure
```
![Screenshot (357)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/703bf1f5-51f7-48c2-b045-4fa73a871f92)

5.5 Install kubectl:
```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short –client
```
![Screenshot (358)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/e24a9b79-20b8-4820-aebb-dc00b11ee80d)

5.6 Install eksctl:
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
![Screenshot (359)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/311b0aa1-e379-4ecf-ae95-04cd4f4750b3)

5.7 Use eksctl to create the EKS cluster
```
eksctl create cluster --name ghulk-cluster --region us-east-1 --node-type t2.small --nodes-min 2 --nodes-max 2
```
![Screenshot (360)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/bc0b30f3-7f0b-4421-9ec9-ae4a084870e9)

5.8 Create manifest for deployment, secrets, configmaps and service yaml for our both tiers by cloning the repository:
```
git clone https://github.com/ghulk123/two-tier-flask-app.git
```
![Screenshot (360)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/a9e99e87-ac9a-47dc-84dd-3f0984de2cd0)

5.9 Apply all the manifest files:
```
kubectl apply -f mysql-secrets.yml -f mysql-configmap.yml -f mysql-svc.yml -f mysql-deployment.yml
kubectl apply -f two-tier-app-deployment.yml -f two-tier-app-svc.yml
```
![Screenshot (361)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/38ffbe18-21a6-4fad-8a1d-c227ea23b2c1)

5.10 Access the application using Load Balancer URL
```
kubectl get svc
```
http://a6397f9ad1f2448d28912a092a10bb81-2104183748.us-east-1.elb.amazonaws.com/

![Screenshot (362)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/d988e844-14f3-46cf-bff0-40b9844f69a0)

![Screenshot (363)](https://github.com/ghulk123/Two-tier-Flask-mongoDb-deployment/assets/104766246/1381f1e1-3774-4767-9df3-43a455858467)






