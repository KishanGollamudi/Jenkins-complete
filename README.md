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

| Component | Purpose | Example IP | Notes |
|------------|----------|------------|-------|
| Jenkins Master | CI/CD orchestration | — | Handles builds, triggers, pipelines |
| SonarQube Server | Code analysis + build agent | — | Can double as a Jenkins agent |
| Nexus Repository | Artifact storage | `3.227.246.21` | Host for `maven-releases` |
| Tomcat Server | Deployment runtime | `13.220.167.254` | Webapp deployment target |

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

Add:

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<user username="admin" password="admin123" roles="manager-gui,manager-script"/>
```

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

### Agents

| Node       | Label        | Purpose          |
| ---------- | ------------ | ---------------- |
| SonarNode  | `SonarNode`  | Build + analysis |
| TomcatNode | `TomcatNode` | Deployment       |

---

## 🧩 Maven Configuration

Update your project’s `pom.xml`:

```xml
<distributionManagement>
  <repository>
    <id>maven-releases</id>
    <url>http://3.227.246.21:8081/repository/maven-releases/</url>
  </repository>
</distributionManagement>
```

Create `/home/jenkins/.m2/settings.xml` on the build agent:

```xml
<settings>
  <servers>
    <server>
      <id>maven-releases</id>
      <username>admin</username>
      <password>admin123</password>
    </server>
  </servers>
</settings>
```

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

---

## 🔔 GitHub Webhook Setup

1. In Jenkins → **New Item → Pipeline**

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

---

## ✅ Validation Checklist

* [ ] Jenkinsfile present in repo root
* [ ] SonarQube reachable and token configured
* [ ] Nexus repository `maven-releases` created
* [ ] Tomcat Manager user configured
* [ ] Jenkins credentials configured
* [ ] Agents online and reachable
* [ ] GitHub webhook connected
* [ ] Successful pipeline execution and deployment

```

Would you like me to also adjust the **agent labels in the documentation** (like changing `SonarNode`, `TomcatNode` → `sonar`, `nexus`, `tomcat`) so they match your pipeline exactly? That would make it perfectly consistent end-to-end.
```
