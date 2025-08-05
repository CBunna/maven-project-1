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

---

## Part 5: Create Jenkins Pipeline

### Step 1: Create Pipeline Job
**Jenkins Web Interface:**
1. **Click**: New Item
2. **Name**: `sonarqube-pipeline`
3. **Type**: Pipeline
4. **Click**: OK

### Step 2: Configure Pipeline Script
```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven-3.9.11'  // Use if configured via Jenkins Tools
    }
    
    stages {
        stage('Environment Check') {
            steps {
                echo '=== Environment Information ==='
                sh '''
                    echo "Java Version:"
                    java -version
                    echo "Maven Version:"
                    mvn -version
                    echo "Current Directory:"
                    pwd
                '''
            }
        }
        
        stage('Get Code') {
            steps {
                echo '=== Checking out source code ==='
                git branch: 'main',
                    url: 'https://github.com/CBunna/maven-project-1.git'
                
                sh 'ls -la'
            }
        }
        
        stage('Build') {
            steps {
                echo '=== Building Maven Project ==='
                sh 'mvn clean compile'
                sh 'mvn package -DskipTests'
            }
            post {
                success {
                    echo 'Build completed successfully!'
                    archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
                }
                failure {
                    echo 'Build failed!'
                }
            }
        }
        
        stage('Test') {
            steps {
                echo '=== Running Unit Tests ==='
                sh 'mvn test'
            }
            post {
                always {
                    publishTestResults testResultsPattern: '**/target/surefire-reports/*.xml'
                }
                success {
                    echo 'Tests passed successfully!'
                }
                failure {
                    echo 'Some tests failed!'
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                echo '=== Running SonarQube Analysis ==='
                withSonarQubeEnv('sonarqube-v25.7.0') {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=maven-project-1 \
                        -Dsonar.projectName="Maven Project 1" \
                        -Dsonar.projectVersion=1.0
                    '''
                }
            }
            post {
                success {
                    echo 'SonarQube analysis completed successfully!'
                }
                failure {
                    echo 'SonarQube analysis failed!'
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                echo '=== Waiting for Quality Gate Result ==='
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
            post {
                success {
                    echo 'Quality Gate passed!'
                }
                failure {
                    echo 'Quality Gate failed!'
                }
            }
        }
    }
    
    post {
        always {
            echo '=== Pipeline completed ==='
        }
        success {
            echo '=== Pipeline succeeded! ==='
        }
        failure {
            echo '=== Pipeline failed! ==='
        }
    }
}
```

### Step 3: Alternative Simple Pipeline (No Tools Configuration)
```groovy
pipeline {
    agent any
    
    environment {
        MAVEN_HOME = '/opt/maven'
        PATH = "${MAVEN_HOME}/bin:${env.PATH}"
    }
    
    stages {
        stage('Get Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/CBunna/maven-project-1.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-v25.7.0') {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=maven-project-1 \
                        -Dsonar.projectName="Maven Project 1"
                    '''
                }
            }
        }
    }
}
```

---

## Part 6: Testing and Verification

### Step 1: Run Pipeline
1. **Go to your pipeline job**
2. **Click**: Build Now
3. **Monitor**: Console Output

### Step 2: Verify Results
**Check each stage:**
- ✅ **Environment Check**: Java and Maven versions displayed
- ✅ **Get Code**: Source code downloaded from Git
- ✅ **Build**: Maven build successful, artifacts created
- ✅ **Test**: Unit tests executed, results published
- ✅ **SonarQube Analysis**: Code analysis completed
- ✅ **Quality Gate**: Quality gate status received

### Step 3: View Results
**SonarQube Dashboard:**
- Go to: `http://your-sonar-server:9000`
- **Projects**: View your project analysis
- **Issues**: Review code quality issues
- **Coverage**: Check test coverage
- **Duplications**: Review code duplications

---

## Troubleshooting

### Common Issues and Solutions

#### 1. Jenkins Won't Start
```bash
# Check Jenkins status
sudo systemctl status jenkins

# Check logs
sudo journalctl -u jenkins -f

# Common fixes
sudo systemctl restart jenkins
sudo chown -R jenkins:jenkins /var/lib/jenkins
```

#### 2. Maven Not Found
```bash
# Check Maven installation
mvn -version
which mvn

# Fix PATH
export PATH=$PATH:/opt/maven/bin
echo 'export PATH=$PATH:/opt/maven/bin' >> ~/.bashrc
```

#### 3. SonarQube Won't Start
```bash
# Check SonarQube status
sudo systemctl status sonarqube

# Check logs
sudo tail -f /opt/sonarqube/logs/sonar.log

# Common fixes
sudo systemctl restart postgresql
sudo systemctl restart sonarqube
```

#### 4. SonarQube Analysis Authorization Error
```bash
# Create project in SonarQube first
# Generate and configure authentication token
# Check Jenkins SonarQube server configuration
```

#### 5. Maven Version Requirements
```bash
# Error: Maven version not in allowed range
# Solution: Upgrade Maven to required version

# Check project requirements in pom.xml
grep -A 5 "maven-enforcer-plugin" pom.xml
```

#### 6. Port Already in Use
```bash
# Check what's using the port
sudo ss -tlnp | grep :8080  # Jenkins
sudo ss -tlnp | grep :9000  # SonarQube

# Kill process or change port
sudo kill -9 <PID>
```

### Useful Commands

#### Jenkins
```bash
# Restart Jenkins
sudo systemctl restart jenkins

# Check Jenkins logs
sudo journalctl -u jenkins -f

# Jenkins configuration location
/var/lib/jenkins/
```

#### Maven
```bash
# Check Maven version
mvn -version

# Clean and build project
mvn clean compile package

# Run tests
mvn test
```

#### SonarQube
```bash
# Restart SonarQube
sudo systemctl restart sonarqube

# Check SonarQube logs
sudo tail -f /opt/sonarqube/logs/sonar.log

# Check SonarQube version via API
curl http://localhost:9000/api/server/version
```

---

## Security Best Practices

### 1. System Security
- **Update systems** regularly: `sudo apt update && sudo apt upgrade`
- **Use firewalls**: Configure UFW or iptables
- **Strong passwords**: Use complex passwords for all accounts
- **SSH keys**: Use SSH key authentication instead of passwords

### 2. Jenkins Security
- **Change default credentials** immediately
- **Install security updates** regularly
- **Use HTTPS** in production
- **Limit user permissions** appropriately
- **Regular backups** of Jenkins configuration

### 3. SonarQube Security
- **Change default admin password**
- **Configure proper authentication** (LDAP/Active Directory)
- **Use HTTPS** in production
- **Regular security scans** of your SonarQube instance
- **Database security**: Secure PostgreSQL installation

### 4. Network Security
```bash
# Configure firewall
sudo ufw allow 22    # SSH
sudo ufw allow 8080  # Jenkins
sudo ufw allow 9000  # SonarQube
sudo ufw enable

# Or use specific source IPs
sudo ufw allow from 192.168.1.0/24 to any port 8080
```

---

## Backup and Maintenance

### Jenkins Backup
```bash
# Backup Jenkins home directory
sudo tar -czf jenkins-backup-$(date +%Y%m%d).tar.gz /var/lib/jenkins/

# Automated backup script
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
sudo tar -czf /backup/jenkins-backup-$DATE.tar.gz /var/lib/jenkins/
find /backup -name "jenkins-backup-*.tar.gz" -mtime +7 -delete
```

### SonarQube Backup
```bash
# Stop SonarQube
sudo systemctl stop sonarqube

# Backup database
sudo -u postgres pg_dump sonarqube > sonarqube-db-backup-$(date +%Y%m%d).sql

# Backup SonarQube data
sudo tar -czf sonarqube-data-backup-$(date +%Y%m%d).tar.gz /opt/sonarqube/data/

# Start SonarQube
sudo systemctl start sonarqube
```

### Regular Maintenance
```bash
# Update systems monthly
sudo apt update && sudo apt upgrade

# Clean up old builds (Jenkins)
# Configure in Jenkins: Manage Jenkins → Configure System → Build Record

# Monitor disk space
df -h
du -sh /var/lib/jenkins/
du -sh /opt/sonarqube/
```

---

## Additional Resources

### Documentation Links
- **Jenkins**: https://www.jenkins.io/doc/
- **Maven**: https://maven.apache.org/guides/
- **SonarQube**: https://docs.sonarsource.com/sonarqube-server/

### Useful Jenkins Plugins
- **Blue Ocean**: Modern UI for Jenkins pipelines
- **Build Timeout**: Automatically timeout builds
- **Workspace Cleanup**: Clean workspace after builds
- **Email Extension**: Advanced email notifications
- **Slack Notification**: Send notifications to Slack

### SonarQube Quality Profiles
- **Sonar way**: Default quality profile
- **Custom profiles**: Create organization-specific rules
- **Rule management**: Enable/disable specific rules
- **Quality gates**: Set quality thresholds

---

## Conclusion

You now have a complete CI/CD pipeline with:
- ✅ **Jenkins** for automation and orchestration
- ✅ **Maven** for building Java projects
- ✅ **SonarQube** for code quality analysis
- ✅ **Integration** between all components
- ✅ **Security** best practices
- ✅ **Backup** and maintenance procedures

The pipeline will automatically:
1. **Pull code** from your Git repository
2. **Build** the project with Maven
3. **Run tests** and publish results
4. **Analyze code quality** with SonarQube
5. **Check quality gates** before deployment
6. **Archive artifacts** for deployment

**Remember to**:
- Keep all systems updated
- Monitor logs regularly
- Perform regular backups
- Follow security best practices
- Document any customizations

**Happy coding and building!** 🚀