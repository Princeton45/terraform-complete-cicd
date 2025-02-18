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
```groovy
   stages {
        stage("build app") {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }
```
   ![artifact](https://github.com/Princeton45/terraform-complete-cicd/blob/main/images/artifact.png)

2. **CI: Docker Image**
   - Created Docker images from the Java artifact
   - Pushed images to my Docker Hub repository

   ![docker-hub](https://github.com/Princeton45/terraform-complete-cicd/blob/main/images/docker-hub.png)

   ![build-image](https://github.com/Princeton45/terraform-complete-cicd/blob/main/images/build-image.png)

   ![docker-push](https://github.com/Princeton45/terraform-complete-cicd/blob/main/images/docker-push.png)


3. **CD: Infrastructure Provisioning**
   - Used Terraform to automatically provision EC2 instances
   - Added Terraform configurations to the Git repository

`main.tf`
```hcl
provider "aws" {
  region = var.region
}

resource "aws_vpc" "myapp-vpc" {
  cidr_block = var.vpc_cidr_block
  tags = {
    Name: "${var.env_prefix}-vpc"
  }
}

resource "aws_subnet" "myapp-subnet-1" {
  vpc_id = aws_vpc.myapp-vpc.id
  cidr_block = var.subnet_cidr_block
  availability_zone = var.avail_zone
  tags = {
    Name = "${var.env_prefix}-subnet-1"
  }
}

resource "aws_internet_gateway" "myapp-igw" {
  vpc_id = aws_vpc.myapp-vpc.id
  tags = {
    Name: "${var.env_prefix}-igw"
  }
}

resource "aws_route_table" "myapp-route-table" {
  vpc_id = aws_vpc.myapp-vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.myapp-igw.id
  }
  tags = {
    Name: "${var.env_prefix}-rtb"
  }
}

resource "aws_route_table_association" "aws-rtb-subnet" {
  subnet_id = aws_subnet.myapp-subnet-1.id
  route_table_id = aws_route_table.myapp-route-table.id
}

resource "aws_security_group" "myapp-sg" {
  name = "myapp-sg"
  vpc_id = aws_vpc.myapp-vpc.id

  ingress {
    from_port = 22
    to_port = 22
    protocol = "TCP"
    cidr_blocks = [var.my_ip, var.jenkins_ip]
  }

  ingress {
    from_port = 8080
    to_port = 8080
    protocol = "TCP"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    prefix_list_ids = []
  }

  tags = {
    Name: "${var.env_prefix}-sg"
  }
}

data "aws_ami" "latest-amazon-linux-image" {
  most_recent = true
  owners = ["amazon"]
   filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

resource "aws_instance" "myapp-server" {
  ami = data.aws_ami.latest-amazon-linux-image.id
  instance_type = var.instance_type

  subnet_id = aws_subnet.myapp-subnet-1.id
  vpc_security_group_ids = [aws_security_group.myapp-sg.id]
  availability_zone = var.avail_zone

  associate_public_ip_address = true
  key_name = "myapp-key-pair"

  user_data = file("entry-script.sh")
  user_data_replace_on_change = true
  tags = {
    Name: "${var.env_prefix}-server"
  }

}

output "ec2_public_ip" {
    value = aws_instance.myapp-server.public_ip
}
```

`variables.tf`
```hcl
variable vpc_cidr_block {
    default = "10.0.0.0/16"
}
variable subnet_cidr_block {
    default = "10.0.10.0/24"
}
variable avail_zone {
    default = "us-east-1a"
}
variable env_prefix {
    default = "dev"
}
variable my_ip {
    default = "73.180.207.54/32"
}
variable instance_type {
    default = "t2.micro"
}
variable region {
    default = "us-east-1"
}

variable "jenkins_ip" {
    default = "159.203.188.237/32"
}
```

   ![ec2-instance](https://github.com/Princeton45/terraform-complete-cicd/blob/main/images/ec2-instance.png)


4. **CD: Deployment**
   - Deployed the application using Docker Compose
   - Set up automated deployment to the provisioned EC2 instance

   ![docker-compose](https://github.com/Princeton45/terraform-complete-cicd/blob/main/images/docker-compose.png)

  ![deployed](https://github.com/Princeton45/terraform-complete-cicd/blob/main/images/deployed.png)





