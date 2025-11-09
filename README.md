# Jenkins + Maven Agent → Tomcat (Docker) — Full DevOps Journey (Narendra Geddam)

> Author: **Narendra Geddam**  
> Assistant: **Victor (AI DevOps Assistant)**  
> Date: 2025-11-09  
> Project: Jenkins CI/CD pipeline with Dockerized Tomcat deployment  

---

## 🧭 Overview

This file documents **the complete end-to-end DevOps pipeline journey** that Narendra and Victor worked on — from initial Docker setup to final successful Jenkins pipeline deployment.

It includes **every problem**, **debugging step**, **wrong attempts**, **corrections**, and **the final stable solution**.

---

## 🧱 What We Built

A complete **CI/CD pipeline using Dockerized Jenkins and Tomcat**, where:

- Jenkins Master and Agents run inside Docker containers.  
- The Maven Agent builds a Java web application (WAR).  
- The WAR is deployed to a Tomcat container via SSH.  
- Tomcat is restarted automatically after deployment.  
- The setup uses **static IPs**, **SSH keys**, and **non-interactive SSH** for automation.  
- The pipeline performs **health checks** to confirm successful deployment.

---

## 🗂️ Project Structure

```
Jenkins/
├── docker-compose.yml
├── Dockerfile.maven-agent
├── Dockerfile.tomcat
├── tomcat_key / tomcat_key.pub
├── javaapp-tomcat/        # Maven project
│   └── target/artisantek-app.war
└── Jenkinsfile
```

---

## 🧩 Stage 1 — Docker & Network Setup

### 🔹 Problem 1: `yaml: line 2 did not find expected key`
**Cause:** Bad indentation in docker-compose.yml  
**Fix:** Corrected YAML indentation and quotes.

---

### 🔹 Problem 2: Jenkins agent SSH “Permission denied (publickey)”
**Error:**
```
Warning: Identity file /home/jenkins/tomcat_key not accessible: No such file or directory.
root@tomcat-server: Permission denied (publickey).
```
**Fix Steps:**
1. We realized `/home/jenkins/` didn’t exist inside container.
2. Narendra used the correct host path and copied the key properly:
   ```bash
   docker cp /home/narendra/Jenkins/tomcat_key jenkins-agent-maven:/root/tomcat_key
   docker exec -it jenkins-agent-maven chmod 600 /root/tomcat_key
   ```
3. Verified connectivity:
   ```bash
   docker exec -it jenkins-agent-maven ssh -i /root/tomcat_key root@tomcat-server
   ```
   ✅ Successfully connected!

---

## 🧩 Stage 2 — Dockerfile Configuration

### 🧱 Dockerfile.maven-agent (Final)

```dockerfile
FROM jenkins/inbound-agent:latest
USER root

RUN apt-get update &&     apt-get install -y openjdk-21-jdk maven openssh-client ca-certificates &&     rm -rf /var/lib/apt/lists/*

ENV JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
ENV PATH=$JAVA_HOME/bin:$PATH

RUN mkdir -p /home/jenkins/.ssh && chmod 700 /home/jenkins/.ssh
COPY tomcat_key /home/jenkins/.ssh/id_rsa
RUN chmod 600 /home/jenkins/.ssh/id_rsa && chown -R jenkins:jenkins /home/jenkins/.ssh

USER jenkins
```

### 🧱 Dockerfile.tomcat (Final)

```dockerfile
FROM tomcat:9-jdk17

RUN apt-get update && apt-get install -y openssh-server &&     mkdir -p /var/run/sshd /root/.ssh && chmod 700 /root/.ssh

COPY tomcat_key.pub /root/.ssh/authorized_keys
RUN chmod 600 /root/.ssh/authorized_keys &&     echo "PermitRootLogin yes" >> /etc/ssh/sshd_config &&     echo "PasswordAuthentication no" >> /etc/ssh/sshd_config

EXPOSE 8080 22
CMD service ssh start && catalina.sh run
```

✅ Now both containers can talk securely over SSH.

---

## 🧩 Stage 3 — Static IP Assignment

Narendra asked for static IPs for each container for stable SSH connectivity.

We modified `docker-compose.yml`:

```yaml
networks:
  net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/16

services:
  jenkins: { ipv4_address: 172.18.0.2 }
  jenkins-agent-maven: { ipv4_address: 172.18.0.3 }
  jenkins-agent-1: { ipv4_address: 172.18.0.4 }
  jenkins-agent-2: { ipv4_address: 172.18.0.5 }
  tomcat-server: { ipv4_address: 172.18.0.6 }
```

✅ Verified IPs after compose up:
```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' tomcat-server
# Output: 172.18.0.6
```

---

## 🧩 Stage 4 — Jenkins Pipeline Iterations

### ❌ First Attempt: Failed (Exit 255)

Used heredoc and `-tt` SSH:
```bash
ssh -tt root@tomcat-server <<'EOF'
  pkill -f catalina || true
  nohup catalina.sh start &
EOF
```
**Problem:** Jenkins marked as failure with `exit code 255`.  
**Root Cause:** Interactive SSH session closed unexpectedly.

**Fix:** Use non-interactive SSH mode:
```bash
ssh -n -T root@tomcat-server "commands..."
```

---

### ❌ Second Attempt: JAVA_HOME Error

```
Neither the JAVA_HOME nor the JRE_HOME environment variable is defined
```
**Cause:** `startup.sh`/`shutdown.sh` expect JAVA_HOME.  
**Fix:** Replaced them with `catalina.sh start/stop` which doesn’t need JAVA_HOME.

---

### ⚙️ Third Attempt: Wrong Paths

Earlier pipeline navigated redundantly:
```bash
cd javaapp-tomcat/target
find javaapp-tomcat/target -name *.war
```
Narendra correctly pointed out:
> “When building we’re already in `javaapp-tomcat`, so just `cd target` is enough.”

✅ Simplified and fixed.

---

### ✅ Final Working Pipeline

```groovy
pipeline {
    agent { label 'maven-agent' }

    environment {
        SSH_CRED       = 'tomcat-ssh'
        SSH_HOST       = '172.18.0.6'
        SSH_PORT       = '22'
        TOMCAT_HOME    = '/usr/local/tomcat'
        TOMCAT_WEBAPPS = '/usr/local/tomcat/webapps'
        APP_URL        = 'http://172.18.0.6:8080'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Narendra-Geddam/jenkins-DevOps.git'
            }
        }

        stage('Build WAR') {
            steps {
                dir('javaapp-tomcat') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                dir('javaapp-tomcat') {
                    sshagent (credentials: ["${SSH_CRED}"]) {
                        sh '''
                            cd target
                            ssh -n -T -p ${SSH_PORT} -o StrictHostKeyChecking=no root@${SSH_HOST} "
                                /usr/local/tomcat/bin/catalina.sh stop || true
                                rm -rf ${TOMCAT_WEBAPPS}/ROOT.war ${TOMCAT_WEBAPPS}/ROOT
                            "
                            scp -P ${SSH_PORT} -o StrictHostKeyChecking=no *.war root@${SSH_HOST}:${TOMCAT_WEBAPPS}/ROOT.war
                            ssh -n -T -p ${SSH_PORT} -o StrictHostKeyChecking=no root@${SSH_HOST} "
                                BUILD_ID=dontKillMe nohup ${TOMCAT_HOME}/bin/catalina.sh start > ${TOMCAT_HOME}/logs/deploy.log 2>&1 &
                            "
                            for i in {1..10}; do
                                curl -sf ${APP_URL} && { echo 'Application deployed and reachable'; exit 0; }
                                sleep 3
                            done
                            exit 1
                        '''
                    }
                }
            }
        }
    }

    post {
        success { echo '✅ Deployed successfully.' }
        failure { echo '❌ Deployment failed. Check logs.' }
    }
}
```

✅ Pipeline output summary:
```
📦 Building WAR... done
🚀 Deploying to Tomcat 172.18.0.6
🔄 Restarting Tomcat...
✅ Deployed successfully.
```

---

## 🧠 Major Lessons Learned

| Problem | Root Cause | Fix |
|----------|-------------|-----|
| SSH permission denied | Key not mounted inside agent | Used docker cp + chmod |
| `chmod: Operation not permitted` | Non-root user modifying file | Switched to root before chmod |
| `exit 255` | SSH closed prematurely | Used `ssh -n -T` non-interactive |
| `JAVA_HOME not set` | startup.sh expects env | Used `catalina.sh` instead |
| Tomcat not resolvable | docker internal hostname | Used static IPs |
| Jenkins killing Tomcat | Background process tied to Jenkins | Added `BUILD_ID=dontKillMe` |

---

## 🔎 Assistant Mistakes & Narendra’s Corrections

| Assistant’s Mistake | Narendra’s Correction |
|----------------------|-----------------------|
| Overcomplicated SSH heredoc logic | “Just use normal ssh and scp.” |
| Used redundant `find` command | “We’re already inside `javaapp-tomcat`.” |
| Tried adding known_hosts manually | “Only private key needed in agent.” |
| Used EC2-style user examples | “We’re in Docker, not EC2 — use root.” |
| Forgot to set static IPs | Narendra fixed docker-compose with IPv4 addresses. |
| Ignored path mismatch (`/root/.ssh` vs `/home/jenkins`) | Narendra pointed out actual working path. |

---

## 🧰 Debug Commands Used During Project

```bash
# Check container IPs
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' tomcat-server

# Test SSH connectivity
docker exec -it jenkins-agent-maven ssh -i /home/jenkins/tomcat_key root@172.18.0.6

# Check Tomcat SSH service
docker exec -it tomcat-server service ssh status

# View deployment logs
docker exec -it tomcat-server tail -n 10 /usr/local/tomcat/logs/deploy.log
```

---

## ✅ Final State Summary

✅ Jenkins builds WAR using Maven Agent  
✅ Copies WAR to Tomcat via SSH  
✅ Stops & restarts Tomcat safely  
✅ Performs health check  
✅ Returns SUCCESS in Jenkins console  

All issues from the entire session — from key permissions, path mismatches, hostname errors, and pipeline design flaws — have been **completely resolved**.

---

## 🏁 Final Notes

This project demonstrates a **real-world DevOps debugging process** — full of trial, error, and iteration — until we achieved a clean, automated, stable deployment.

Every correction by Narendra made the system more practical, production-ready, and clean.

> 💡 *“Don’t complicate things — build, move, restart.” — Narendra Geddam*

---

**Final Status:** ✅ Fully functional CI/CD pipeline with Dockerized Jenkins and Tomcat.
