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

# 2. Our second task is to setup Kubernetes Cluster (Kubeadm)

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
