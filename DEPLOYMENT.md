# Deployment Steps

These are the steps to set up the Netflix clone DevSecOps pipeline on AWS from scratch. Replace the placeholders in `< >` with your own values.

## 1. Launch the EC2 instance

In the AWS Console, go to **EC2** → **Instances** → **Launch instances**:

- Name: `netflix-jenkins`
- AMI: **Ubuntu Server 24.04 LTS**
- Instance type: `m7i-flex.large`
- Key pair: `<your-key>`
- Network settings: **Create security group**, allow SSH (22)
- Storage: **25 GiB** gp3

Click **Launch instance**.

## 2. Allocate and associate an Elastic IP

Go to **EC2** → **Elastic IPs** → **Allocate Elastic IP address**, name it `netflix-eip`, click **Allocate**.

Then **Actions** → **Associate Elastic IP address** → select `netflix-jenkins` → **Associate**.

Connect to the instance:

```bash
ssh -i <your-key>.pem ubuntu@<netflix-eip>
```

## 3. Install Docker

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

## 4. Give the ubuntu user Docker access

This lets the ubuntu user run Docker commands without `sudo`.

```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

Verify:

```bash
docker ps
```

## 5. Create the repo and import the app

On GitHub, click **+** → **Import repository**:

- Source: `https://github.com/N4si/DevSecOps-Project.git`
- Repository name: `devsecops-netflix-k8s`
- Visibility: **Public**

Click **Begin import**.

## 6. Restructure the repo

Clone the repo on your PC and open it in VS Code:

```bash
git clone https://github.com/<your-github-username>/devsecops-netflix-k8s.git
cd devsecops-netflix-k8s
```

Move all app files (everything except `Kubernetes/`, `README.md` and `.git`) into a new `app/` folder and delete `README.md`.

Rename `Kubernetes` to `kubernetes` in two steps (Windows ignores renames that only change upper/lower case):

```bash
git mv Kubernetes kubernetes-temp
git mv kubernetes-temp kubernetes
```

Add `pipeline.txt` at the root with the Jenkins pipeline as reference.

`yarn.lock` is listed in `.gitignore`, so it has to be added with `-f`:

```bash
git add -f app/yarn.lock
git add -A
git commit -m "Restructure repo"
git push
```

## 7. Clone the repo on the EC2

```bash
git clone https://github.com/<your-github-username>/devsecops-netflix-k8s.git
cd devsecops-netflix-k8s/app
```

## 8. Open the ports

Go to **EC2** → **Instances** → `netflix-jenkins` → **Security** tab → click the security group → **Edit inbound rules** → **Add rule** for each port (Type **Custom TCP**, Source **Anywhere-IPv4**):

- `8080` (Jenkins)
- `8081` (Netflix app)
- `9000` (SonarQube)

Click **Save rules**.

## 9. Get a TMDB API key

1. Create an account on [themoviedb.org](https://www.themoviedb.org/) and log in.
2. Go to your profile → **Settings** → **API**.
3. Click **Create**, accept the terms and fill in the details.
4. Copy the API key.

## 10. Build and run the app

Start the `docker build` line with a space so the API key isn't saved in the bash history:

```bash
 docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest
```

Verify: open `http://<netflix-eip>:8081`. The movies should load.

## 11. Run SonarQube

```bash
docker run -d --name sonar --restart unless-stopped -p 9000:9000 sonarqube:community
```

Open `http://<netflix-eip>:9000`, log in with `admin` / `admin` and set a new password.

## 12. Install Trivy and scan

```bash
sudo apt-get install wget gnupg -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

Scan the code and the image:

```bash
trivy fs .
trivy image netflix:latest
```

## 13. Install Java and Jenkins

```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre -y
java -version

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
sudo systemctl enable --now jenkins
```

Get the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open `http://<netflix-eip>:8080`, paste the password, click **Install suggested plugins** and create your admin user with a strong password.

Remove the initial password file if it still exists:

```bash
sudo rm -f /var/lib/jenkins/secrets/initialAdminPassword
```

## 14. Install Jenkins plugins

Go to **Manage Jenkins** → **Plugins** → **Available plugins**, search and tick:

- Eclipse Temurin installer
- SonarQube Scanner
- NodeJS

Click **Install**.

## 15. Configure the JDK and Node.js tools

Go to **Manage Jenkins** → **Tools**:

- **JDK installations** → **Add JDK**: Name `jdk17`, tick **Install automatically**, **Add Installer** → **Install from adoptium.net**, pick the newest `jdk-17` version.
- **NodeJS installations** → **Add NodeJS**: Name `node16`, version **NodeJS 16.2.0**.

Click **Save**.

## 16. Connect SonarQube to Jenkins

Create a token in SonarQube: click your **avatar** (top right) → **My Account** → **Security** → **Generate Tokens**:

- Name: `jenkins`
- Type: **User Token**

Click **Generate** and copy the token.

Add it to Jenkins: **Manage Jenkins** → **Credentials** → **System** → **Global credentials (unrestricted)** → **Add Credentials**:

- Kind: **Secret text**
- Secret: `<sonar-token>`
- ID: `Sonar-token`

Add the server: **Manage Jenkins** → **System** → **SonarQube servers** → **Add SonarQube**:

- Name: `sonar-server`
- Server URL: `http://<netflix-eip>:9000`
- Server authentication token: `Sonar-token`

Click **Save**.

Add the scanner: **Manage Jenkins** → **Tools** → **SonarQube Scanner installations** → **Add SonarQube Scanner**: Name `sonar-scanner`, tick **Install automatically**. Click **Save**.

Create the webhook in SonarQube: **Administration** → **Configuration** → **Webhooks** → **Create**:

- Name: `jenkins`
- URL: `http://<netflix-eip>:8080/sonarqube-webhook/`

Click **Create**.

## 17. Create the pipeline job

In Jenkins, click **New Item**, name it `netflix`, select **Pipeline**, click **OK**. Under **Pipeline**, choose **Pipeline script**, paste this and click **Save**:

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/<your-github-username>/devsecops-netflix-k8s.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                dir('app') {
                    withSonarQubeEnv('sonar-server') {
                        sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix'''
                    }
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                dir('app') {
                    sh "npm install"
                }
            }
        }
    }
}
```

Click **Build Now**.

Verify: the build is green and the project **Netflix** shows **Passed** in SonarQube.

## 18. Install the Dependency-Check and Docker plugins

Go to **Manage Jenkins** → **Plugins** → **Available plugins**, search and tick:

- OWASP Dependency-Check
- Docker
- Docker Commons
- Docker Pipeline
- Docker API
- docker-build-step

Click **Install**.

## 19. Configure the Dependency-Check and Docker tools

Go to **Manage Jenkins** → **Tools**:

- **Dependency-Check installations** → **Add Dependency-Check**: Name `DP-Check`, tick **Install automatically**, **Add Installer** → **Install from github.com**, pick the newest version.
- **Docker installations** → **Add Docker**: Name `docker`, untick **Install automatically**, Installation root `/usr` (uses the Docker already installed on the EC2).

Click **Save**.

## 20. Get an NVD API key

Dependency-Check downloads the vulnerability database from the NVD and needs an API key for it.

1. Go to [nvd.nist.gov/developers/request-an-api-key](https://nvd.nist.gov/developers/request-an-api-key).
2. Fill in your organization type and email, accept the terms and submit.
3. Open the link in the email to activate the key and copy it.

## 21. Add the Docker Hub credentials

Create a token in Docker Hub: click your **avatar** → **Account settings** → **Personal access tokens** → **Generate new token**:

- Description: `jenkins`
- Access permissions: **Read & Write** (only what the pipeline needs)

Click **Generate** and copy the token.

Add it to Jenkins: **Manage Jenkins** → **Credentials** → **System** → **Global credentials (unrestricted)** → **Add Credentials**:

- Kind: **Username with password**
- Username: `<your-dockerhub-username>`
- Password: `<docker-hub-token>`
- ID: `docker`

## 22. Give Jenkins Docker access

Jenkins runs as its own `jenkins` user. It needs to be in the docker group to build, push and run containers.

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

## 23. Remove the old app container

The pipeline starts a new container called `netflix` on port 8081, so the one from step 10 has to go first.

```bash
docker stop netflix
docker rm netflix
```

## 24. Update the pipeline job with the full pipeline

Open the `netflix` job → **Configure**, replace the script with this and click **Save**:

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/<your-github-username>/devsecops-netflix-k8s.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                dir('app') {
                    withSonarQubeEnv('sonar-server') {
                        sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix'''
                    }
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                dir('app') {
                    sh "npm install"
                }
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dir('app') {
                    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey <nvd-api-key>', odcInstallation: 'DP-Check'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                }
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                dir('app') {
                    sh "trivy fs . > trivyfs.txt"
                }
            }
        }
        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        dir('app') {
                            sh "git checkout -- yarn.lock"
                            sh "docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix ."
                        }
                        sh "docker tag netflix <your-dockerhub-username>/netflix:latest"
                        sh "docker push <your-dockerhub-username>/netflix:latest"
                    }
                }
            }
        }
        stage("TRIVY") {
            steps {
                sh "trivy image <your-dockerhub-username>/netflix:latest > trivyimage.txt"
            }
        }
        stage('Deploy to container') {
            steps {
                sh 'docker run -d --name netflix -p 8081:80 <your-dockerhub-username>/netflix:latest'
            }
        }
    }
}
```

Click **Build Now**. The first OWASP scan downloads the whole NVD database and takes 10–20+ minutes.

Verify: the build is green, the image is on Docker Hub, and the app loads at `http://<netflix-eip>:8081`.

## 25. Launch the monitoring EC2 instance

Go to **EC2** → **Instances** → **Launch instances**:

- Name: `netflix-monitoring`
- AMI: **Ubuntu Server 24.04 LTS**
- Instance type: `c7i-flex.large`
- Key pair: `<your-key>`
- Network settings: **Create security group**, allow SSH (22) and HTTPS (443), then add **Custom TCP** rules (Source **Anywhere-IPv4**) for:
  - `9090` (Prometheus)
  - `3000` (Grafana)
  - `9100` (Node Exporter)
- Storage: **20 GiB** gp3

Click **Launch instance**.

## 26. Allocate and associate an Elastic IP for monitoring

Go to **EC2** → **Elastic IPs** → **Allocate Elastic IP address**, name it `monitoring-eip`, click **Allocate**.

Then **Actions** → **Associate Elastic IP address** → select `netflix-monitoring` → **Associate**.

Connect and update the packages:

```bash
ssh -i <your-key>.pem ubuntu@<monitoring-eip>
sudo apt update -y
```

## 27. Install Prometheus

```bash
sudo useradd --system --no-create-home --shell /bin/false prometheus
cd ~
wget https://github.com/prometheus/prometheus/releases/download/v3.16.0/prometheus-3.16.0.linux-amd64.tar.gz
tar -xvf prometheus-3.16.0.linux-amd64.tar.gz
cd prometheus-3.16.0.linux-amd64/
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
cd ~
rm -rf prometheus-3.16.0.linux-amd64*
```

Create the service file:

```bash
sudo nano /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

Start Prometheus:

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

Verify: open `http://<monitoring-eip>:9090`.

## 28. Install Node Exporter

```bash
sudo useradd --system --no-create-home --shell /bin/false node_exporter
cd ~
wget https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.12.1.linux-amd64.tar.gz
sudo mv node_exporter-1.12.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
```

Create the service file:

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

Start Node Exporter:

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

Verify: open `http://<monitoring-eip>:9100/metrics`.

## 29. Install the Prometheus plugin in Jenkins

Go to **Manage Jenkins** → **Plugins** → **Available plugins**, search `Prometheus metrics`, tick it and click **Install**.

Restart Jenkins on the Jenkins EC2 so it loads the plugin:

```bash
sudo systemctl restart jenkins
```

Verify: open `http://<netflix-eip>:8080/prometheus/`. You should see the Jenkins metrics as text.

## 30. Configure Prometheus to scrape Node Exporter and Jenkins

On the monitoring EC2, open the Prometheus config:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Add these jobs at the end of the file, in line with the existing `- job_name: "prometheus"`:

```yaml
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<netflix-eip>:8080']
```

Check the config and reload Prometheus:

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

Verify: open `http://<monitoring-eip>:9090/targets`. All three jobs (prometheus, node_exporter, jenkins) should show **UP**.

## 31. Install Grafana

On the monitoring EC2:

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get -y install grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Open `http://<monitoring-eip>:3000`, log in with `admin` / `admin` and set a new password.

Add Prometheus as a data source: **Connections** → **Data sources** → **Add data source** → **Prometheus**, set **Prometheus server URL** to `http://localhost:9090`, click **Save & test**.

## 32. Import the Grafana dashboards

Go to **Dashboards** → **New** → **Import**, enter the ID, click **Load**, then **Import**:

- `1860` – Node Exporter Full (CPU, RAM, disk and network of the monitoring server)
- `9964` – Jenkins: Performance and Health Overview (jobs, builds, executors)

On each dashboard, check that **Datasource** is set to `prometheus`.

Verify: both dashboards show graphs. The Jenkins build panels fill in after the next pipeline run.

## 33. Create a Google App Password

Jenkins sends the emails through Gmail and needs an App Password for it. 2-Step Verification must be on in your Google account.

1. Open [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords).
2. Enter the app name `Netflix` and click **Create**.
3. Copy the 16-character password.

## 34. Configure email in Jenkins

Go to **Manage Jenkins** → **System**.

**E-mail Notification** (bottom of the page):

- SMTP server: `smtp.gmail.com`
- Default user e-mail suffix: `@gmail.com`
- **Advanced**: tick **Use SMTP Authentication**, User Name `<your-gmail-address>` (the full address), Password `<app-password>` (without spaces), tick **Use SSL**, SMTP Port `465`

Verify: tick **Test configuration by sending test e-mail**, enter your email address and click **Test configuration**. It should say **Email was successfully sent**.

**Extended E-mail Notification**:

- SMTP server: `smtp.gmail.com`
- SMTP Port: `465`
- **Advanced** → **Credentials** → **+ Add** → **Jenkins**, create a separate credential for Gmail:
  - Kind: **Username with password**
  - Username: `<your-gmail-address>` (the full address)
  - Password: `<app-password>` (without spaces)
  - ID: `mail`
  - Description: `Gmail App Password`
- Select **Gmail App Password** in the Credentials dropdown (not the Docker Hub credential)
- Tick **Use SSL**
- Default Content Type: **HTML (text/html)**
- **Default Triggers**: tick **Always** and **Failure - Any**

Click **Save**.

## 35. Add the email notification to the pipeline

Open the `netflix` job → **Configure** and add this `post` block at the end of the pipeline, after the `}` that closes `stages`:

```groovy
post {
    always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: '<your-email>',
            attachmentsPattern: 'app/trivyfs.txt,trivyimage.txt'
    }
}
```

Click **Save**. The app container from the last build is still running, so remove it first on the Jenkins EC2:

```bash
docker stop netflix
docker rm netflix
```

Click **Build Now**.

Verify: the build is green and you get an email with the build log and both Trivy reports attached.

## 36. Launch the Kubernetes EC2 instance

Go to **EC2** → **Instances** → **Launch instances**:

- Name: `netflix-k8s`
- AMI: **Ubuntu Server 24.04 LTS**
- Instance type: `c7i-flex.large`
- Key pair: `<your-key>`
- Network settings: **Create security group**, allow SSH (22), HTTP (80) and HTTPS (443), then add **Custom TCP** rules (Source **Anywhere-IPv4**) for:
  - `30007` (Netflix app)
  - `9100` (Node Exporter)
- Storage: **20 GiB** gp3

Click **Launch instance**.

## 37. Allocate and associate an Elastic IP for Kubernetes

Go to **EC2** → **Elastic IPs** → **Allocate Elastic IP address**, name it `k8s-eip`, click **Allocate**.

Then **Actions** → **Associate Elastic IP address** → select `netflix-k8s` → **Associate**.

Connect and update the packages:

```bash
ssh -i <your-key>.pem ubuntu@<k8s-eip>
sudo apt update -y
```

## 38. Install k3s

Set the hostname (becomes the node name):

```bash
sudo hostnamectl set-hostname netflix-k8s
echo "127.0.1.1 netflix-k8s" | sudo tee -a /etc/hosts
```

Reconnect via SSH so the new hostname is active. Install k3s without Traefik, so ports 80 and 443 stay free for the Argo CD load balancer:

```bash
curl -sfL https://get.k3s.io | sh -s - --disable traefik
```

Let the ubuntu user use kubectl without `sudo`:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown ubuntu:ubuntu ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

Verify: node **netflix-k8s** is **Ready**.

```bash
kubectl get nodes
kubectl get pods -A
```

A **Completed** `helm-install-gateway-api-crd` job is normal.

## 39. Install Helm and Node Exporter

```bash
sudo snap install helm --classic
helm version
```

Install Node Exporter with Helm:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace prometheus-node-exporter
helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter
```

Verify:

```bash
kubectl get pods -n prometheus-node-exporter
```

The pod shows **Running**, and `http://<k8s-eip>:9100/metrics` shows the metrics.

## 40. Add the K8s Node to Prometheus

On **netflix-monitoring**:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Add at the end of `scrape_configs`:

```yaml
  - job_name: 'Netflix'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['<k8s-eip>:9100']
```

Check and reload:

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

Verify: **Status → Target health** in Prometheus shows the **Netflix** target as **UP**.

## 41. Argo CD

On **netflix-k8s**:

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w
```

Wait until all pods are **Running**, then **Ctrl+C**.

Expose the UI:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd
```

Verify: **TYPE** is **LoadBalancer** with ports 80 and 443.

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Open `https://<k8s-eip>`, accept the certificate warning, and log in as **admin**.

Verify: the Argo CD **Applications** page opens.

Change the admin password: **User Info → UPDATE PASSWORD** → **SAVE NEW PASSWORD**.

Delete the initial password secret:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

Verify: `kubectl -n argocd get secret argocd-initial-admin-secret` returns **NotFound**.

## 42. Set the Image in deployment.yml

In `kubernetes/deployment.yml`, set the image:

```yaml
image: emlykf/netflix:latest
```

Push the change to GitHub.

Verify: the Docker Hub repo **emlykf/netflix** is **Public**.

## 43. Create the Argo CD Application

In the Argo CD UI, click **+ NEW APP**:

- **Application Name:** `netflix`
- **Project Name:** `default`
- **Sync Policy:** **Automatic**, **Prune Resources**, **Self Heal**
- **Repository URL:** `https://github.com/emlykf/devsecops-netflix-k8s.git`
- **Revision:** `HEAD`
- **Path:** `kubernetes`
- **Cluster URL:** `https://kubernetes.default.svc`
- **Namespace:** `default`

Click **CREATE**.

Verify: the app shows **Healthy** and **Synced**.

## 44. Access the App

```bash
kubectl get pods,svc -n default
```

Open `http://<k8s-eip>:30007`.

Verify: the Netflix app loads with movies.

## 45. Restrict security groups

By default, all ports were open to the internet (`0.0.0.0/0`). Each port is now only open to the IP that needs it.
`My IP` = your current public IP (choose **My IP** in the Source dropdown). `/32` = exactly one IP.

EC2 → **Instances** → select the instance → **Security** tab → click the security group → **Inbound rules** → **Edit inbound rules** → change the rules as below → **Save rules**.

**netflix-jenkins** (delete 80 and 443)

| Type | Port | Source | Description |
|---|---|---|---|
| SSH | 22 | My IP | SSH |
| Custom TCP | 8080 | My IP | jenkins |
| Custom TCP | 8080 | `<monitoring-eip>/32` | prometheus |
| Custom TCP | 8080 | `<netflix-eip>/32` | sonarqube webhook |
| Custom TCP | 9000 | My IP | sonarqube |
| Custom TCP | 9000 | `<netflix-eip>/32` | jenkins to sonarqube |

**netflix-monitoring** (delete 443 and 9100; Prometheus scrapes its own Node Exporter via `localhost:9100`)

| Type | Port | Source | Description |
|---|---|---|---|
| SSH | 22 | My IP | SSH |
| Custom TCP | 3000 | My IP | grafana |
| Custom TCP | 9090 | My IP | prometheus |

**netflix-k8s**

| Type | Port | Source | Description |
|---|---|---|---|
| SSH | 22 | My IP | SSH |
| HTTP | 80 | My IP | Argo CD UI |
| HTTPS | 443 | My IP | Argo CD UI |
| Custom TCP | 30007 | My IP | Netflix app |
| Custom TCP | 9100 | `<monitoring-eip>/32` | node exporter |

Check:
- `http://<monitoring-eip>:9090/targets` → all 4 targets UP
- `http://<k8s-eip>:9100/metrics` from your PC → does not load
- Jenkins **Build Now** → SonarQube Analysis and quality gate stages pass
- Argo CD (`https://<k8s-eip>`) and the app (`http://<k8s-eip>:30007`) load

> If your public IP changes (other network, router restart), set the **My IP** rules again.

## 46. Move API keys into Jenkins credentials

The NVD and TMDB keys were written in plain text in the pipeline. They are now stored as Jenkins credentials and hidden in the build logs.

Jenkins → **Manage Jenkins** → **Credentials** → **System** → **Global credentials (unrestricted)** → **Add Credentials**:

| Kind | Secret | ID | Description |
|---|---|---|---|
| Secret text | your TMDB API key | `tmdb-api-key` | TMDB API key |
| Secret text | your NVD API key | `nvd-api-key` | NVD API key for Dependency-Check |

Update the pipeline (job → **Configure** → **Pipeline**):

OWASP FS SCAN, replace the `dependencyCheck` line:
```groovy
dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', nvdCredentialsId: 'nvd-api-key', odcInstallation: 'DP-Check'
```

Docker Build & Push, replace the `docker build` line:
```groovy
withCredentials([string(credentialsId: 'tmdb-api-key', variable: 'TMDB_KEY')]) {
    sh 'docker build --build-arg TMDB_V3_API_KEY=$TMDB_KEY -t netflix .'
}
```
> Use single quotes so the shell fills in `$TMDB_KEY`, not Jenkins. This keeps the key out of the log.

Remove the old app container on netflix-jenkins, then run the pipeline:
```bash
docker rm -f netflix
```

Check:
- All stages green
- Console Output does not contain either key
- `http://<netflix-eip>:8081` loads movies

## 47. Use only yarn

The app had two lock files: `package-lock.json` (npm) and `yarn.lock` (yarn). The scans checked npm's list, but the Docker image is built with yarn. Now only yarn is used, so the scans check exactly what goes into the image.

In `app/Dockerfile`, replace `RUN yarn install` with:
```dockerfile
RUN yarn install --frozen-lockfile
```
> `--frozen-lockfile` installs exactly what's in `yarn.lock` and stops if something doesn't match.

Update the pipeline (job → **Configure** → **Pipeline**):

Install Dependencies, replace `npm install` with:
```groovy
sh "npm install -g yarn"
sh "yarn install --frozen-lockfile"
```

OWASP FS SCAN, remove `--disableYarnAudit` so Dependency-Check checks `yarn.lock`:
```groovy
dependencyCheck additionalArguments: '--scan ./ --disableNodeAudit', nvdCredentialsId: 'nvd-api-key', odcInstallation: 'DP-Check'
```

Docker Build & Push, remove this line (it only undid npm's changes to `yarn.lock`):
```groovy
sh "git checkout -- yarn.lock"
```

Deploy to container, add this line before `docker run` (removes the old container first):
```groovy
sh 'docker rm -f netflix || true'
```

Check:
- All stages green
- Trivy fs report only shows `yarn.lock` (no `package-lock.json`)
- Dependency-Check still shows findings

## 48. Update Node to 26 LTS

Node 16 stopped getting security fixes in 2023. Node 26 is the current LTS version.

In `app/Dockerfile`, replace the first line and add the yarn install. The top should look like this:
```dockerfile
FROM node:26-alpine AS builder
WORKDIR /app
COPY ./package.json .
COPY ./yarn.lock .
RUN npm install -g yarn
RUN yarn install --frozen-lockfile
```
> The `node:26-alpine` image does not include yarn anymore, so it is installed first.

On netflix-jenkins, install libatomic (needed by Node 26 on Linux):
```bash
sudo apt-get update && sudo apt-get install -y libatomic1
```

Jenkins → **Manage Jenkins** → **Tools** → **NodeJS installations** → **Add NodeJS**:
- **Name:** `node26`
- **Install automatically** → **Version:** newest NodeJS 26.x

→ **Save**

In the pipeline, change the Node tool:
```groovy
nodejs 'node26'
```

Check:
- All stages green
- Console Output shows `FROM node:26-alpine`
- `http://<netflix-eip>:8081` loads movies

## 49. Fix vulnerable packages

Only the vulnerable packages are updated. A full `yarn upgrade` breaks the build because other libraries change their code.

Requires Node and yarn on your PC (`winget install OpenJS.NodeJS.LTS`, then `npm install -g yarn`).

In the `app` folder:

Update React Router to the newest 6.x (also fixes `react-router` and `@remix-run/router`):
```bash
yarn upgrade react-router-dom@^6.30.4
```

Force fixed versions for packages that other libraries use:
```bash
npm pkg set "resolutions.@babel/runtime=^7.26.10"
npm pkg set "resolutions.@xmldom/xmldom=^0.8.15"
npm pkg set "resolutions.brace-expansion=^1.1.21"
npm pkg set "resolutions.json5=^2.2.2"
npm pkg set "resolutions.min-document=^2.19.1"
npm pkg set "resolutions.minimatch=^3.1.4"
npm pkg set "resolutions.yaml=^1.10.3"
```

Apply and test the build locally:
```bash
yarn install
yarn build
```

Push `app/package.json` and `app/yarn.lock`, then run the pipeline.

Check:
- All stages green
- Trivy fs: 2 MEDIUM left (react-router, fixed only in React Router 7)
- App loads and movie pages open

## 50. Make the scans stop the build

Before, every scan only wrote a report and the pipeline continued. Now the build fails on HIGH or CRITICAL findings in what ships, and the image is scanned before it is pushed.

Update the pipeline (job → **Configure** → **Pipeline**):

quality gate, stop the pipeline if SonarQube's gate fails:
```groovy
waitForQualityGate abortPipeline: true, credentialsId: 'Sonar-token'
```

OWASP FS SCAN, skip build tools (dev dependencies) and fail on HIGH/CRITICAL:
```groovy
dependencyCheck additionalArguments: '--scan ./ --disableNodeAudit --nodeAuditSkipDevDependencies --nodePackageSkipDevDependencies', nvdCredentialsId: 'nvd-api-key', odcInstallation: 'DP-Check'
dependencyCheckPublisher pattern: '**/dependency-check-report.xml', failedTotalCritical: 1, failedTotalHigh: 1, stopBuild: true
```
> Build tools (vite, rollup, babel, ...) only run in the first Docker stage and are not in the final image.

TRIVY FS SCAN, add a gate after the report:
```groovy
sh "trivy fs . > trivyfs.txt"
sh "trivy fs --skip-db-update --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ."
```

Split "Docker Build & Push" into **Docker Build → TRIVY → Docker Push**, so the image is scanned before it reaches Docker Hub:
```groovy
stage("Docker Build") {
    steps {
        dir('app') {
            withCredentials([string(credentialsId: 'tmdb-api-key', variable: 'TMDB_KEY')]) {
                sh 'docker build --build-arg TMDB_V3_API_KEY=$TMDB_KEY -t netflix .'
            }
        }
        sh "docker tag netflix emlykf/netflix:latest"
    }
}
stage("TRIVY") {
    steps {
        sh "trivy image emlykf/netflix:latest > trivyimage.txt"
        sh "trivy image --skip-db-update --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 emlykf/netflix:latest"
    }
}
stage("Docker Push") {
    steps {
        script {
            withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                sh "docker push emlykf/netflix:latest"
            }
        }
    }
}
```
> Each Trivy stage runs twice: first the full report (for the email), then the gate (`--exit-code 1` fails the build). `--ignore-unfixed` only fails when a fix exists.

Check:
- All stages green
- Stage view shows Docker Build → TRIVY → Docker Push

## 51. Image tags per build + automatic deployment

Each build gets its own image tag (`emlykf/netflix:<build number>`). Jenkins writes the tag into `kubernetes/deployment.yml` and pushes it to GitHub, and Argo CD deploys it automatically. The test container on port 8081 is removed.

Create a GitHub token: GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens** → **Generate new token**:
- **Token name:** `netflix-jenkins`
- **Expiration:** `30 days`
- **Repository access:** **Only select repositories** → `devsecops-netflix-k8s`
- **Permissions** → **Add permissions** → search `Contents` → **Read and write**

→ **Generate token** and copy it.

Jenkins → **Manage Jenkins** → **Credentials** → **System** → **Global credentials (unrestricted)** → **Add Credentials**:
- **Kind:** `Username with password`
- **Username:** `emlykf`
- **Password:** the GitHub token
- **ID:** `github-token`

Update the pipeline (job → **Configure** → **Pipeline**):

Add to `environment`:
```groovy
IMAGE = "emlykf/netflix:${BUILD_NUMBER}"
```

Use `$IMAGE` instead of `netflix` / `emlykf/netflix:latest` in Docker Build, TRIVY and Docker Push (remove the `docker tag` line):
```groovy
sh 'docker build --build-arg TMDB_V3_API_KEY=$TMDB_KEY -t $IMAGE .'
sh 'trivy image $IMAGE > trivyimage.txt'
sh 'trivy image --skip-db-update --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 $IMAGE'
sh 'docker push $IMAGE'
```

Replace the "Deploy to container" stage with:
```groovy
stage('Update Kubernetes Manifest') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh '''
                git config user.name "Jenkins"
                git config user.email "jenkins@netflix-jenkins"
                sed -i "s|image: emlykf/netflix:.*|image: $IMAGE|" kubernetes/deployment.yml
                git add kubernetes/deployment.yml
                git commit -m "Update image to $IMAGE"
                git push https://$GIT_USER:$GIT_TOKEN@github.com/emlykf/devsecops-netflix-k8s.git HEAD:main
            '''
        }
    }
}
```

Remove the old test container on netflix-jenkins:
```bash
docker rm -f netflix
```

Close port 8081: EC2 → **Instances** → `netflix-jenkins` → **Security** tab → security group → **Edit inbound rules** → delete the **8081** rule → **Save rules**.

> Jenkins now pushes commits. Run `git pull` before changing anything locally.

Check:
- GitHub shows a commit by Jenkins: "Update image to emlykf/netflix:<build number>"
- Argo CD → netflix: **Synced** + **Healthy**, pod image `emlykf/netflix:<build number>`
- `http://<k8s-eip>:30007` loads the app

## Clean Up

1. **EC2** → **Elastic IPs**: select `netflix-eip`, `monitoring-eip` and `k8s-eip` → **Actions** → **Disassociate**, then **Release**.
2. **EC2** → **Instances**: terminate `netflix-jenkins`, `netflix-monitoring` and `netflix-k8s`.
3. **EC2** → **Security Groups**: delete the security groups of all three instances.
4. Delete the tokens and passwords:
   - SonarQube token (goes away with the instance)
   - Docker Hub token: **Account settings** → **Personal access tokens**
   - GitHub token `netflix-jenkins`: **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
   - Google App Password `Netflix`: [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
5. Delete the TMDB API key and the TMDB account.
6. Docker Hub: delete the repo `<your-dockerhub-username>/netflix` if you don't need it (**Repository** → **Settings** → **Delete repository**).