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
