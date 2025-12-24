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