Setup Infra for Boardgame app

Jenkins-Server - req-tools: java-21-openjdk
Nexus server  - docker
SonarQube      - docker
Kubernetes cluster - docker/containerd.io

To add ports in firewall in required machine. 
# Allow Jenkins port
sudo firewall-cmd --permanent --add-port=8080/tcp

# Allow Nexus port
sudo firewall-cmd --permanent --add-port=8081/tcp

# Allow SonarQube port
sudo firewall-cmd --permanent --add-port=9000/tcp

# Allow Application port (on K8s node)
sudo firewall-cmd --permanent --add-port=8080/tcp

# Allow all Kubernetes ports (common ports for k8s components)
# Kubernetes uses a wide range of ports, you can open some key ones:

sudo firewall-cmd --permanent --add-port=6443/tcp   # Kubernetes API server
sudo firewall-cmd --permanent --add-port=2379-2380/tcp  # etcd server client API
sudo firewall-cmd --permanent --add-port=10250-10255/tcp  # Kubelet, kube-scheduler, controller-manager
sudo firewall-cmd --permanent --add-port=30000-32767/tcp  # NodePort Services range

# Reload firewall to apply changes
sudo firewall-cmd --reload

# Verify open ports
sudo firewall-cmd --list-ports


## Install Docker
Ref: https://docs.docker.com/engine/install/rhel/

sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
sudo systemctl status docker
sudo docker run hello-world


## Installing Jenkins 
REF: https://www.jenkins.io/doc/book/installing/linux/#red-hat-centos

# Add required dependencies for the jenkins package
sudo yum install -y fontconfig java-21-openjdk wget
java --version

sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade
sudo yum install -y jenkins

sudo systemctl daemon-reload
sudo systemctl enable Jenkins
sudo systemctl start Jenkins
sudo systemctl status jenkins


Get Initial admin password:
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

Open your browser
 http://<<Server-IP>>:8080

## Install plugins 
jdk- eclipse Temurin installer
maven - config file provider
      Pipeline Maven Integration
      maven Integration
Sonar- SonarQube Scanner
Docker- Docker
        Docker pipeline
Kubernetes - kubernetes
        kubernetes cli
        kuberntes Client API
        kubernetes Credentials





## Install Trivy
REF: https://trivy.dev/v0.18.3/installation/
cat <<EOF | sudo tee /etc/yum.repos.d/trivy.repo > /dev/null
[trivy]
name=Trivy repository
baseurl=https://aquasecurity.github.io/trivy-repo/rpm/releases/\$releasever/\$basearch/
gpgcheck=0
enabled=1
EOF

sudo yum -y update
sudo yum -y install trivy
trivy --version

## Nexus server   - docker
Create Nexus on docker container
docker pull sonatype/nexus3
docker run -d --name nexus -p 8081:8081 sonatype/nexus3
docker ps
http://<machine0-IP:8081>
Get Nexus user & password
user: admin
pass: docker exec -it nexus /bin/bash -c "cat /opt/sonatype/sonatype-work/nexus3/admin.password"


## SonarQube      - docker
Create SonarQube on docker container 
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
docker ps

http://<machine0-IP:9000>
Sonar default user & password
user: amdin
pass: amdin

To use docker all user change permission. 
sudo usermod -aG docker jenkins
sudo usermod -aG docker sonar
sudo usermod -aG docker nexus
sudo usermod -aG docker $USER

sudo systemctl restart jenkins
sudo systemctl restart sonarqube
sudo systemctl restart nexus

For login users, log out and log back in or use:
newgrp docker

Check Current Group Access
getent group docker
OR
chmod 666 /var/run/docker.sock  # Avoid on production

## Create systemd unit files for your Nexus and SonarQube Docker containers
## nexus service
cat <<EOF >> /etc/systemd/system/docker.nexus.service
[Unit]
Description=My container Nexus server
After=docker.service
Requires=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker start -a nexus
ExecStop=/usr/bin/docker stop nexus
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

## sonar service
cat <<EOF >> /etc/systemd/system/docker.sonar.service
[Unit]
Description=My container SonarQube server
After=docker.service
Requires=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker start -a sonar
ExecStop=/usr/bin/docker stop sonar
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

## To verify service files
ll  /etc/systemd/system/ | grep -E "docker.nexus.service|docker.sonar.service"

## To Enable and Start nexus & sonar Services:
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now docker.nexus.service
sudo systemctl status docker.nexus.service
sudo systemctl enable --now docker.sonar.service
sudo systemctl status docker.sonar.service



## Kubernetes cluster - docker/containerd.io
ping google.com
dnf update -y
ifconfig
   35  dnf update -y
   36  ping -c 4 192.168.95.128
   37  ping -c 4 192.168.95.129
   38  cat <<EOF>> /etc/hosts

192.168.95.128 control-plane
192.168.95.129 worker-01
EOF

   39  cat /etc/hosts
   40  ping -c 4 control-plane
   41  ping -c 4 worker-01
   42  sudo setenforce 0
   43  sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
   44  sudo swapoff -a
   45  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
   46  cat /etc/fstab
   47  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
        overlay
        br_netfilter
       EOF

   48  sudo modprobe overlay
   49  sudo modprobe br_netfilter
   50  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

   51  sudo sysctl --system
   52  lsmod | grep br_netfilter
   53  lsmod | grep overlay
   54  sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

    sudo dnf -y install dnf-plugins-core
    sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
    sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

    sudo systemctl enable --now containerd.service
    sudo systemctl status containerd.service
 
 sudo mkdir -p /etc/containerd
   67  containerd config default | sudo tee /etc/containerd/config.toml
   68  sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
   69  sudo sed -n '/SystemdCgroup /p' /etc/containerd/config.toml
   70  sudo systemctl daemon-reload
   71  sudo systemctl restart containerd.service
   73  sudo systemctl status containerd.service

   yum install -y wget vim-enhanced git curl

# kubeadm, kubectl , kubelet repo version-v1-30
   # Ref: https://v1-30.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
   80  cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

   81  yum update -y
   82  sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
   83  sudo systemctl enable --now kubelet
   84  sudo systemctl start --now kubelet
   85  sudo systemctl status kubelet
   86  kubeadm config images pull
   87  sudo kubeadm init --ignore-preflight-errors=all
   88  mkdir -p $HOME/.kube
   89  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   90  sudo chown $(id -u):$(id -g) $HOME/.kube/config
   # Calico 
   #Ref: https://docs.tigera.io/calico/3.27/getting-started/kubernetes/self-managed-onprem/onpremises
   91  kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

   93  kubectl get nodes
   94  kubectl get pod -n kube-system
   95  watch kubectl get pod -n kube-system


Create github repository
gaurav1517/BoardGame

Create Access Token 
Setting > Devloper Settings > Token Classic >Generate New Token > name: github-token > Select scope (all select) > Genearte Token
Copy token & past notepad because it page refresh it's gone. 


Push the source code to github
git init
git add .
git commit -m "source code"
git config --global user.name "Gaurav Chauhan"
git config --global user.email gaurav.cloud000@gmail.com
git branch -M main
git remote add origin https://github.com/gaurav1517/<repository-name>.git
git push origin -u main

Jenkinsfile

tools {
  jdk 'jdk-17'
  dockerTool 'docker'
  maven 'maven'
}

Dashboard
Manage Jenkins
Credentials
System
Global credentials (unrestricted)
New credentials
Kind

Username with password
Scope
?

Global (Jenkins, nodes, items, all child items, etc)
Username
?
Gaurav1517

Treat username as secret
?
Password
?
••••••••••••••••••••••••••••••••••••••••
ID
?
git-cred
Description
?
git-cred
Create


Create webhook  sonarqube 
Create Webhook
All fields marked with * are required
Name*
jenkins
URL*
http://192.168.70.135:8080/sonar-webhook/


Configure RBACK

namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webapps

sericeAccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
 name: jenkins
 namespace: webapps

kubectl create -f namespace.yaml
kubectl  get ns | grep webapps
kubectl create -f serviceAccount.yaml
 kubectl  get serviceaccounts -n webapps


REF: https://github.com/jaiswaladi246/EKS-Complete/blob/main/Steps-eks.md

Create Role
role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

kubectl create -f role.yaml
 kubectl  get role -n webapps
 kubctl describe role app-role -n webapps


Bind the role to service account

role-bind.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins 

kubectl create -f role-bind.yaml
 kubectl get rolebindings -n webapps

Generate token using service account in the namespace
Create Token 
REF:
https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#:~:text=To%20create%20a%20non%2Dexpiring,with%20that%20generated%20token%20data.


vim secret.yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  namespace: webapps 
  annotations:
    kubernetes.io/service-account.name: jenkins

kubectl create -f secret.yaml -n webapps
 kubectl get secrets -n webapps
kubectl describe secret mysecretname -n webapps



deployment-service.yaml

apiVersion: apps/v1
kind: Deployment # Kubernetes resource kind we are creating
metadata:
  name: boardgame-deployment
spec:
  selector:
    matchLabels:
      app: boardgame
  replicas: 2 # Number of replicas that will be created for this deployment
  template:
    metadata:
      labels:
        app: boardgame
    spec:
      containers:
        - name: boardgame
          image: gchauhan1517/boardgame:latest # Image that will be used to containers in the cluster
          imagePullPolicy: Always
          ports:
            - containerPort: 8080 # The port that the container is running on in the cluster


---

apiVersion: v1 # Kubernetes API version
kind: Service # Kubernetes resource kind we are creating
metadata: # Metadata of the resource kind we are creating
  name: boardgame-ssvc
spec:
  selector:
    app: boardgame
  ports:
    - protocol: "TCP"
      port: 8080 # The port that the service is running on in the cluster
      targetPort: 8080 # The port exposed by the service
  type: LoadBalancer # type of the service.


Configure email
Create app password

Jnekins > Mange Jenkins > system
Extended E-mail Notification
SMTP server
smtp.gmail.com
SMTP Port
465
Advanced
Edited
Credentials

gaurav.mau854@gmail.com/****** (mail-cred)
Add

Use SSL

Use TLS

Use OAuth 2.0



E-mail Notification
SMTP server
smtp.gmail.com
Default user e-mail suffix
?
Advanced
Edited

Use SMTP Authentication
?
User Name
gaurav.mau854@gmail.com
Password
•••••••••••••••••••

Use SSL
?

Use TLS
SMTP Port
?
465
Reply-To Address
Charset
UTF-8

Test configuration by sending test e-mail
Test e-mail recipient
gaurav.mau854@gmail.com
Test configuration
Email was successfully sent


Add port 465 for email notificatio on jenkins server. 
firewall-cmd --add-port=465/tcp --permanent
firewall-cmd --reload
firewall-cmd --list-ports

