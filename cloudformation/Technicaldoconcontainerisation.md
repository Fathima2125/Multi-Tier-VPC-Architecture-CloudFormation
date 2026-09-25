# Technical Documentation: Containerizing the Application Inside the VPC

## 1. Document Information

| Item                | Details                                  |
| ------------------- | ---------------------------------------- |
| Project             | AWS Multi-Tier VPC Architecture          |
| Purpose             | Containerization of the Application Tier |
| Container Runtime   | Docker                                   |
| Container Image     | Nginx                                    |
| Image Registry      | Amazon Elastic Container Registry (ECR)  |
| Configuration Store | AWS Systems Manager Parameter Store      |
| Infrastructure      | AWS CloudFormation                       |
| Application Tier    | Private Subnets                          |
| Load Balancer       | Application Load Balancer (ALB)          |

---

## 2. Purpose

This document describes how the Nginx application running in the private Application tier is containerized using Docker and deployed inside the existing multi-tier VPC.

The process covers:

1. Preparing the application and Dockerfile
2. Building and testing the Docker image locally
3. Pushing the image to Amazon ECR
4. Deploying the container on private EC2 instances using CloudFormation UserData
5. Retrieving configuration from SSM Parameter Store
6. Configuring IAM permissions required for ECR and SSM access

> **Note:** The VPC itself is not containerized. The application workload running inside the Application tier is containerized.

---

## 3. Existing VPC Architecture

The existing architecture consists of three logical tiers:

```text
Internet
    |
Internet Gateway
    |
Application Load Balancer
    |
    +--------------------+
    |                    |
Public Subnet A      Public Subnet B
    |                    |
    +--------------------+
             |
       Private App Tier
       +-------------+
       |             |
    EC2-A          EC2-B
       |             |
   Docker          Docker
   Container       Container
       |             |
       +-------------+
             |
       Private DB Tier
```

The Application EC2 instances remain in private subnets.

The ALB receives external HTTP traffic and forwards requests to the EC2 instances on port 80.

---

## 4. Containerization Architecture

### 4.1 The containerization process aims to:

*   Package the application and dependencies    together.
*   Create a consistent runtime environment.
*   Reduce dependency on the underlying EC2 operating system.
*   Make application deployment portable.
*   Simplify application versioning.
*   Prepare the application for container orchestration.
*   Maintain the existing network isolation and security model.


### 4.2  The containerized deployment changes the application layer from:

```text
EC2
  |
Nginx installed directly on EC2
```

to:

```text
EC2
  |
Docker
  |
Nginx Container
```

The overall traffic flow becomes:

```text
Internet
   ↓
Internet Gateway
   ↓
Application Load Balancer
   ↓
Target Group
   ↓
Private EC2-A / EC2-B
   ↓
Docker Container
   ↓
Nginx :80
```

The EC2 host remains part of the VPC. The Docker container runs inside the EC2 host.

### 4.3 Architecture Diagram

![Diagram](<FinalMulti-VPC Diagram.drawio.png>)

-------

## 5. Phase 1 – Application Preparation

The application is prepared as a simple Nginx web application.

For testing the multi-AZ architecture, separate pages can be created:

```text
containerization/
├── Dockerfile
├── app-a/
│   └── index.html
└── app-b/
    └── index.html
```

Example:

```text
app-a/index.html
    → Application Server A
    → Availability Zone A

app-b/index.html
    → Application Server B
    → Availability Zone B
```

The separate pages make it easier to observe which EC2 instance responds through the ALB.

In a production application, both instances would normally run the same application image.

---

## 6. Phase 2 – Create Dockerfile, Build and Test

## 6.1 Dockerfile

Create a single Dockerfile:

```dockerfile
FROM nginx:alpine

ARG APP_SERVER

COPY ${APP_SERVER}/index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

The same Dockerfile is used for both application instances.

---

### 6.2 Build Image for EC2-A

```bash
docker build \
  --build-arg APP_SERVER=app-a \
  -t vpc-app-a:1.0 .
```

---

### 6.3 Build Image for EC2-B

```bash
docker build \
  --build-arg APP_SERVER=app-b \
  -t vpc-app-b:1.0 .
```

Verify the images:

```bash
docker images
```

---

### 6.4 Test EC2-A Image Locally

```bash
docker run -d \
  --name app-a \
  -p 8080:80 \
  vpc-app-a:1.0
```

Test:

```bash
curl http://localhost:8080
```

---

### 6.5 Test EC2-B Image Locally

```bash
docker run -d \
  --name app-b \
  -p 8081:80 \
  vpc-app-b:1.0
```

Test:

```bash
curl http://localhost:8081
```

Verify running containers:

```bash
docker ps
```

After testing:

```bash
docker stop app-a app-b
docker rm app-a app-b
```

The Docker images do not need to be deleted.

---

## 7. Phase 3 – Push Docker Image to Amazon ECR

Amazon ECR is used as the centralized registry for storing the Docker image.

### 7.1 Create ECR Repository

Example repository:

```text
multi-tier-vpc-app
```

The repository URI follows this format:

```text
<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/multi-tier-vpc-app
```

---

### 7.2 Authenticate Docker with ECR

```bash
aws ecr get-login-password \
  --region us-east-1 | \
docker login \
  --username AWS \
  --password-stdin \
  <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

---
### 7.3 Build the image

For A

```text
docker build --build-arg APP_SERVER=app-a -t multi-tier-vpc-app:1.0 .

```
For B
```text
docker build --build-arg APP_SERVER=app-b -t multi-tier-vpc-app:1.0 .
```


### 7.4 Tag the Image

Example:

```bash
docker tag \
  vpc-app-a:1.0 \
  <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/multi-tier-vpc-app:1.0
```

---

### 7.5 Push the Image

```bash
docker push \
  <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/multi-tier-vpc-app:1.0
```

Verify that the image with tag `1.0` appears in the ECR repository.

---

# 8. SSM Parameter Store Configuration

The following non-sensitive configuration values are stored in SSM Parameter Store:

```text
/multi-tier-vpc/aws-region
/multi-tier-vpc/ecr-repository
/multi-tier-vpc/image-tag
```

Example values:

```text
AWS_REGION=us-east-1

ECR_REPOSITORY=<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/multi-tier-vpc-app

IMAGE_TAG=1.0
```

These values are retrieved at instance startup rather than hardcoded directly into the UserData script.

---

# 9. EC2 IAM Permissions

The EC2 instance role requires the following permissions.

### AmazonEC2ContainerRegistryReadOnly

Allows the EC2 instance to:

* Authenticate with ECR
* Retrieve Docker images
* Pull images from ECR

### AmazonSSMManagedInstanceCore

Allows the EC2 instance to be managed through AWS Systems Manager, including SSM Session Manager.

### AmazonSSMReadOnlyAccess

Allows the EC2 instance to retrieve configuration values from Systems Manager Parameter Store.

Therefore, the EC2 role contains:

```text
AmazonEC2ContainerRegistryReadOnly
AmazonSSMManagedInstanceCore
AmazonSSMReadOnlyAccess
```

---

# 10. CloudFormation EC2 UserData

CloudFormation provisions the EC2 instances in the private Application subnets.

During instance startup, UserData performs the following operations:

```text
EC2 starts
   ↓
Install Docker + AWS CLI
   ↓
Retrieve configuration from SSM
   ↓
Authenticate with ECR
   ↓
Pull Docker image
   ↓
Run Docker container
   ↓
Nginx listens on port 80
```


The following UserData installs Docker, retrieves the configuration from SSM Parameter Store, authenticates with ECR, pulls the image, and starts the container.

```yaml
UserData:
  Fn::Base64: |
    #!/bin/bash

    # Update packages and install Docker + AWS CLI
    dnf update -y
    dnf install -y docker awscli

    # Start Docker
    systemctl enable docker
    systemctl start docker

    # Retrieve configuration from SSM Parameter Store
    AWS_REGION=$(aws ssm get-parameter \
      --name "/multi-tier-vpc/aws-region" \
      --query "Parameter.Value" \
      --output text)

    ECR_REPOSITORY=$(aws ssm get-parameter \
      --name "/multi-tier-vpc/ecr-repository" \
      --query "Parameter.Value" \
      --output text)

    IMAGE_TAG=$(aws ssm get-parameter \
      --name "/multi-tier-vpc/image-tag" \
      --query "Parameter.Value" \
      --output text)

    # Export environment variables
    export AWS_REGION
    export ECR_REPOSITORY
    export IMAGE_TAG

    # Authenticate with Amazon ECR
    aws ecr get-login-password \
      --region "$AWS_REGION" | \
      docker login \
      --username AWS \
      --password-stdin "$ECR_REPOSITORY"

    # Pull Docker image
    docker pull "$ECR_REPOSITORY:$IMAGE_TAG"

    # Run containerized Nginx
    docker run -d \
      --name web-app \
      --restart unless-stopped \
      -p 80:80 \
      "$ECR_REPOSITORY:$IMAGE_TAG"
```

The same UserData can be used for both EC2-A and EC2-B when both instances use the same application image.

---

# 11. Networking and Security

Containerization does not replace the existing VPC networking or security controls.

The existing architecture remains:

```text
ALB
 ↓
ALB Security Group
 ↓
Private EC2
 ↓
App Security Group
 ↓
Docker container
```

The Application Security Group continues to allow:

```text
TCP 80
Source: ALB Security Group
```

The container exposes port 80 and Docker maps:

```text
EC2 port 80 → Container port 80
```

Therefore, the ALB can continue using the existing target group configuration.

---

# 12. ECR Connectivity from Private Subnets

The EC2 instances do not have public IP addresses.

To pull the Docker image from ECR, the private EC2 instances require outbound connectivity.

The current architecture provides this through:

```text
Private EC2
     ↓
Private App Route Table
     ↓
NAT Gateway
     ↓
Internet Gateway
     ↓
AWS ECR
```

A more advanced production architecture can use VPC endpoints for AWS services such as ECR, reducing dependence on NAT Gateway connectivity.

---

# 13. Validation

After CloudFormation deploys the EC2 instances, verify:

### EC2

```bash
docker ps
```

Expected:

```text
 web-app 
```

### Docker Image

```bash
docker images
```

### Container Logs

```bash
 docker logs web-app
```

### Local Application Test

From the EC2 instance:

```bash
curl http://localhost
```

### ALB Test

Access the ALB DNS name and verify that the Nginx application is reachable through the load balancer.

---

# 14. Complete Deployment Flow

The complete containerization workflow is:

```text
                  DEVELOPMENT
                      │
                      ▼
                 Dockerfile
                      │
                      ▼
              Build Docker Image
                      │
                      ▼
                Local Testing
                      │
                      ▼
                Amazon ECR
                      │
                 Docker Push
                      │
                      ▼
                ECR Repository
                      │
                      │
              AWS PRIVATE NETWORK
                      │
                      ▼
              CloudFormation EC2
                      │
                      ▼
                  UserData
                      │
             ┌────────┴────────┐
             ▼                 ▼
       SSM Parameter         Docker
          Store             installed
             │                 │
             └────────┬────────┘
                      ▼
                ECR Authentication
                      │
                      ▼
                 Docker Pull
                      │
                      ▼
                 Docker Run
                      │
                      ▼
               Nginx Container
                      │
                    Port 80
                      │
                      ▼
                    ALB
                      │
                      ▼
                  Internet
```

---

# 15. Security Considerations

* EC2 instances remain in private subnets.
* Docker images are stored in Amazon ECR rather than directly on public infrastructure.
* ECR access is controlled through the EC2 IAM role.
* SSM Parameter Store is used for centralized configuration.
* Sensitive values such as database passwords should use `SecureString` or AWS Secrets Manager rather than plain text.
* The Application Security Group only accepts HTTP traffic from the ALB Security Group.
* Network ACLs continue to provide subnet-level traffic control.
* Docker containers should use trusted and regularly updated base images.

---

# 16. Future Improvements

The current implementation demonstrates Docker containerization on EC2. Future improvements could include:

* CI/CD pipeline for automatic image builds and ECR pushes
* Amazon ECS or Amazon EKS for container orchestration
* HTTPS/TLS termination at the ALB
* AWS Secrets Manager for application secrets
* ECR image scanning
* Container health checks
* CloudWatch container logging
* Auto Scaling Groups for EC2
* VPC endpoints for ECR and other AWS services

---

# 17. Conclusion

The application has been transitioned from a directly installed Nginx service to a containerized Nginx workload running on EC2 instances inside the private Application tier.

The resulting deployment provides a clear separation between:

```text
Application Code
      ↓
Docker Image
      ↓
Amazon ECR
      ↓
Private EC2
      ↓
Docker Container
      ↓
ALB
```

CloudFormation continues to manage the underlying infrastructure, while Docker manages the application runtime. SSM Parameter Store provides centralized configuration, and IAM controls access to ECR and Systems Manager services.
