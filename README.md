# Jenkins Freestyle CI/CD Pipeline with 3 EC2 Instances (Amazon Linux)

## Overview

| Role     | Server | Tools Installed                     |
|----------|--------|-------------------------------------|
| Master   | t2.micro | Jenkins, Java 17                  |
| Slave 1  | t2.micro | Git, Maven, Java 17               |
| Slave 2  | t2.micro | Tomcat 9, Java 17                 |

## EC2 Setup (Common for All)

1. Use Amazon Linux 2 AMI
2. Security Group:
   - Port 8080 (Tomcat)
   - Port 22 (SSH)
   - Port 8080 (Jenkins)
3. Set hostname:
   ```
   sudo hostnamectl set-hostname <jenkins-master/slave1/slave2>
   ```

---

## Jenkins Master Setup

```bash
sudo yum update -y
sudo yum install -y java-17-amazon-corretto
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade
sudo yum install -y jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Initial password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## SSH User Setup (on Slaves)

```bash
sudo useradd -m -s /bin/bash jenkins
echo "jenkins:jenkins" | sudo chpasswd
sudo mkdir -p /home/jenkins/.ssh
sudo chmod 700 /home/jenkins/.ssh
sudo chown -R jenkins:jenkins /home/jenkins/.ssh
```

---

## SSH Key Setup (from Master)

```bash
ssh-keygen -t rsa -b 2048 -N "" -f ~/.ssh/id_rsa
ssh-copy-id jenkins@<slave1-ip>
ssh-copy-id jenkins@<slave2-ip>
```

---

## Slave 1 Setup (Build Node)

```bash
sudo yum install -y git java-17-amazon-corretto

# Maven
wget https://downloads.apache.org/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz -P /tmp
sudo tar -xzf /tmp/apache-maven-3.9.6-bin.tar.gz -C /opt
sudo ln -s /opt/apache-maven-3.9.6 /opt/maven

# Environment Variables
echo 'export M2_HOME=/opt/maven' | sudo tee -a /etc/profile
echo 'export PATH=$PATH:/opt/maven/bin' | sudo tee -a /etc/profile
source /etc/profile
```

---

## Slave 2 Setup (Deployment Node)

```bash
sudo yum install -y java-17-amazon-corretto

cd /opt
sudo wget https://downloads.apache.org/tomcat/tomcat-9/v9.0.85/bin/apache-tomcat-9.0.85.tar.gz
sudo tar -xvzf apache-tomcat-9.0.85.tar.gz
sudo mv apache-tomcat-9.0.85 tomcat9
sudo chmod +x /opt/tomcat9/bin/*.sh

# Start Tomcat
sudo /opt/tomcat9/bin/startup.sh
```

---

## Jenkins Node Configuration

1. Manage Jenkins → Manage Nodes and Clouds → New Node
2. Type: Permanent
3. Labels: `build` (Slave 1), `deploy` (Slave 2)
4. Launch via SSH, Remote directory: `/home/jenkins`

---

## Jenkins Freestyle Job Setup

1. New Item → Freestyle project → `build-and-deploy`
2. **Restrict where to run**: `build`
3. **Git Repo**: `https://github.com/suneelprojects/jenkins-project.git`

### Build Step

```bash
mvn clean package
```

### Post-build: Deploy WAR to Slave 2 (via SSH)

- Install **Publish Over SSH Plugin**
- Manage Jenkins → Configure System → Publish over SSH
- Add:
  - Name: `slave2`
  - Host: `<slave2-ip>`
  - Username: `jenkins`
  - Remote Dir: `/opt/tomcat9/webapps/`

#### Configure Post-build Action:

- Source files:
  ```
  target/*.war
  ```
- Remote dir:
  ```
  /opt/tomcat9/webapps/
  ```
- Exec command:
  ```bash
  /opt/tomcat9/bin/shutdown.sh
  sleep 5
  /opt/tomcat9/bin/startup.sh
  ```

---
✅ Run It!
Click Build Now

It will:

Clone the repo on Slave 1

Build WAR file

Transfer to Slave 2

Deploy to Tomcat

## Done!

Access your app at:
```
http://<slave2-ip>:8080/<war-name>
```
