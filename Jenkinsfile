pipeline {
    agent any
    
    tools {
        maven 'maven-3.9.9'
    }
    
    environment {
        IMAGE_NAME = 'prince450/demo-app:java-maven-2.0'
    }
    
    stages {
        stage("build app") {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }
        
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build -t ${IMAGE_NAME} ."
                        sh 'echo $PASS | docker login -u $USER --password-stdin'
                        sh "docker push ${IMAGE_NAME}"
                    }
                }
            }
        }
        stage("provision server") {

        }
        stage("deploy") {
            steps {
                script {
                    echo 'deploying docker image to EC2...'
                    
                    def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME}"
                    def ec2Instance = "ec2-user@35.180.151.121"  
                    
                    sshagent(['server-ssh-key']) {
                        sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"  // Fixed scp command
                        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"  // Fixed scp command
                        sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
                    }
                }
            }
        }
    }
}