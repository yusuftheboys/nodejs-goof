pipeline {
    agent none
    environment {
        DOCKERHUB_CREDENTIALS = credentials('DockerLogin')
    }
    stages {
        stage('Secret Scanning Using Trufflehog') {
            agent {
                docker {
                    image 'trufflesecurity/trufflehog:latest'
                    args '-u root --entrypoint='
                }
            }
            steps {
                sh 'trufflehog filesystem . --exclude-paths trufflehog-excluded-paths.txt > trufflehog-scan-result.txt'
                sh 'cat trufflehog-scan-result.txt'
                archiveArtifacts artifacts: 'trufflehog-scan-result.txt'
            }
        }
        stage('Build') {
            agent {
              docker {
                  image 'node:lts-buster-slim'
              }
            }
            steps {
                sh 'npm install'
            }
        }
        stage('Test') {
            agent {
              docker {
                  image 'node:lts-buster-slim'
              }
            }
            steps {
                sh 'echo test'
            }
        }
        stage('SCA Retire Js') {
            agent {
              docker {
                  image 'node:lts-buster-slim'
              }
            }
            steps {
                sh 'npm install -g retire'
                sh 'retire > retire-scan-report.txt'
                sh 'cat retire-scan-report.txt' 
                archiveArtifacts artifacts: 'retire-scan-report.txt'
            }
        }
        stage('OWASP Dependency Check') {
            agent {
              docker {
                  image 'owasp/dependency-check:latest'
                  args '-u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint='
              }
            }
            steps {
                sh '/usr/share/dependency-check/bin/dependency-check.sh --scan . --project "nodejs-goof" --format ALL'
                archiveArtifacts artifacts: 'dependency-check-report.html'
                archiveArtifacts artifacts: 'dependency-check-report.json'
                archiveArtifacts artifacts: 'dependency-check-report.xml'
            }
        }
        stage('Build Docker Image and Push to Docker Registry') {
            agent {
                docker {
                    image 'docker:dind'
                    args '--user root --network host -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'docker build -t xenjutsu/nodejsgoof:0.1 .'
                sh 'docker push xenjutsu/nodejsgoof:0.1'
            }
        }
        stage('Deploy Docker Image') {
            agent {
                docker {
                    image 'kroniak/ssh-client'
                    args '--user root --network host'
                }
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: "DeploymentSSHKey", keyFileVariable: 'keyfile')]) {
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no jenkins@192.168.0.104 "echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin"'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no jenkins@192.168.0.104 docker pull xenjutsu/nodejsgoof:0.1'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no jenkins@192.168.0.104 docker rm --force nodejsgoof'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no jenkins@192.168.0.104 docker run -it --detach -p 4000:4000 --name nodejsgoof --network host xenjutsu/nodejsgoof:0.1'
                }
            }
        }
    }
}
