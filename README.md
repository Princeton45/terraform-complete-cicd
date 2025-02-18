# Complete CI/CD Pipeline with Terraform

A complete CI/CD pipeline that automates the entire process from building Java applications to deploying on automatically provisioned AWS EC2 instances using Terraform.

   ![diagram](https://github.com/Princeton45/terraform-complete-cicd/blob/main/images/diagram.jpg)

## Technologies Used
- Terraform
- Jenkins
- Docker & Docker Hub
- AWS
- Git
- Java/Maven
- Linux

## Project Overview

I built an end-to-end CI/CD pipeline that includes infrastructure provisioning using Terraform. The pipeline automatically builds Java applications, creates Docker images, provisions EC2 instances, and deploys the application.

![CI/CD Steps](suggested-image: flowchart showing the 4 main pipeline steps)

## Pipeline Flow

### 1. Continuous Integration (CI)
#### Build Artifact
- Pulls source code from Git repository.
- Builds Java application using Maven.
- Runs unit tests.
- Generates JAR artifact.

#### Docker Image Creation
- Builds Docker image using the Dockerfile that contains the location of the generated artifact.
- Tags the Docker image.
- Pushes the image to Docker Hub.

### 2. Continuous Deployment (CD)
#### Infrastructure Provisioning
- Executes Terraform configurations.
- Creates SSH key pair for EC2 access.
- Provisions EC2 instance on AWS.
- Configures security groups and networking.
#### Application Deployment
- Connects to provisioned EC2 instance.
- Pulls Docker image from Docker Hub.
- Deploys application using Docker Compose.

## Infrastructure Setup

### Prerequisites
- Set up a Jenkins server with Docker installed
- Installed Terraform on Jenkins container - https://developer.hashicorp.com/terraform/install

   ![terraform-install](https://github.com/Princeton45/terraform-complete-cicd/blob/main/images/terraform-install.png)


### Security Configuration
- Generated an SSH key pair for the EC2 instance and added the .pem contents in Jenkins as a credential.
- Jenkins will be able to SSH into the EC2 instance to deploy the application using the .pem key.

   ![key-pair](https://github.com/Princeton45/terraform-complete-cicd/blob/main/images/key-pair.png)

   ![ssh-jenkins](https://github.com/Princeton45/terraform-complete-cicd/blob/main/images/ssh-jenkins.png)


## Pipeline Steps

1. **CI: Build Java Artifact**
   - Successfully built Java Maven application artifacts
   
2. **CI: Docker Image**
   - Created Docker images from the Java artifact
   - Pushed images to my Docker Hub repository

3. **CD: Infrastructure Provisioning**
   - Used Terraform to automatically provision EC2 instances
   - Added Terraform configurations to the Git repository

4. **CD: Deployment**
   - Deployed the application using Docker Compose
   - Set up automated deployment to the provisioned EC2 instance

![Deployment Architecture](suggested-image: diagram showing the deployed application architecture)






