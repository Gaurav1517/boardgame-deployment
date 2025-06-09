# Boardgame App Infrastructure Setup

This document outlines the infrastructure setup for the **Boardgame App** using Jenkins, SonarQube, Nexus, and a Kubernetes cluster.

---

## 🧰 Prerequisites

- RHEL/CentOS-based systems
- Internet access for downloading packages
- Root or sudo access

---

## 🔥 Firewall Configuration

Allow required ports on all necessary machines:

```bash
# Jenkins
sudo firewall-cmd --permanent --add-port=8080/tcp

# Nexus
sudo firewall-cmd --permanent --add-port=8081/tcp

# SonarQube
sudo firewall-cmd --permanent --add-port=9000/tcp

# Application (Kubernetes NodePort)
sudo firewall-cmd --permanent --add-port=8080/tcp

# Kubernetes Core Components
sudo firewall-cmd --permanent --add-port=6443/tcp      # API Server
sudo firewall-cmd --permanent --add-port=2379-2380/tcp  # etcd
sudo firewall-cmd --permanent --add-port=10250-10255/tcp  # Kubelet and controllers
sudo firewall-cmd --permanent --add-port=30000-32767/tcp  # NodePorts

# Apply and verify
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
````

---

## 🐳 Docker Installation

[Docker Install Guide for RHEL](https://docs.docker.com/engine/install/rhel/)

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
docker run hello-world
```

---

## ⚙️ Jenkins Installation

[Jenkins RHEL Install Guide](https://www.jenkins.io/doc/book/installing/linux/#red-hat-centos)

```bash
sudo yum install -y fontconfig java-21-openjdk wget
java --version

sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

sudo yum upgrade
sudo yum install -y jenkins

sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

### Jenkins Access

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Visit: `http://<your-server-ip>:8080`

---

## 🧩 Jenkins Plugins to Install

* **JDK**: Eclipse Temurin installer
* **Maven**: Config File Provider, Pipeline Maven Integration
* **Sonar**: SonarQube Scanner
* **Docker**: Docker, Docker Pipeline
* **Kubernetes**: Kubernetes CLI, Client API, Credentials

---

## 🔍 Trivy Installation (Security Scanner)

[Trivy Install Guide](https://trivy.dev/v0.18.3/installation/)

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/trivy.repo
[trivy]
name=Trivy repository
baseurl=https://aquasecurity.github.io/trivy-repo/rpm/releases/\$releasever/\$basearch/
gpgcheck=0
enabled=1
EOF

sudo yum -y update
sudo yum -y install trivy
trivy --version
```

---

## 📦 Nexus (Artifact Repository)

```bash
docker pull sonatype/nexus3
docker run -d --name nexus -p 8081:8081 sonatype/nexus3
```

Access: `http://<your-server-ip>:8081`

Get admin password:

```bash
docker exec -it nexus cat /opt/sonatype/sonatype-work/nexus3/admin.password
```

---

## 📊 SonarQube (Code Quality)

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Access: `http://<your-server-ip>:9000`
Default credentials: `admin / admin`

---

## 🧑‍🔧 Docker Group Permissions

```bash
sudo usermod -aG docker jenkins
sudo usermod -aG docker sonar
sudo usermod -aG docker nexus
sudo usermod -aG docker $USER
newgrp docker  # or re-login
```

> ⚠️ In production, avoid using `chmod 666 /var/run/docker.sock`

---

## 🔁 Systemd Services for Dockerized Nexus & Sonar

### Nexus

```bash
cat <<EOF | sudo tee /etc/systemd/system/docker.nexus.service
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
```

### Sonar

```bash
cat <<EOF | sudo tee /etc/systemd/system/docker.sonar.service
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
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now docker.nexus.service
sudo systemctl enable --now docker.sonar.service
```

## Add maven-release & maven-snapshot in pom.xml file 
distributionManagement>
        <repository>
            <id>maven-releases</id>
            <url>http://192.168.70.135:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>maven-snapshots</id>
            <url>http://192.168.70.135:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>

---

## ☸️ Kubernetes Cluster Setup

### Pre-requisites

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### Container Runtime: containerd

```bash
sudo dnf install -y containerd.io
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

---

## 🔧 Kubernetes Installation

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo yum update -y
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

### Initialize Control Plane

```bash
sudo kubeadm init --ignore-preflight-errors=all
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install Calico CNI

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```


## 📁 GitHub Repository Setup

### Create GitHub Repository
- Repository: `gaurav1517/BoardGame`

### Create GitHub Access Token
- Go to: **Settings > Developer Settings > Personal Access Tokens > Tokens (classic)**
- Click **Generate New Token**  
  - Name: `github-token`  
  - Scope: Select all scopes  
  - Save token securely (you will not see it again after page refresh)

### Push Source Code to GitHub

```bash
git init
git add .
git commit -m "source code"
git config --global user.name "Gaurav Chauhan"
git config --global user.email "gaurav.cloud000@gmail.com"
git branch -M main
git remote add origin https://github.com/gaurav1517/BoardGame.git
git push origin -u main
````

---

## 📄 Jenkinsfile Tools Configuration

```groovy
tools {
  jdk 'jdk-17'
  dockerTool 'docker'
  maven 'maven'
}
```

---

## 🔑 Jenkins Credentials Setup

> Dashboard > Manage Jenkins > Credentials > System > Global credentials (unrestricted)

* **Kind:** Username with password
* **Scope:** Global
* **Username:** Gaurav1517
* **Password:** (your GitHub PAT)
* **ID:** `git-cred`
* **Description:** `git-cred`

---

## 🔗 SonarQube Webhook (For Jenkins Integration)

* Go to **SonarQube > Administration > Webhooks**
* Add New Webhook:

  * **Name:** jenkins
  * **URL:** `http://192.168.70.135:8080/sonar-webhook/`

---

## 🔐 Kubernetes RBAC Configuration for Jenkins
> 🔗 [Configure RBAC ](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin)
### 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webapps
```

```bash
kubectl create -f namespace.yaml
kubectl get ns | grep webapps
```

### 2. Service Account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```

```bash
kubectl create -f serviceAccount.yaml
kubectl get serviceaccounts -n webapps
```

### 3. Role

```yaml
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
```

```bash
kubectl create -f role.yaml
kubectl get role -n webapps
kubectl describe role app-role -n webapps
```

### 4. RoleBinding

```yaml
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
```

```bash
kubectl create -f role-bind.yaml
kubectl get rolebindings -n webapps
```

### 5. Generate Token for Jenkins

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  namespace: webapps 
  annotations:
    kubernetes.io/service-account.name: jenkins
```

```bash
kubectl create -f secret.yaml -n webapps
kubectl get secrets -n webapps
kubectl describe secret mysecretname -n webapps
```

> 🔗 [Service Account Token Reference](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin)

---

## 📦 Kubernetes Deployment and Service

### `deployment-service.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: boardgame-deployment
spec:
  selector:
    matchLabels:
      app: boardgame
  replicas: 2
  template:
    metadata:
      labels:
        app: boardgame
    spec:
      containers:
        - name: boardgame
          image: gchauhan1517/boardgame:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: boardgame-ssvc
spec:
  selector:
    app: boardgame
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: LoadBalancer
```

---

## 📧 Jenkins Email Notification Setup

### Create App Password on Gmail

Use App Password (not actual Gmail password) for authentication.

### Configure in Jenkins:

> Manage Jenkins > System

#### Extended E-mail Notification

* **SMTP Server:** `smtp.gmail.com`
* **SMTP Port:** `465`
* **Use SSL:** ✅
* **Credentials:** `gaurav.mau854@gmail.com` / `app-password` (ID: `mail-cred`)

#### E-mail Notification

* **SMTP Server:** `smtp.gmail.com`
* **SMTP Port:** `465`
* **Use SMTP Authentication:** ✅
* **User Name:** `gaurav.mau854@gmail.com`
* **Password:** `app-password`
* **Use SSL:** ✅

#### Test Email

* Recipient: `gaurav.mau854@gmail.com`
* Status: ✅ Successfully sent

### Allow Port 465 on Jenkins Server

```bash
sudo firewall-cmd --add-port=465/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

