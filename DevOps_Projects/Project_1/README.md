## CI pipeline for a Java application/ Continuous integration of a Java application
### 1. EC2 Instance Setup (Admin Server)
* Instance name: admin-server 
* AMI: Ubuntu 
* Instance type: t2.medium 
* Storage: 20 GB 
* Hostname:
```commandline
root@ip-178-65-34-123:~# hostname admin-server
```
[Jenkins setup](https://www.jenkins.io/doc/book/installing/linux/)

### 2. Install Java & Jenkins
#### Script to install Java 17 and Jenkins
```commandline
sudo apt update
sudo apt install openjdk-17-jdk -y
java -version
```
#### Jenkins installation
```commandline
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```
```commandline
root@admin-server:~# vi script.sh
```

#### Access Jenkins
* Open browser → ```http://<Public-IP>:8080```
* Unlock Jenkins using:
```commandline
cat /var/lib/jenkins/secrets/initialAdminPassword
```
### 3. SonarQube Installation
#### Create SonarQube user
```commandline
root@admin-server:~# adduser sonarqube  
root@admin-server:~# apt install unzip -y
root@admin-server:~# su – sonarqube 
```
#### Download & start SonarQube
```commandline
sonarqube@admin-server:~ wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
sonarqube@admin-server:~$ unzip * 
sonarqube@admin-server:~$ ls 
sonarqube-9.4.0.54424.zip sonarqube-9.4.0.54424 
sonarqube@admin-server:~$ chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424 
sonarqube@admin-server:~$ chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.4.0.54424 
sonarqube@admin-server:~$ cd sonarqube-9.4.0.54424/bin/linux-x86-64/ 
sonarqube@admin-server:~/sonarqube-9.4.0.54424/bin/linux-x86-64$ ./sonar.sh start 
Starting SonarQube... 
Started SonarQube. 
```
* Port: 9000 (open in security group)
* Login: admin / admin → old password is admin, change password to ```1234```

_put image here_

### 4. Install Docker
```commandline
root@admin-server:~# vi docker.sh
```
```commandline
#!/bin/bash
echo << EOF
"=========================================================="
"||     Set up Docker's Apt repository ...............   ||"
"=========================================================="
EOF
#Set up Docker's Apt repository
# Add Docker's official GPG key:
sudo apt-get update -y
sudo apt-get install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -y

echo << EOF
"=========================================================="
"||   Docker's Apt repository is completed...........    ||"
"=========================================================="
EOF



echo << EOF
"=========================================================="
"||   Install the Docker packages....................    ||"
"=========================================================="
EOF

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

echo << EOF
"=========================================================="
"||   Install is completed ....................    ||"
"=========================================================="
EOF

dockerStatus=$(systemctl status docker | awk '/Active/ {print $3}' | tr -d "[()]")
dockerVersion=$(docker -v | awk '/version/ {print $3}' | tr -d ",")

echo "The Docker status is $dockerStatus"
echo "The Docker version is $dockerVersion"
```
```commandline
root@admin-server:~# sh docker.sh 
```
### 5. Trivy Installation (Image Scanning)
https://trivy.dev/docs/v0.61/getting-started/installation/#__tabbed_2_2

```commandline
wget https://github.com/aquasecurity/trivy/releases/download/v0.61.1/trivy_0.61.1_Linux-64bit.deb 
sudo dpkg -i trivy_0.61.1_Linux-64bit.deb 
```
### 6. Jenkins Tool Configuration
#### Install Plugins
* Eclipse Temurin Installer 
* Pipeline Stage View

#### Configure Tools
Manage Jenkins → Tools
* JDK: Java 17 (Adoptium)
* Maven: Name = ```maven```

### 7. SonarQube Integration with Jenkins
_Add 3 screenshots_
### Create SonarQube Token
* SonarQube → My Account → Security → Generate Token
### Add Jenkins Credential
* Type: Secret Text 
* ID: ```sonarqube``` 
* Value: Token

### 8. Docker Permission for Jenkins
```commandline
root@admin-server:~# usermod  -aG docker jenkins
root@admin-server:~# usermod  -aG docker ubuntu 
root@admin-server:~# cat /etc/group | tail 
Docker:x:988;jenkins,ubuntu
root@admin-server:~# systemctl restart docker 
root@admin-server:~# systemctl status docker 
```
### 9. Docker Hub Credentials
#### Create Docker Hub Access Token
* Docker Hub → Account Settings → Security 
* Token name: ```cli``` 
* Permissions: Read, Write, Delete

#### Add Jenkins Credential
* Type: Username & Password 
* ID: ```docker-hub-credentials```
* Username: ```swathi971``` 
* Password: Access Token

### 10. Dockerfile (Tomcat Deployment)
```commandline
FROM tomcat:latest
RUN cp -r /usr/local/tomcat/webapps.dist /usr/local/tomcat/webapps
COPY webapp/target/webapp.war /usr/local/tomcat/webapps
```

### 11. Jenkinsfile (CI Pipeline)
```commandline
pipeline {
    agent any

    tools {
        jdk 'java-17'
        maven 'maven'
    }

    stages {
        stage('Git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Swathi971/test-1.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Build') {
            steps {
                sh "mvn clean install"
            }
        }

       stage('codescan') {
    steps {
        withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
            sh '''
                mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                -Dsonar.login=$SONAR_AUTH_TOKEN \
                -Dsonar.host.url=http://44.202.152.245:9000/
            '''
            }
           }
       }


        stage('Build and tag') {
            steps {
                sh "docker build -t swathi971/webapp:1 ."
            }
        }

        stage('Docker image scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html swathi971/webapp:1"
            }
        }

        stage('Containersation') {
            steps {
                sh '''
                    docker stop c2 || true
                    docker rm c2 || true
                    docker run -it -d --name c2 -p 9003:8080 swathi971/webapp:1
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin"
                    }
                }
            }
        }

        stage('Pushing image to repository') {
            steps {
                sh 'docker push swathi971/webapp:1'
            }
        }
    }
}
```
### Final Expected Outputs
* Jenkins Pipeline: SUCCESS 
* SonarQube: Code quality report visible 
* Docker image pushed to Docker Hub 
* Application accessible at:
```commandline
http://<EC2-IP>:9003/webapp
```


















Jenkinsfile:
```commandline
pipeline {
    agent any

    tools {
        jdk 'java-11'
        maven 'maven'
    }

    stages {
        stage('Git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Swathi971/trail.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Build') {
            steps {
                sh "mvn clean install"
            }
        }

        stage('codescan') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
                    sh "mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}"
                }
            }
        }

        stage('Build and tag') {
            steps {
                sh "docker build -t Swathi971/webapp:1 ."
            }
        }

        stage('Docker image scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html Swathi971/webapp:1"
            }
        }

        stage('Containersation') {
            steps {
                sh '''
                    docker stop c1
                    docker rm c1
                    docker run -it -d --name c1 -p 9001:8080 Swathi971/webapp:1
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin"
                    }
                }
            }
        }

        stage('Pushing image to repository') {
            steps {
                sh 'docker push Swathi971/webapp:1'
            }
        }
    }
}
```

#### SonarQube installation:
```commandline
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
unzip *
chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424
chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.4.0.54424
cd sonarqube-9.4.0.54424/bin/linux-x86-64/
./sonar.sh start
```

#### Installying Trivy
```commandline
wget https://github.com/aquasecurity/trivy/releases/download/v0.61.1/trivy_0.61.1_Linux-64bit.deb 
sudo dpkg -i trivy_0.61.1_Linux-64bit.deb
```
errot in pushing image to repository:
```commandline
root@ip-10-0-19-162:/# docker push swathi971/webapp:1
The push refers to repository [docker.io/swathi971/webapp]
20043066d3d5: Waiting
05624b880a51: Waiting
378e3a6f165e: Waiting
841c6b2bc7ed: Waiting
45e2e3388eeb: Waiting
9be852926522: Waiting
ff4b6af59d8f: Waiting
627c55a201a9: Waiting
4f4fb700ef54: Waiting
901b8cfcfda7: Waiting
authentication required - access token has insufficient scopes
```
solution:
there was only read permission I allowed in dockerhub. 
You are using a Docker Hub token with READ-ONLY scope.
I need to allow acccess permistion as read and write in dockerhub while genereting the token.
### Step 1: Completely reset Docker login
On the Jenkins server:
```commandline
docker logout
```
Step 2: Create a NEW Docker Hub Access Token
On Docker Hub website:
* Login to Docker Hub
* Click profile → Account Settings 
* Go to Security 
* Click New Access Token 
* Name: jenkins-push 
* Permissions: Read, Write, Delete 
* Copy the token (VERY IMPORTANT)

⚠️ Do NOT use an old token

### Step 3: Login manually using the token
On Jenkins server terminal:
```commandline
docker login -u swathi971
```
When it asks for password:

👉 PASTE THE ACCESS TOKEN (not Docker password)

Expected output:
```commandline
Login Succeeded
```
### Wrong Docker Hub credentials in Jenkins
Even one character mismatch causes this.
* Go to Jenkins → Manage Jenkins → Credentials 
* Open docker-hub-credentials 
* Verify:
   * Username: swathi971 
   * Password:
  
     👉 Either your Docker Hub password
  
     👉 OR Access Token (recommended)

🔹 If unsure → delete and recreate the credential.

_I have updated the credentials by chnaging password to token._