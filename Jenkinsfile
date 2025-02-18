pipeline {
    agent any
    
    parameters {
        choice(name: 'ACTION', choices: ['deploy', 'destroy'], description: 'Select action to perform')
        booleanParam(name: 'CONFIRM_DESTROY', defaultValue: false, description: 'Confirm infrastructure destruction')
    }
    
    tools {
        maven 'maven-3.9.9'
    }
    
    environment {
        IMAGE_NAME = 'prince450/demo-app:java-maven-2.0'
    }
    
    stages {
        // Only run build and deploy stages if ACTION is 'deploy'
        stage("build app") {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }
        
        stage('build image') {
            when {
                expression { params.ACTION == 'deploy' }
            }
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
            when {
                expression { params.ACTION == 'deploy' }
            }
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
                TF_VAR_env_prefix = 'test'
            }
            steps {
                script {
                    dir('terraform') {
                        sh "terraform init"
                        sh "terraform apply --auto-approve"
                        EC2_PUBLIC_IP = sh(
                            script: "terraform output ec2_public_ip",
                            returnStdout: true
                        ).trim()
                    }
                }
            }
        }

        stage("deploy") {
            when {
                expression { params.ACTION == 'deploy' }
            }
            environment {
                DOCKER_CREDS = credentials('docker-hub-repo')
            }
            steps {
                script {
                    echo "waiting for EC2 server to intialize "
                    sleep(time: 90, unit: "SECONDS")

                    echo 'deploying docker image to EC2...'
                    echo "${EC2_PUBLIC_IP}"

                    def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
                    def ec2Instance = "ec2-user@${EC2_PUBLIC_IP}"
                    
                    sshagent(['server-ssh-key']) {
                        sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
                        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
                        sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
                    }
                }
            }
        }

        stage("destroy infrastructure") {
            when {
                allOf {
                    expression { params.ACTION == 'destroy' }
                    expression { params.CONFIRM_DESTROY == true }
                }
            }
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
                TF_VAR_env_prefix = 'test'
            }
            steps {
                script {
                    // Additional confirmation step
                    input message: 'Are you absolutely sure you want to destroy the infrastructure?', ok: 'Yes, destroy it'
                    
                    dir('terraform') {
                        try {
                            sh "terraform init"
                            sh "terraform destroy --auto-approve"
                            echo "Infrastructure destroyed successfully"
                        } catch (Exception e) {
                            error "Failed to destroy infrastructure: ${e.getMessage()}"
                        }
                    }
                }
            }
            post {
                success {
                    echo "Cleanup completed successfully"
                }
                failure {
                    echo "Destruction failed. Please check the logs and manually verify the infrastructure state"
                }
            }
        }
    }
    
    post {
        always {
            cleanWs() // Clean workspace after pipeline execution
        }
    }
}