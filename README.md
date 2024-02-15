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
sudo apt install kubeadm=1.20.0-00 kubectl=1.20.0-00 kubelet=1.20.0-00 -y
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

3.3 We can access the application on <public-ip of worker node>:30004
