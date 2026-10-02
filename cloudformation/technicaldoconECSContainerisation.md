# Technical Documentation: Containerizing the EC2 Application Using Amazon ECS

## 1. Document Information

| Item                    | Details                                                      |
| ----------------------- | ------------------------------------------------------------ |
| Project                 | AWS Multi-Tier VPC Architecture                              |
| Purpose                 | Migrate the containerized application from EC2 to Amazon ECS |
| Container Runtime       | Docker                                                       |
| Container Orchestration | Amazon ECS                                                   |
| Compute                 | AWS Fargate                                                  |
| Image Registry          | Amazon ECR                                                   |
| Infrastructure          | AWS CloudFormation                                           |
| Configuration           | AWS Systems Manager Parameter Store                          |
| Application Tier        | Private Subnets                                              |
| Load Balancer           | Application Load Balancer (ALB)                              |
| Availability            | Two Availability Zones                                       |

---

## 2. Purpose

This document describes the strategy for migrating the containerized Nginx application currently running on private EC2 instances to **Amazon Elastic Container Service (ECS)** using **AWS Fargate**.

The previous implementation runs the Docker container directly on EC2. The ECS implementation removes the need to manage Docker on the EC2 instances and allows ECS/Fargate to manage the container workload.

The existing architecture already uses Docker and Amazon ECR as shown in the current containerization implementation.

The migration will therefore reuse the existing:

* Docker image
* Amazon ECR repository
* VPC
* Private application subnets
* Application Load Balancer
* Security model
* CloudFormation infrastructure approach

---

## 3. Current State

The current containerized architecture is:

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
Docker
    ↓
Nginx Container
```

The current implementation installs Docker on the EC2 instances through CloudFormation UserData, retrieves configuration from SSM Parameter Store, pulls the image from ECR, and runs the container.

Therefore, EC2 is currently responsible for both:

1. Providing the compute host
2. Running and managing the Docker container

---

## 4. Target State

The target architecture replaces the Docker-on-EC2 deployment with ECS Fargate.

```text
                         Internet
                            |
                            ↓
                    Internet Gateway
                            |
                            ↓
                Application Load Balancer
                     Public Subnets
                            |
                            ↓
                    Target Group
                            |
             ┌──────────────┴──────────────┐
             ↓                             ↓
       ECS Task - AZ A               ECS Task - AZ B
       Private App Subnet            Private App Subnet
             |                             |
             └──────────────┬──────────────┘
                            ↓
                     Nginx Container
                            |
                            ↓
                       Container :80
```

The key change is:

```text
CURRENT

ALB
 ↓
EC2
 ↓
Docker
 ↓
Container


TARGET

ALB
 ↓
ECS Service
 ↓
Fargate Tasks
 ↓
Container
```

---

## 5. Why ECS Fargate?

ECS provides container orchestration, while Fargate provides the compute capacity for the containers without requiring us to manage EC2 hosts.

With the current EC2 implementation, we must manage:

* EC2 instances
* Docker installation
* Docker service
* Docker container lifecycle
* Container restart configuration
* EC2 UserData

With ECS Fargate, ECS manages the container workload.

Therefore:

```text
EC2 + Docker Management
        ↓
ECS + Fargate
        ↓
Managed Container Workload
```

This also prepares the application for future improvements such as ECS service scaling, rolling deployments, and centralized container logging.

---

## 6. Components of the ECS Architecture

The target solution contains the following components:

| Component           | Purpose                                         |
| ------------------- | ----------------------------------------------- |
| Amazon ECR          | Stores the Docker image                         |
| ECS Cluster         | Logical grouping for ECS workloads              |
| ECS Task Definition | Defines how the container runs                  |
| ECS Service         | Maintains the desired number of tasks           |
| Fargate             | Provides serverless compute for ECS tasks       |
| ALB                 | Receives external traffic                       |
| Target Group        | Routes ALB traffic to ECS tasks                 |
| Task Security Group | Controls traffic to the ECS containers          |
| IAM Execution Role  | Allows ECS/Fargate to pull images and send logs |
| CloudWatch Logs     | Stores container logs                           |
| Private Subnets     | Host ECS tasks without public IPs               |
| NAT Gateway         | Provides outbound connectivity where required   |

---

## 7. Amazon ECR

The existing ECR repository remains the container image source.

Example:

```text
multi-tier-vpc-app
```

The image flow remains:

```text
Dockerfile
    ↓
Docker Build
    ↓
Docker Image
    ↓
Amazon ECR
    ↓
ECS Fargate
```

The existing implementation already pushes the Docker image to ECR.

No new image registry is required for the ECS migration.

---

## 8. ECS Cluster

Create an ECS cluster for the application.

Example:

```text
Cluster Name:
multi-tier-vpc-cluster
```

The ECS cluster provides the logical environment in which the application service runs.

The cluster itself does not require EC2 instances when using Fargate.

---

## 9. ECS Task Definition

The ECS Task Definition defines how the Nginx container should run.

It should specify:

* Fargate launch type
* CPU
* Memory
* Container name
* ECR image
* Container port 80
* CloudWatch logging
* IAM execution role

Example conceptual configuration:

```text
Task Definition
│
├── Launch Type: FARGATE
├── Network Mode: awsvpc
├── CPU: 256
├── Memory: 512
│
└── Container
    ├── Name: web-app
    ├── Image: ECR image
    ├── Container Port: 80
    └── Log Driver: awslogs
```

The exact CPU and memory values can be adjusted according to application requirements.

---

## 10. ECS Task Execution Role

Fargate requires an IAM execution role so ECS can perform actions required to start the task.

The execution role should allow ECS to:

* Pull the container image from Amazon ECR
* Send container logs to CloudWatch Logs

The role should use the AWS-managed policy:

```text
AmazonECSTaskExecutionRolePolicy
```

This is different from the EC2 IAM role used in the previous implementation.

### Previous EC2 role

The EC2 instance required permissions for:

```text
ECR
SSM
Systems Manager
```

because the EC2 instance itself performed the Docker/ECR operations.

### ECS implementation

The Fargate task instead uses:

```text
ECS Task Execution Role
        ↓
ECR image pull
        +
CloudWatch Logs
```

The old EC2 Docker-specific IAM permissions are therefore no longer required for the ECS workload.

---

## 11. ECS Service

Create an ECS Service to manage the application tasks.

Example:

```text
ECS Service
Name: multi-tier-vpc-service
Desired Tasks: 2
Launch Type: Fargate
```

The service ensures that the desired number of tasks remain running.

For this two-AZ architecture:

```text
ECS Service
     |
     +---- Task 1 → AZ-A
     |
     +---- Task 2 → AZ-B
```

This provides application-level redundancy across the two Availability Zones.

---

## 12. ECS Networking

The ECS tasks will be deployed into the existing private Application subnets.

Current subnets:

```text
Private-App-A
10.0.11.0/24

Private-App-B
10.0.12.0/24
```

The target configuration is:

```text
ALB
 ↓
Public Subnet A/B

ECS Fargate Tasks
 ↓
Private App Subnet A/B
```

The ECS tasks should not receive public IP addresses.

This maintains the existing security model where the application tier remains private.

---

## 13. Security Groups

The existing ALB security model can be retained.

### ALB Security Group

```text
Inbound:
HTTP 80
Source: 0.0.0.0/0
```

### ECS Task Security Group

```text
Inbound:
TCP 80
Source: ALB Security Group
```

Traffic therefore follows:

```text
Internet
   ↓
ALB SG
   ↓
ALB
   ↓
ECS Task SG
   ↓
Nginx Container :80
```

The ECS task should not allow HTTP traffic directly from the internet.

---

## 14. Application Load Balancer Integration

The existing ALB can be reused.

The target group must be changed from registering EC2 instances to registering ECS tasks.

### Current

```text
ALB
 ↓
Target Group
 ↓
EC2 Instances
```

### Target

```text
ALB
 ↓
Target Group
 ↓
ECS Tasks
```

The ECS service will automatically register and deregister tasks with the target group.

Because Fargate uses `awsvpc` networking, the target group should use:

```text
Target Type: IP
```

The ALB continues to listen on port 80 and forwards traffic to:

```text
ECS Task Port: 80
```

---

## 15. Container Health Check

The ECS task should have a container health check.

Example:

```text
Command:
curl http://localhost:80
```

The health check confirms that Nginx is responding inside the container.

The ALB also performs its own target health check.

Therefore, two levels of health monitoring are available:

```text
ECS Container Health Check
          +
ALB Target Health Check
```

---

## 16. CloudWatch Logging

Container logs should be sent to Amazon CloudWatch Logs.

Example log group:

```text
/aws/ecs/multi-tier-vpc-app
```

The ECS task definition configures the AWS Logs driver.

Conceptually:

```text
Nginx Container
      ↓
awslogs
      ↓
CloudWatch Logs
```

This removes the need to connect to an EC2 instance to inspect container logs.

---

## 17. ECR Connectivity from Private Subnets

The Fargate tasks are placed in private subnets.

They need connectivity to AWS services required during task startup, including ECR.

The current architecture already provides outbound connectivity through NAT Gateway:

```text
ECS Task
   ↓
Private Route Table
   ↓
NAT Gateway
   ↓
Internet Gateway
   ↓
AWS Services
```

A future production improvement could use VPC endpoints for services such as ECR and CloudWatch Logs, reducing reliance on NAT Gateway connectivity.

---

## 18. CloudFormation Changes

The existing CloudFormation template currently provisions the VPC, networking, security groups, EC2 instances and related infrastructure.

For the ECS implementation, add the following resources.

```text
AWS::ECS::Cluster

AWS::ECS::TaskDefinition

AWS::ECS::Service

AWS::IAM::Role
(ECS Task Execution Role)

AWS::Logs::LogGroup

AWS::ElasticLoadBalancingV2::TargetGroup
(updated for ECS/IP targets)

AWS::EC2::SecurityGroup
(ECS Task Security Group)
```

The existing ALB can be reused with the required target group/listener updates.

---

## 19. What Happens to EC2 UserData?

This is one of the main changes from the current implementation.

### Current EC2 implementation

UserData performs:

```text
Install Docker
      ↓
Install AWS CLI
      ↓
Retrieve SSM parameters
      ↓
Authenticate with ECR
      ↓
Docker Pull
      ↓
Docker Run
```

This is documented in the current EC2 containerization implementation.

### ECS implementation

This entire Docker-related UserData process is removed.

Instead:

```text
CloudFormation
      ↓
ECS Task Definition
      ↓
ECS Service
      ↓
Fargate launches task
      ↓
Fargate pulls image from ECR
      ↓
Container starts
```

Therefore, **Docker should no longer be installed on the EC2 instances for the ECS architecture**.

In fact, the ECS/Fargate application tier no longer requires those application EC2 instances at all.

---

## 20. What Happens to the Existing EC2 Instances?

The existing EC2-based application deployment is replaced by ECS Fargate.

### Before

```text
Private EC2-A
   ↓
Docker
   ↓
Nginx Container

Private EC2-B
   ↓
Docker
   ↓
Nginx Container
```

### After

```text
ECS Fargate Task A
   ↓
Nginx Container

ECS Fargate Task B
   ↓
Nginx Container
```

The EC2 instances used only for hosting the Docker application are therefore no longer required for the ECS implementation.

The underlying VPC, ALB, subnets, route tables, NAT gateways and other shared infrastructure can remain.

---

## 21. SSM Parameter Store

The existing SSM parameters are:

```text
/multi-tier-vpc/aws-region
/multi-tier-vpc/ecr-repository
/multi-tier-vpc/image-tag
```

These were previously retrieved by EC2 UserData.

For ECS, the image URI and deployment configuration should preferably be defined through the ECS Task Definition/CloudFormation parameters.

SSM Parameter Store can still be used for application configuration if required.

For sensitive configuration such as:

```text
Database passwords
API keys
Secrets
```

use:

```text
AWS Secrets Manager
```

rather than plain Parameter Store values.

---

## 22. Deployment Process

The complete ECS deployment process is:

```text
1. Prepare Application
          ↓
2. Create Dockerfile
          ↓
3. Build Docker Image
          ↓
4. Test Image
          ↓
5. Push Image to ECR
          ↓
6. Create ECS Cluster
          ↓
7. Create Task Execution Role
          ↓
8. Create CloudWatch Log Group
          ↓
9. Create ECS Task Definition
          ↓
10. Create ECS Service
          ↓
11. Deploy Tasks into Private Subnets
          ↓
12. Register Tasks with ALB Target Group
          ↓
13. ALB Health Check
          ↓
14. Application Available
```

---

## 23. Traffic Flow

The final application traffic flow is:

```text
                       INTERNET
                           |
                           ↓
                    Internet Gateway
                           |
                           ↓
                  Application Load Balancer
                     Public Subnets
                           |
                           ↓
                     Target Group
                     Target Type: IP
                           |
              ┌────────────┴────────────┐
              ↓                         ↓
       Fargate Task A            Fargate Task B
       Private App-A             Private App-B
              ↓                         ↓
       Nginx Container            Nginx Container
              |                         |
              └────────────┬────────────┘
                           ↓
                         Port 80
```

---

## 24. Validation

After deployment, verify the following.

### ECS Cluster

Confirm that the ECS cluster is active.

### ECS Service

Verify:

```text
Desired count: 2
Running count: 2
Pending count: 0
```

### ECS Tasks

Confirm that both tasks are:

```text
RUNNING
```

and distributed across the Availability Zones.

### ECR

Confirm that the required image and tag are available.

Example:

```text
multi-tier-vpc-app:1.0
```

### Target Group

Verify both ECS tasks appear as:

```text
Healthy
```

### ALB

Access the ALB DNS name and verify that the Nginx application is reachable.

### CloudWatch

Verify that container logs are being delivered to the configured CloudWatch log group.

---

## 25. Failure and Recovery

One advantage of using an ECS Service is that ECS can maintain the desired number of tasks.

For example:

```text
Desired Count = 2

Task A fails
     ↓
ECS detects failure
     ↓
Task A replaced
     ↓
Service returns to 2 running tasks
```

The ALB only sends traffic to healthy registered targets.

This improves the application's operational resilience compared with manually managing Docker containers on EC2.

---

## 26. Deployment Updates

When a new application version is available:

```text
New Docker Image
       ↓
Push to ECR
       ↓
New Image Tag
       ↓
Update ECS Task Definition
       ↓
Update ECS Service
       ↓
ECS launches new tasks
       ↓
Health Checks
       ↓
Traffic moves to healthy tasks
```

This provides a foundation for controlled application deployments without manually connecting to EC2 instances.

---

## 27. Security Considerations

The ECS implementation maintains the existing security principles:

* ECS tasks remain in private subnets.
* Tasks do not receive public IP addresses.
* ALB is the public entry point.
* ECS task security group only accepts traffic from the ALB security group.
* ECR stores the container image.
* IAM controls ECS access to ECR and CloudWatch.
* Secrets should not be hardcoded into container images.
* CloudWatch provides centralized container logging.
* Container images should use trusted and regularly updated base images.

---

## 28. Current EC2 Containerization vs ECS

| Area                | Current EC2 Deployment | ECS Fargate Deployment             |
| ------------------- | ---------------------- | ---------------------------------- |
| Compute             | EC2                    | Fargate                            |
| Docker Host         | EC2                    | Managed by Fargate                 |
| Docker Installation | Required               | Not required                       |
| UserData            | Installs/runs Docker   | Not required for container startup |
| Image Registry      | ECR                    | ECR                                |
| Orchestration       | Manual/EC2             | ECS Service                        |
| Scaling             | EC2-based              | ECS Service                        |
| Container Health    | Docker/ALB             | ECS + ALB                          |
| Logs                | EC2/container          | CloudWatch                         |
| ALB                 | Used                   | Reused                             |
| Private Subnets     | Used                   | Used                               |
| IAM                 | EC2 role               | ECS Task Execution Role            |

---

## 29. Migration Strategy

The migration should be performed in stages rather than immediately deleting the existing EC2 implementation.

### Phase 1 – Existing Docker Implementation

```text
EC2
 ↓
Docker
 ↓
Nginx
 ↓
ECR
```

This validates that the application can successfully run as a container.

### Phase 2 – ECS Preparation

Create:

```text
ECS Cluster
Task Execution Role
Log Group
Task Definition
ECS Task Security Group
Target Group
```

### Phase 3 – ECS Deployment

Deploy the application using:

```text
ECS Service
+
Fargate
+
Private Subnets
```

### Phase 4 – ALB Validation

Verify:

```text
ALB
 ↓
ECS Target Group
 ↓
Healthy ECS Tasks
```

### Phase 5 – EC2 Removal

After successful validation:

```text
Stop/Remove old application EC2 deployment
```

The shared VPC infrastructure remains in place.

---

## 30. Rollback Strategy

If the ECS deployment fails validation, the previous EC2 deployment can temporarily remain available until ECS is confirmed to be working.

Rollback can therefore follow:

```text
ECS deployment fails
        ↓
Investigate ECS task / image / networking / IAM
        ↓
Restore traffic to previous EC2 deployment if required
        ↓
Correct ECS configuration
        ↓
Redeploy ECS
        ↓
Validate
        ↓
Complete migration
```

The EC2 implementation should only be removed after the ECS deployment has been successfully validated.

---

## 31. Future Improvements

After the ECS implementation is working, the architecture can be further improved with:

* ECS Service Auto Scaling
* HTTPS/TLS using ACM
* CI/CD with GitHub Actions
* Automated ECR image builds
* ECR image scanning
* AWS Secrets Manager
* CloudWatch alarms
* VPC endpoints for ECR and CloudWatch
* Blue/green or canary deployments
* Infrastructure deployment entirely through CloudFormation

---

## 32. Final Architecture

The final architecture is:

```text
                         INTERNET
                            |
                            ↓
                    Internet Gateway
                            |
                            ↓
                Application Load Balancer
                     Public Subnet A/B
                            |
                            ↓
                       Target Group
                       Target Type: IP
                            |
              ┌─────────────┴─────────────┐
              ↓                           ↓
       Fargate Task A              Fargate Task B
       Private App-A               Private App-B
              ↓                           ↓
       Nginx Container             Nginx Container
              |                           |
              └─────────────┬─────────────┘
                            ↓
                         Port 80


              CONTAINER IMAGE FLOW

 Dockerfile
     ↓
 Docker Build
     ↓
 Docker Image
     ↓
 Amazon ECR
     ↓
 ECS Task Definition
     ↓
 Fargate Tasks
```

---

## 33. Conclusion

The current implementation successfully demonstrates how the application can be containerized using Docker and run on EC2, with the image stored in Amazon ECR.

The next step is to remove the responsibility of managing Docker containers from the EC2 hosts and move the workload to **Amazon ECS using AWS Fargate**.

The resulting architecture is:

```text
Docker Image
     ↓
Amazon ECR
     ↓
ECS Task Definition
     ↓
ECS Service
     ↓
Fargate Tasks
     ↓
Private Subnets
     ↓
ALB
     ↓
Internet
```

This creates a clear progression in the project:

```text
On-Premises Application
        ↓
Rehost to AWS
        ↓
EC2
        ↓
Docker Containerization
        ↓
Amazon ECR
        ↓
Amazon ECS / Fargate
```

The EC2 Docker implementation demonstrates the containerization process, while the ECS implementation demonstrates **container orchestration and managed container deployment**.
