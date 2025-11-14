# Git → GitHub → Jenkins → EC2 (Nginx) Deployment Pipeline

This document provides a clear, step-by-step explanation of how to set up and use a complete CI/CD pipeline using Git, GitHub, Jenkins (on Windows), and EC2 with Nginx, including **passwordless SSH connection** between Jenkins and EC2. It is based on the full approach explained throughout the conversation.

---

## 1. Overview of the Architecture

### ✔ Components

* **Git** – local development on your machine
* **GitHub** – remote repository storing your project
* **Jenkins (Windows)** – automation server pulling code and deploying
* **EC2 (Ubuntu)** – server hosting Nginx
* **Nginx** – web server serving your static website
* **Passwordless SSH** – secure automated connection between Jenkins → EC2

### ✔ Workflow Diagram

```
Developer (Git)
      ↓
GitHub Repository
      ↓
Jenkins (Windows)
      ↓ (Passwordless SSH)
EC2 (Ubuntu)
      ↓
Nginx Web Server → Website Live
```

---

## 2. Passwordless SSH Setup (Jenkins → EC2)

### Step 1 — Generate SSH Key Pair

Run on Jenkins server (Windows) or your local machine:

```
ssh-keygen -t rsa -b 4096 -f jenkins_deploy_key -N ""
```

This creates:

* `jenkins_deploy_key` (private key)
* `jenkins_deploy_key.pub` (public key)

### Step 2 — Add Public Key to EC2

```
ssh-copy-id -i jenkins_deploy_key.pub ubuntu@<EC2_IP>
```

OR manually append to:

```
~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Step 3 — Add Private Key to Jenkins

Navigate:

```
Manage Jenkins → Credentials → Global → Add Credentials
```

Choose:

* Kind: **SSH Username with private key**
* Username: **ubuntu**
* Private key: paste contents of `jenkins_deploy_key`
* ID: **ec2-ssh-key**

This enables Jenkins to connect to EC2 without a password.

---

## 3. Prepare EC2 with Nginx

### Install Nginx

```
sudo apt update
sudo apt install nginx -y
```

### Set Web Root Permissions (for direct deployment)

```
sudo chown -R ubuntu:ubuntu /var/www/html
sudo chmod -R 755 /var/www/html
```

Nginx default document root:

```
/var/www/html
```

---

## 4. Jenkins Pipeline Setup

A Pipeline job in Jenkins pulls the code from GitHub and deploys it to EC2.

### Jenkinsfile Used in Deployment

Below is the final working Jenkinsfile including secure handling of SSH key permissions on Windows:

````
```groovy
pipeline {
  agent any

  stages {
    stage('Checkout Code') {
      steps {
        checkout scm
      }
    }

    stage('Deploy to EC2 (direct)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY')]) {
          bat """
            echo === Deploy starting ===
            echo Workspace: %WORKSPACE%
            echo Securing temporary SSH key file: %SSH_KEY%

            REM Remove inherited ACLs so key is not world-readable
            icacls "%SSH_KEY%" /inheritance:r

            REM Give Read permission to the Jenkins process user (USERNAME) and SYSTEM
            icacls "%SSH_KEY%" /grant:r "%USERNAME%:R" || echo Grant to %USERNAME% failed
            icacls "%SSH_KEY%" /grant:r "SYSTEM:F" || echo Grant to SYSTEM failed

            REM Remove BUILTIN\\Users entry if present (ignore errors)
            icacls "%SSH_KEY%" /remove "BUILTIN\\Users" || echo Remove BUILTIN\\Users failed

            REM Make file read-only as an extra protection
            attrib +R "%SSH_KEY%" || echo attrib failed

            echo Now running scp to copy workspace to EC2 webroot
            scp -i "%SSH_KEY%" -o StrictHostKeyChecking=no -r "%WORKSPACE%\\*" ubuntu@<EC2_IP>:/var/www/html/

            echo === Deploy finished ===
          """
        }
      }
    }
  }

  post {
    success {
      echo 'Deployment Successful!'
    }
    failure {
      echo 'Deployment Failed — check console output for details.'
    }
  }
}
```
````

This Jenkinsfile:

* Checks out code from GitHub
* Secures SSH key permissions (important on Windows)
* Copies files via `scp` to `/var/www/html`
* Instantly updates website served by Nginx

---

## 5. Deployment Flow

### ✔ Step 1 — Push Updated Code to GitHub

```
git add .
git commit -m "Your update message"
git push
```

### ✔ Step 2 — Run Jenkins Build

* Jenkins pulls the latest code
* Jenkins deploys via SCP to EC2
* Nginx instantly serves new version

### ✔ Website instantly updates

Visit:

```
http://<EC2_PUBLIC_IP>
```

Press **Ctrl+F5** to bypass browser cache.

---

## 6. Optional: Enable Auto-Deploy (Recommended)

You can avoid clicking "Build Now" by setting up a GitHub webhook.

### Steps:

1. Jenkins → Configure Job → Check **GitHub hook trigger for GITScm polling**
2. GitHub → Repo Settings → Webhooks → Add webhook:

```
http://<JENKINS_IP>:8080/github-webhook/
```

3. Choose event: **Just the push event**

Now:

```
Any push to GitHub → Jenkins auto-builds → EC2 auto-deploys
```

---

## 7. Summary

You have built a complete CI/CD pipeline:

### ✔ Git → GitHub → Jenkins → EC2 → Nginx

### ✔ Automated deployments

### ✔ Passwordless SSH from Jenkins → EC2

### ✔ Full pipeline using Jenkinsfile

### ✔ Website updates instantly after deployment

This is a production-style pipeline suitable for static websites, HTML apps, frontend projects, and more.

---


