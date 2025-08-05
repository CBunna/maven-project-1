# Jenkins, Maven, and SonarQube Setup Guide

## Overview
This guide provides step-by-step instructions for setting up a complete CI/CD pipeline using Jenkins, Maven, and SonarQube on Ubuntu servers.

## Architecture
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Jenkins   │───▶│    Maven    │───▶│  SonarQube  │
│   Server    │    │   Server    │    │   Server    │
│             │    │             │    │             │
│ Port: 8080  │    │ Build Tool  │    │ Port: 9000  │
└─────────────┘    └─────────────┘    └─────────────┘
```

## Prerequisites

### System Requirements
- **Operating System**: Ubuntu 18.04+ (64-bit)
- **Java**: OpenJDK 17 or higher
- **Memory**: Minimum 8GB RAM (4GB for each server)
- **Disk Space**: Minimum 50GB free space
- **Network**: Servers should be able to communicate with each other

### Required Ports
- **Jenkins**: 8080
- **SonarQube**: 9000
- **SSH**: 22 (for communication between servers)

---

## Part 1: Jenkins Server Setup

### Step 1: Install Java
```bash
# Update system packages
sudo apt update
sudo apt upgrade -y

# Install OpenJDK 17
sudo apt install openjdk-17-jdk -y

# Verify Java installation
java -version
javac -version

# Set JAVA_HOME
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' >> ~/.bashrc
source ~/.bashrc
```

### Step 2: Install Jenkins
```bash
# Add Jenkins repository key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Add Jenkins repository
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update packages and install Jenkins
sudo apt update
sudo apt install jenkins -y

# Start and enable Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Check Jenkins status
sudo systemctl status jenkins
```

### Step 3: Initial Jenkins Configuration
```bash
# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Access Jenkins web interface
# http://your-jenkins-server:8080
```

**Web Configuration:**
1. **Enter initial admin password**
2. **Install suggested plugins**
3. **Create admin user**
4. **Configure Jenkins URL**

### Step 4: Install Required Jenkins Plugins
**Go to**: Manage Jenkins → Plugins → Available plugins

**Install these plugins:**
- Pipeline Maven Integration Plugin
- SonarQube Scanner for Jenkins
- Git Plugin
- JUnit Plugin
- SSH Build Agents Plugin

---

## Part 2: Maven Installation

### Option A: Install Maven on Jenkins Server

```bash
# SSH to Jenkins server
ssh jenkins-user@jenkins-server

# Remove any existing Maven
sudo apt remove maven -y

# Download Maven 3.9.11
cd /tmp
wget https://archive.apache.org/dist/maven/maven-3/3.9.11/binaries/apache-maven-3.9.11-bin.tar.gz

# Extract and install
sudo tar -xzf apache-maven-3.9.11-bin.tar.gz -C /opt/
sudo ln -sf /opt/apache-maven-3.9.11 /opt/maven

# Set ownership
sudo chown -R jenkins:jenkins /opt/apache-maven-3.9.11

# Configure environment for jenkins user
sudo -u jenkins bash -c 'cat >> ~/.bashrc << EOF
export MAVEN_HOME=/opt/maven
export PATH=\$MAVEN_HOME/bin:\$PATH
EOF'

# Apply changes
sudo -u jenkins bash -c 'source ~/.bashrc'

# Verify installation
sudo -u jenkins mvn -version
```

### Option B: Configure Maven in Jenkins (Alternative)

**Jenkins Web Interface:**
1. **Go to**: Manage Jenkins → Tools
2. **Maven section**: Click "Add Maven"
3. **Configuration**:
   - **Name**: `Maven-3.9.11`
   - **Install automatically**: ✅
   - **Version**: 3.9.11
4. **Save**

---

## Part 3: SonarQube Server Setup

### Step 1: Install Prerequisites
```bash
# Update system
sudo apt update
sudo apt upgrade -y

# Install Java 17
sudo apt install openjdk-17-jdk -y

# Install PostgreSQL
sudo apt install postgresql postgresql-contrib -y

# Start and enable PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### Step 2: Configure PostgreSQL
```bash
# Switch to postgres user and create database
sudo -u postgres createuser sonarqube
sudo -u postgres createdb sonarqube -O sonarqube

# Set password for sonarqube user
sudo -u postgres psql -c "ALTER USER sonarqube WITH ENCRYPTED PASSWORD 'sonar123';"

# Grant privileges
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonarqube;"

# Verify database creation
sudo -u postgres psql -l | grep sonarqube
```

### Step 3: System Configuration
```bash
# Configure system limits
sudo tee -a /etc/security/limits.conf << EOF
sonarqube   -   nofile   65536
sonarqube   -   nproc    4096
EOF

# Configure kernel parameters
sudo tee -a /etc/sysctl.conf << EOF
vm.max_map_count=524288
fs.file-max=131072
EOF

# Apply sysctl changes
sudo sysctl -p
```

### Step 4: Install SonarQube
```bash
# Create sonarqube user
sudo useradd -r -d /opt/sonarqube -s /bin/false sonarqube

# Download SonarQube (latest version)
cd /tmp
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-25.7.0.110598.zip

# Extract and install
sudo unzip sonarqube-25.7.0.110598.zip -d /opt/
sudo mv /opt/sonarqube-25.7.0.110598 /opt/sonarqube

# Set ownership
sudo chown -R sonarqube:sonarqube /opt/sonarqube
```

### Step 5: Configure SonarQube
```bash
# Edit SonarQube configuration
sudo nano /opt/sonarqube/conf/sonar.properties
```

**Add these lines to sonar.properties:**
```properties
# Database configuration
sonar.jdbc.username=sonarqube
sonar.jdbc.password=sonar123
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube

# Web server configuration
sonar.web.host=0.0.0.0
sonar.web.port=9000

# SonarQube server configuration
sonar.path.data=/opt/sonarqube/data
sonar.path.temp=/opt/sonarqube/temp
```

### Step 6: Create SonarQube Service
```bash
# Create systemd service file
sudo tee /etc/systemd/system/sonarqube.service << EOF
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonarqube
Group=sonarqube
Restart=always
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd and start SonarQube
sudo systemctl daemon-reload
sudo systemctl enable sonarqube
sudo systemctl start sonarqube

# Check status
sudo systemctl status sonarqube

# Monitor logs
sudo tail -f /opt/sonarqube/logs/sonar.log
```

### Step 7: Initial SonarQube Configuration
```bash
# Wait for SonarQube to start (2-5 minutes)
# Access web interface: http://your-sonar-server:9000

# Default credentials: admin/admin
```

**Web Configuration:**
1. **Login with admin/admin**
2. **Change default password**
3. **Generate authentication token**:
   - User Account → Security → Generate Tokens
   - Name: `Jenkins`
   - Copy the generated token

---

## Part 4: Integration Configuration

### Step 1: Configure SonarQube in Jenkins

**Jenkins Web Interface:**
1. **Go to**: Manage Jenkins → Configure System
2. **Find**: SonarQube servers section
3. **Add SonarQube server**:
   - **Name**: `sonarqube-v25.7.0`
   - **Server URL**: `http://your-sonar-server:9000`
   - **Server authentication token**: Paste your SonarQube token
4. **Save configuration**

### Step 2: Create Project in SonarQube

**SonarQube Web Interface:**
1. **Go to**: Projects → Create Project
2. **Choose**: Manually
3. **Project Settings**:
   - **Project key**: `maven-project-1`
   - **Display name**: `Maven Project 1`
4. **Click**: Set Up
5. **Choose**: With Jenkins

### Step 3: Test Connectivity
```bash
# From Jenkins server, test SonarQube connection
curl -I http://your-sonar-server:9000

# Test API with token
curl -H "Authorization: Bearer YOUR_TOKEN" http://your-sonar-server:9000/api/server/version
```
