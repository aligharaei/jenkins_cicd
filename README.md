
# 🚀 Jenkins CI/CD Pipeline Setup with GitLab Integration

Welcome to the Jenkins CI/CD Pipeline project! This guide will walk you through setting up Jenkins using Docker Compose, connecting it to GitLab, and deploying changes automatically to a production server using SSH and Rsync.

## 🛠 Prerequisites

Before starting, make sure you have the following:

- **Docker & Docker Compose** installed on your server.
- **GitLab** instance with an accessible repository.
- **Production server** accessible via SSH.
  
## 🐳 1) Docker Compose for Jenkins Setup

Create a `docker-compose.yml` file to initiate Jenkins:

\`\`\`yaml
version: '3.9'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8085:8085"  # Jenkins UI port
      - "50085:50085"  # Jenkins agent port
    environment:
      JENKINS_OPTS: --httpPort=8085
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/local/bin/docker:/usr/bin/docker
      - ./init.groovy.d:/var/jenkins_home/init.groovy.d
    networks:
      - gitlab

networks:
  gitlab:

volumes:
  jenkins_home:
    driver: local
\`\`\`

💡 **Tip:** Use \`docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword\` to retrieve the admin password once Jenkins starts.

## 📦 2) Required Jenkins Plugins

Ensure the following plugins are installed:

- 🧩 **Git Plugin**
- 🧩 **GitLab Plugin**
- 🧩 **SSH Agent Plugin**
- 🧩 **Credentials Binding Plugin**
- 🧩 **Pipeline Plugin**

## 🔐 3) Generate GitLab Access Token

1. Go to your GitLab **profile**.
2. Navigate to **Access Tokens** and create one.
  
You'll use this in Jenkins to allow GitLab to communicate with Jenkins securely.

## 🔑 4) Jenkins Credentials Setup

1. Navigate to \`Manage Jenkins -> Credentials -> Global -> Add Credentials\`.
2. Add a new credential:
   - **Type**: GitLab username and password.
   - **Username**: \`\${gitlab_user}\`
   - **Password**: \`\${gitlab_password}\`

## 🗝 5) Set Up SSH for Deployment

Run the following commands to generate and copy SSH keys to your production server:

\`\`\`bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"  # Generate SSH keys
ssh-copy-id -i ~/.ssh/id_rsa.pub your_ssh_user@production_server  # Copy public key
ssh your_ssh_user@production_server  # Test SSH connection
ls -ld \${DEVELOPMENT_PROJECT_PATH}  # Verify access to project path
\`\`\`

## 🛠 6) Create a Jenkins Pipeline

1. In Jenkins, create a new item (**Pipeline**).
2. Set the following options:
   - **Definition**: Pipeline script from SCM.
   - **SCM**: Git.
   - **Repository**: \`\${git_repository_url}\`.
   - **Credentials**: Use the credential created in step 4.
   - **Branch**: \`develop\`.
   - **Build trigger**: Enable "Build when a change is pushed to GitLab."

Save your settings.

## 🔔 7) Configure GitLab Webhook

1. In GitLab, navigate to your project settings.
2. Add a webhook to trigger on push events.
   - Webhook URL: \`http://your_jenkins_server/project/your_project\`
   - **Secret token**: Add the Jenkins token created in \`Profile -> Configure -> API/Token\`.

## 📝 8) Create the Jenkins Pipeline Script

Create a \`Jenkinsfile\` in the root of your project repository with the following content:

\`\`\`groovy
pipeline {
    agent any
    stages {
        stage('Clone repository') {
            steps {
                git branch: 'develop', url: '\${GIT_URL}', credentialsId: '\${CREDENTIAL_NAME}'
            }
        }
        stage('Deploy to Server via Rsync') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ssh-credentials', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
                    sh '''
                    rsync -avz --no-perms --no-owner --no-group --no-times -e "ssh -i \${SSH_KEY}" ./ \${SSH_USER}@\${PRODUCTION_SSH_IP}:\${PRODUCTION_PROJECT_PATH}
                    '''
                    sh '''
                    ssh -i \${SSH_KEY} \${SSH_USER}@\${PRODUCTION_SSH_IP} 'cd \${DEVELOPMENT_PROJECT_PATH} && composer install --no-interaction --prefer-dist --optimize-autoloader'
                    '''
                    sh '''
                    ssh -i \${SSH_KEY} \${SSH_USER}@\${PRODUCTION_SSH_IP} 'cd \${DEVELOPMENT_PROJECT_PATH} && php artisan cache:clear'
                    '''
                }
            }
        }
    }
}
\`\`\`

## ✅ 9) Finish

Now your Jenkins pipeline is fully configured! Each push to the \`develop\` branch will trigger the pipeline to run, deploying your app to the production server. 🎉

---

Feel free to adapt this README based on your project specifics.
