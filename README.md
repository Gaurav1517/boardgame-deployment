# 🎲 Boardgame App Infrastructure Setup

This document outlines the complete infrastructure setup for the **Boardgame App**, including CI/CD pipelines, security scanning, artifact management, monitoring, and Kubernetes deployment.

---

## 📦 Components Used

- Jenkins (CI/CD)
- SonarQube (Code Quality)
- Nexus (Artifact Repository)
- Docker (Containerization)
- Kubernetes (Orchestration)
- Prometheus + Grafana (Monitoring)
- Trivy (Security Scanning)
- Blackbox & Node Exporter (System Metrics)

---


## 🔥 Firewall Configuration

Ensure these ports are open on respective servers:

| Service       | Port   |
|---------------|--------|
| Jenkins       | 8080   |
| Nexus         | 8081   |
| SonarQube     | 9000   |
| Prometheus    | 9090   |
| Grafana       | 3000   |
| Node Exporter | 9100   |
| Blackbox      | 9115   |
| K8s API       | 6443   |
| NodePort      | 30000-32767 |


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

> 🔗 [RBACK Configuration](https://github.com/jaiswaladi246/EKS-Complete/blob/main/Steps-eks.md)

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

# Install kubectl on jenkins server
#!/bin/bash

set -e  # Exit immediately if a command exits with a non-zero status

echo "📥 Downloading the latest kubectl binary..."
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

echo "🔐 Making kubectl executable..."
chmod +x kubectl

echo "🚚 Moving kubectl to /usr/local/bin (requires sudo)..."
sudo mv kubectl /usr/local/bin/

echo "✅ Verifying kubectl installation..."
kubectl version --client

echo "🎉 kubectl installation completed successfully!"


# RUn jenkins pipeline 
http://<worker-node-IP:service-port>



# Prometheus v3.4.1 Installation and Setup on Linux
REF: https://prometheus.io/download/
---

## 1. Download and Extract Prometheus

```bash
wget https://github.com/prometheus/prometheus/releases/download/v3.4.1/prometheus-3.4.1.linux-amd64.tar.gz
mkdir -p /opt/prometheus
tar -xvzf prometheus-3.4.1.linux-amd64.tar.gz -C /opt/prometheus --strip-components=1
rm -f prometheus-3.4.1.linux-amd64.tar.gz
```

---

## 2. Create Prometheus User and Directories

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown -R prometheus:prometheus /opt/prometheus /etc/prometheus /var/lib/prometheus
```

---

## 3. Copy Configuration Files

```bash
sudo cp /opt/prometheus/prometheus.yml /etc/prometheus/
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

> **Note:** Prometheus v3.x **does not include** `consoles` and `console_libraries` folders anymore, so **skip copying those**.

---

## 4. Create systemd Service File

Create `/etc/systemd/system/prometheus.service` with:

```bash
sudo tee /etc/systemd/system/prometheus.service > /dev/null <<EOF
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/opt/prometheus/prometheus \\
  --config.file=/etc/prometheus/prometheus.yml \\
  --storage.tsdb.path=/var/lib/prometheus/

Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

---

## 5. Reload systemd and Start Prometheus

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

---

## 6. Verify Prometheus Service Status

```bash
sudo systemctl status prometheus
```

## 7. Access Prometheus UI

Open your browser to:

```
http://<your-server-ip>:9090
```

---

# Add port in Firewall Adjustment 

```bash
sudo firewall-cmd --add-port=9090/tcp --permanent
sudo firewall-cmd --reload
```


## ✅ Step-by-Step: Install & Start Grafana on RHEL/CentOS
REF: https://grafana.com/grafana/download

### 🔹 1. Download and Install Grafana Enterprise

```bash
sudo yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-12.0.1-1.x86_64.rpm
```

> 💡 You can use `dnf` instead of `yum` if you're on a newer RHEL version:
>
> ```bash
> sudo dnf install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-12.0.1-1.x86_64.rpm
> ```

---

### 🔹 2. Enable and Start Grafana Server

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

---

### 🔹 3. Check Grafana Status

```bash
sudo systemctl status grafana-server
```

You should see:
`Active: active (running)`

---

### 🔹 4. Open Grafana in Browser

Visit:

```
http://<your-server-ip>:3000
```

### 🔐 Default login:

* **Username:** `admin`
* **Password:** `admin` (you will be prompted to change it)

---

### 🔹 5. Add Prometheus as a Data Source in Grafana

Once logged into Grafana:

1. Go to **"Gear" → "Data Sources"**.
2. Click **"Add data source"**.
3. Choose **Prometheus**.
4. In the URL field, enter:

   ```
   http://localhost:9090
   ```

   (or the IP where Prometheus is running)
5. Click **"Save & Test"**.

---

### 🔹 6. (Optional) Import a Dashboard

* Go to **"Dashboards" → "Import"**
* Use an existing dashboard ID from [Grafana Dashboards](https://grafana.com/grafana/dashboards/)

  * For example: **1860** (Prometheus Stats)

---


# Download blackbox exporter 
NOTE": Before insalling blackbox exporter change is prometheus configureation file 
REF: https://github.com/prometheus/blackbox_exporter
vim /etc/prometheus/prometheus.yml
# my global config
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: []

rule_files: []

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - http://prometheus.io    # Target to probe with http.
        - http://<192.168.70.130:30248> # k8s master node ip with deploymnet service port.
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: <192.168.70.135:9115>  # The blackbox exporter's real hostname:port.

###  Restart Prometheus to Apply Config
sudo systemctl restart prometheus
sudo systemctl status prometheus


### 🔹 1. Download & Install Blackbox Exporter

```bash
# Download
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.26.0/blackbox_exporter-0.26.0.linux-amd64.tar.gz

# Extract
mkdir -p /opt/blackbox_exporter
tar -xvzf blackbox_exporter-0.26.0.linux-amd64.tar.gz -C /opt/blackbox_exporter --strip-components=1
rm -f blackbox_exporter-0.26.0.linux-amd64.tar.gz

# Create system user
sudo useradd --no-create-home --shell /bin/false blackbox_exporter

# Set permissions
sudo chown -R blackbox_exporter:blackbox_exporter /opt/blackbox_exporter
```

---

### 🔹 2. Create systemd Service File

```bash
sudo tee /etc/systemd/system/blackbox_exporter.service > /dev/null <<EOF
[Unit]
Description=Prometheus Blackbox Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=blackbox_exporter
Group=blackbox_exporter
Type=simple
ExecStart=/opt/blackbox_exporter/blackbox_exporter --config.file=/opt/blackbox_exporter/blackbox.yml

Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

---

### 🔹 3. Start the Blackbox Exporter

```bash
sudo systemctl daemon-reload
sudo systemctl enable blackbox_exporter
sudo systemctl start blackbox_exporter
sudo systemctl status blackbox_exporter
```


### 🔹 6. Test in Browser or CLI

**Test Blackbox Exporter is listening:**

```bash
curl "http://localhost:9115/probe?target=https://www.google.com&module=http_2xx"
```

---

### 🔹 7. (Optional) Open Firewall Port

If you want to access Blackbox Exporter remotely:

```bash
sudo firewall-cmd --add-port=9115/tcp --permanent
sudo firewall-cmd --reload
```

---

Install plugins 
Prometheus metricsVersion

## ✅ Step-by-Step: Install Node Exporter (v1.9.1)

### 🔹 1. Download & Extract

```bash
cd ~
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz

# Extract to /opt
mkdir -p /opt/node_exporter
tar -xvzf node_exporter-1.9.1.linux-amd64.tar.gz -C /opt/node_exporter --strip-components=1

# Remove archive
rm -f node_exporter-1.9.1.linux-amd64.tar.gz
```

---

### 🔹 2. Create a User

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
sudo chown -R node_exporter:node_exporter /opt/node_exporter
```

---

### 🔹 3. Create systemd Service

```bash
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/opt/node_exporter/node_exporter

Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

---

### 🔹 4. Start & Enable Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

You should see `Active: active (running)` ✅

---

### 🔹 5. (Optional) Open Firewall Port

Node Exporter listens on port **9100**:

```bash
sudo firewall-cmd --add-port=9100/tcp --permanent
sudo firewall-cmd --reload
```

---

### 🔹 6. Add to Prometheus Scrape Config (Optional)

Edit `/etc/prometheus/prometheus.yml` and add:

```yaml
  - job_name: "node_exporter_9100"
  static_configs:
    - targets:
        - "192.168.70.135:9100"

- job_name: "custom_metrics_8080"
  metrics_path: '/prometheus'
  static_configs:
    - targets:
        - "192.168.70.135:8080"

```

Then restart Prometheus:

```bash
sudo systemctl restart prometheus
```

---

## ✅ Final Verification

* **Browser**: Visit `http://<your-server-ip>:9100/metrics`
* **Prometheus UI**: Check target status under **Status > Targets**
* **Metrics**: Search for `node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`, etc.

---

REF: 
Docker Installation: https://docs.docker.com/engine/install/rhel/
Docker Hub: https://hub.docker.com/
Git Installation: https://git-scm.com/downloads/linux
Nexus docker hub image: https://hub.docker.com/r/sonatype/nexus3
SonarQube docker hub image: https://hub.docker.com/layers/library/sonarqube/lts-community/images/sha256-d3d04c0fec696dcf92657ae25ee5662aba32b1a44f61571ea7b1adca001a647a
Jenkins Installation: https://www.jenkins.io/doc/book/installing/linux/#red-hat-centos
Trivy installation: https://trivy.dev/v0.18.3/installation/
Kubernetes cluster setup: https://v1-32.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
K8s RBAC: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
K8s SeviceAccount: https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/
Kubectl download: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
Prometheous download: https://prometheus.io/download/
Grafana download: https://grafana.com/grafana/download
Node exporter dashboard : https://grafana.com/grafana/dashboards/1860-node-exporter-full/
Prometheus Blackbox exporter: https://grafana.com/grafana/dashboards/7587-prometheus-blackbox-exporter/
Prometheus configuration: https://github.com/prometheus/blackbox_exporter



