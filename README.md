# 🚀 CI/CD Integration: GitHub → Jenkins → SonarQube → Nexus → Tomcat

A complete end-to-end automation pipeline integrating **GitHub**, **Jenkins**, **SonarQube**, **Nexus**, and **Tomcat** for continuous integration, static analysis, artifact management, and automated deployment.

---

## 🎯 Project Goal

On every push to the main branch of your GitHub repository:

1. **Jenkins** checks out the latest code.
2. **SonarQube** performs static code analysis.
3. **Maven** builds the project into a `.war` file.
4. The artifact is uploaded to **Nexus (maven-releases)**.
5. Jenkins fetches the latest `.war` from Nexus.
6. The **WAR** is automatically deployed to **Tomcat** using the Manager API.

Everything is orchestrated through a **Jenkinsfile**, triggered by a **GitHub webhook**.

---

## 🧰 Infrastructure Requirements

Provision the following (Ubuntu 22.04 LTS recommended):

| Component | Purpose | Example IP |
|------------|----------|------------|
| Jenkins Master | CI/CD orchestration | Handles builds, triggers, pipelines |
| SonarQube Server | Code analysis + build agent | Can double as a Jenkins agent |
| Nexus Repository | Artifact storage  | Host for `maven-releases` |
| Tomcat Server | Deployment runtime  | Webapp deployment target |
<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/335abfa7-898d-48e1-a492-71f3ce0193fa" />
<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/ac3144a8-ca87-4bcd-87a6-fadecbc24d8c" />
<img width="1920" height="1080" alt="3" src="https://github.com/user-attachments/assets/d103435a-b6f4-4822-beed-fbc00ae9c990" />
<img width="1920" height="1080" alt="4" src="https://github.com/user-attachments/assets/611943d8-4956-4164-9ae3-d9a986e5196d" />

- **Jenkins Agents:**
  - `SonarNode` → build + analysis
  - `TomcatNode` → deployment

### Network & Access
- Jenkins must reach SonarQube, Nexus, and Tomcat.
- Agents must reach Nexus and Tomcat.
- SSH key-based access from Jenkins → agents.
- Required credentials:
  - GitHub Personal Access Token (PAT)
  - Nexus user
  - Tomcat manager user (manager-script role)
  - SonarQube token

---

## ⚙️ Step-by-Step Setup

### 1 — Create & Prepare VMs
Run on each VM:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git unzip jq ufw
sudo ufw allow OpenSSH && sudo ufw enable
sudo apt install -y openjdk-17-jdk
sudo mkdir -p /home/ubuntu/jenkins && sudo chown ubuntu:ubuntu /home/ubuntu/jenkins
````

---

### 2 — Install & Configure SonarQube

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk unzip
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.6.0.92116.zip
sudo unzip sonarqube-10.6.0.92116.zip
sudo mv sonarqube-10.6.0.92116 /opt/sonarqube
sudo useradd -r -s /bin/false sonar
sudo chown -R sonar:sonar /opt/sonarqube
```
<img width="1920" height="1080" alt="9" src="https://github.com/user-attachments/assets/7c5c5e43-ba87-4e1b-84b3-1c66b24e356e" />
<img width="1920" height="1080" alt="10" src="https://github.com/user-attachments/assets/2ec6051f-11d3-4c66-ba95-be829cafd193" />
<img width="1920" height="1080" alt="11" src="https://github.com/user-attachments/assets/0a91ec06-3068-454a-82f5-848fbd8e71e7" />
<img width="1920" height="1080" alt="12" src="https://github.com/user-attachments/assets/1413323d-8159-4a18-8919-48bbc9532b19" />
<img width="1920" height="1080" alt="14" src="https://github.com/user-attachments/assets/57c56270-1ad8-4721-b8b1-72b4f9c619c8" />

Create a **systemd** service:

```bash
sudo tee /etc/systemd/system/sonarqube.service > /dev/null <<'EOF'
[Unit]
Description=SonarQube service
After=network.target

[Service]
Type=forking
User=sonar
Group=sonar
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now sonarqube
```

Access SonarQube at:
👉 `http://<SONAR_IP>:9000`
Default credentials: `admin / admin`
Create a token for Jenkins under **My Account → Security**.
<img width="1920" height="1080" alt="50" src="https://github.com/user-attachments/assets/779e5316-46a8-4ed4-ab92-7044aa4f5900" />
<img width="1920" height="1080" alt="51" src="https://github.com/user-attachments/assets/e076dd16-33b9-4854-9a4e-3722c603383d" />

---

### 3 — Install & Configure Nexus Repository

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk wget tar
cd /opt
sudo wget https://download.sonatype.com/nexus/3/nexus-3.85.0-03-linux-x86_64.tar.gz
sudo tar -xzf nexus-3.85.0-03-linux-x86_64.tar.gz
sudo mv nexus-3* nexus
sudo useradd -r -s /bin/false nexus
sudo chown -R nexus:nexus /opt/nexus /opt/sonatype-work
echo 'run_as_user="nexus"' | sudo tee /opt/nexus/bin/nexus.rc
```
<img width="1920" height="1080" alt="15" src="https://github.com/user-attachments/assets/55ef589d-9628-488c-95e5-a760b006d29e" />
<img width="1920" height="1080" alt="17" src="https://github.com/user-attachments/assets/b773a55e-a225-4dc5-ab09-31ed08248231" />
<img width="1920" height="1080" alt="18" src="https://github.com/user-attachments/assets/e430ddb1-a049-43bd-97d9-fbbac61295f6" />
<img width="1920" height="1080" alt="20" src="https://github.com/user-attachments/assets/c16b679f-8f73-441e-b89c-8dd02719fb4e" />

Create a **systemd** service:

```bash
sudo tee /etc/systemd/system/nexus.service > /dev/null <<'EOF'
[Unit]
Description=Nexus service
After=network.target

[Service]
Type=forking
User=nexus
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
Restart=on-abort

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now nexus
```

Access Nexus at:
👉 `http://<NEXUS_IP>:8081`
Initial admin password: `/opt/sonatype-work/nexus3/admin.password`

Create a hosted Maven repository:
**Administration → Repositories → Create repository → maven2 (hosted) → Name:** `maven-releases`

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/8d78a706-cf2e-479e-8c8a-0e763dbedb4d" />

---

### 4 — Install & Configure Tomcat (Manual Installation)

```bash
# Install Java (if not already installed)
sudo apt update
sudo apt install -y openjdk-17-jdk

# Download and extract Tomcat 9.0.111
cd /opt
sudo wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.111/bin/apache-tomcat-9.0.111.tar.gz
sudo tar -xzf apache-tomcat-9.0.111.tar.gz
sudo mv apache-tomcat-9.0.111 tomcat9

# Create dedicated user and adjust permissions
sudo useradd -m -U -d /opt/tomcat9 -s /bin/false tomcat
sudo chown -R tomcat:tomcat /opt/tomcat9
```
<img width="1920" height="1080" alt="21" src="https://github.com/user-attachments/assets/611126fb-e408-4f04-a08a-cbe990c1097a" />
<img width="1920" height="1080" alt="22" src="https://github.com/user-attachments/assets/acdeeef9-efe2-473a-90ba-f32451f56ecf" />
<img width="1920" height="1080" alt="23" src="https://github.com/user-attachments/assets/35cda8cf-219a-4a3c-9755-19011c1a2e6a" />
<img width="1920" height="1080" alt="24" src="https://github.com/user-attachments/assets/7d09c438-ffba-4d1c-a2fc-06cc5693ad8e" />
<img width="1920" height="1080" alt="25" src="https://github.com/user-attachments/assets/320880a8-dfaf-4414-8451-79d316aa9fcd" />

Create a **systemd service** for Tomcat:

```bash
sudo tee /etc/systemd/system/tomcat.service > /dev/null <<'EOF'
[Unit]
Description=Apache Tomcat 9.0.111 Web Application Container
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
Environment="CATALINA_PID=/opt/tomcat9/temp/tomcat.pid"
Environment="CATALINA_HOME=/opt/tomcat9"
Environment="CATALINA_BASE=/opt/tomcat9"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat9/bin/startup.sh
ExecStop=/opt/tomcat9/bin/shutdown.sh

Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now tomcat
```

Configure the **Tomcat Manager User**:

```bash
sudo nano /opt/tomcat9/conf/tomcat-users.xml
```
<img width="1920" height="1080" alt="28" src="https://github.com/user-attachments/assets/82c30a1f-42df-48e7-942f-80d7cf0f2a72" />

Add:

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<user username="admin" password="admin123" roles="manager-gui,manager-script"/>
```
<img width="1920" height="1080" alt="29" src="https://github.com/user-attachments/assets/39a46723-7205-4545-b97e-f475b76c4fc7" />

Restart Tomcat:

```bash
sudo systemctl restart tomcat
```

---

### 5 — Install & Configure Jenkins

Install Jenkins LTS:

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
```

Access Jenkins at:
👉 `http://<JENKINS_IP>:8080`
Use the initial admin password from:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

## 🔧 Jenkins Configuration

### Plugins

Install:

* Git
* GitHub Integration
* Pipeline
* Maven Integration
* SonarQube Scanner
* SSH Agents
* Credentials Binding

<img width="1920" height="1080" alt="52" src="https://github.com/user-attachments/assets/2703fa1b-affe-4e3f-bddd-ac4b3b0b9748" />
<img width="1920" height="1080" alt="53" src="https://github.com/user-attachments/assets/a429499b-2c37-42d4-8b5f-c05180adad44" />

### Global Tools

Add:

* **JDK 17**
* **Maven**

### Credentials

| ID               | Type              | Purpose            |
| ---------------- | ----------------- | ------------------ |
| `ssh-ubuntu`     | SSH key           | Jenkins → agents   |
| `nexus-creds`    | Username/password | Nexus deploy       |
| `tomcat-manager` | Username/password | Tomcat Manager API |
| `github-token`   | Secret text       | GitHub PAT         |

<img width="1920" height="1080" alt="56" src="https://github.com/user-attachments/assets/b965082f-d26c-48f8-9a22-377928087a52" />
<img width="1920" height="1080" alt="57" src="https://github.com/user-attachments/assets/924344a0-64b4-4eec-ae55-d5ec237915ce" />

### Agents

| Node       | Label        | Purpose          |
| ---------- | ------------ | ---------------- |
| SonarNode  | `SonarNode`  | Build + analysis |
| TomcatNode | `TomcatNode` | Deployment       |


<img width="1920" height="1080" alt="58" src="https://github.com/user-attachments/assets/919d4cef-b777-4d0c-b777-3040cee6ad47" />

---


---

## 🧠 Jenkinsfile (Pipeline Script)

Add this file at the **root of your repo** as `Jenkinsfile`:

```groovy
pipeline {
   agent none
   stages {

       stage('sonar-mvn') {
           agent { label 'sonar' }
           steps {
               script {
                   sh '''
                       echo "hii from sonar" >> sonar.txt
                       whoami
                       hostname -i
                   '''
               }
           }
       }

       stage('nexus') {
           agent { label 'nexus' }
           steps {
               script {
                   sh '''
                       echo "hii from nexus" >> nexus.txt
                       whoami
                       hostname -i
                   '''
               }
           }
       }

       stage('tomcat') {
           agent { label 'tomcat' }
           steps {
               script {
                   sh '''
                       echo "hii from tomcat" >> tomcat.txt
                       whoami
                       hostname -i
                   '''
               }
           }
       }
   }
}
```
<img width="1920" height="1080" alt="61" src="https://github.com/user-attachments/assets/88aebcf5-262c-481a-b925-56a0dfdeed57" />

---

## 🔔 GitHub Webhook Setup

1. In Jenkins → **New Item → Pipeline**
<img width="1920" height="1080" alt="59" src="https://github.com/user-attachments/assets/802ca99d-b61a-447a-b3bb-2dd485fe64ed" />

   * Definition: *Pipeline script from SCM*
   * SCM: *Git*
   * Repository URL: `https://github.com/you/your-app.git`
   * Branch: `*/main`
   * Script path: `Jenkinsfile`
2. Check **GitHub hook trigger for GITScm polling**.
3. In your GitHub repo → **Settings → Webhooks → Add webhook**

   * Payload URL: `http://<JENKINS_PUBLIC_IP>:8080/github-webhook/`
   * Content type: `application/json`
   * Event: *Push events*
4. Push a commit — the pipeline runs automatically.

<img width="1920" height="1080" alt="60" src="https://github.com/user-attachments/assets/09b79dde-affc-44b9-93be-9b8fb7e7e067" />

---
## Other Screenshots

<img width="1920" height="1080" alt="62" src="https://github.com/user-attachments/assets/9276df07-9358-48de-b8b4-15f95546e940" />

<img width="1920" height="1080" alt="63" src="https://github.com/user-attachments/assets/ff3b27f6-4bbc-4fd6-b89a-b6af32a06a81" />
