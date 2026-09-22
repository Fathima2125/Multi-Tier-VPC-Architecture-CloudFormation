# Technical Documentation

## MULTI-TIER AWS VPC ARCHITECTURE
=========================================

## 1. Document Information


| Item                         | Details                                         |
|----------------------------- | ------------------------------------------------|        
|   Project Name               |    Multi-Tier AWS VPC Architecture              | 
|                              |                                                 |
|   Program                    |    AWS She Builds Mentorship Program 2026       |
|                              |                                                 |
|   Author                     |     Fathima Yosra Ajeeb                         |
|                              |                                                 |
|    AWS Region                |     us-east-1(Northern - Virginia)              |
|                              |                                                 |
|  Architecture Type           |     Three-Tier, Multi-AZ VPC                    |
|                              |                                                 |
| Infrastructure Approach      |      Manual AWS Console + Infrastructure as Code|
|                              |                                                 |
|  IaC Tool                    |      AWS CloudFormation                         |
|                              |                                                 |
|    Date                      |       September 22, 2026                        |



----------------------------------------------
## 2. Purpose

The purpose of this project is to design and deploy a secure, highly available, and logically isolated **three-tier AWS VPC architecture**.

The architecture separates resources into **Public, Application, and Database tiers** across two Availability Zones. Network traffic is controlled using **Route Tables, Security Groups, and Network ACLs**, while an Application Load Balancer distributes incoming traffic across application servers.

The architecture was first implemented manually through the AWS Management Console and then reproduced using **AWS CloudFormation** to demonstrate Infrastructure as Code.

----------------------------------------------

## 3. Architecture Objectives

The main objectives of the architecture are:

* Provide network isolation between different application layers.
* Separate public-facing and private resources.
* Improve availability by using two Availability Zones.
* Control network traffic using Security Groups and Network ACLs.
* Provide controlled internet access through an Internet Gateway and NAT Gateways.
* Distribute incoming application traffic using an Application Load Balancer.
* Allow secure management of EC2 instances through AWS Systems Manager.
* Create a reproducible infrastructure using CloudFormation.
* Follow security principles such as least privilege and restricted network access.

-------------------------------------------

## 4. Architecture Overview

The VPC follows a **three-tier architecture**:

1. **Public Tier** – Contains the Application Load Balancer and public-facing networking components.
2. **Application Tier** – Contains private EC2 instances running the application/web layer.
3. **Database Tier** – Contains private subnets reserved for database resources.

The architecture spans **two Availability Zones** to provide redundancy.

### High-Level Traffic Flow

```text
Internet
   |
Internet Gateway
   |
Application Load Balancer
   |
Target Group
   |
Private Application EC2 Instances
   |
Database Tier
```

The Application and Database tiers are not directly accessible from the public internet.

-------------------------------------------

## 5. Architecture Diagram

![Diagram](<Multi-Tier VPC -architecture.drawio.png>)

-------------------------------------------

## 6. VPC Specification

| Component                     | Specification                   |
| ----------------------------- | --------------------------------|
| **VPC Name**                  | `cfn-devops-vpc`                |
| **VPC CIDR**                  | `10.0.0.0/16`                   |
| **Availability Zones**        | Two AZs                         |
| **Public Subnets**            | `10.0.101.0/24`, `10.0.102.0/24`|
| **Private App Subnets**       | `10.0.111.0/24`, `10.0.112.0/24`|
| **Private DB Subnets**        | `10.0.121.0/24`, `10.0.122.0/24`|
| **Internet Gateway**          | 1                               |
| **NAT Gateways**              | 2                               |
| **Application Load Balancer** | 1                               |
| **EC2 Instances**             | 2                               |
| **Target Group**              | 1                               |

The VPC uses a `/16` CIDR block, which provides sufficient address space for the current architecture and future expansion.

--------------------------------------

## 7. Subnet Design

The VPC contains six subnets distributed across two Availability Zones.

| Tier        | AZ-A            | AZ-B               | Purpose               |
| ----------- | --------------  | ------------------ | --------------------- |
| Public      | `10.0.101.0/24` | `10.0.102.0/24`    | ALB and NAT Gateways  |
| Application | `10.0.111.0/24` | `10.0.112.0/24`    | Private EC2 instances |
| Database    | `10.0.121.0/24` | `10.0.122.0/24`    | Database resources    |

### Public Tier

The public subnets have routes to the Internet Gateway. They are designed to host internet-facing networking components such as the ALB and NAT Gateways.

### Application Tier

The application subnets are private and contain the EC2 instances. They do not have direct routes to the Internet Gateway.

### Database Tier

The database subnets are private and isolated from direct internet access. They are intended for database resources and only allow required traffic from the application layer.

---------------------------------------

## 8. Routing & Internet Connectivity

### 8.1 Internet Gateway

The Internet Gateway provides connectivity between the VPC and the public internet.

The public route tables contain:

```text
0.0.0.0/0 → Internet Gateway
```

This allows resources in the public subnets to communicate with the internet where permitted.

### 8.2 NAT Gateways

Two NAT Gateways are deployed, one in each public subnet.

```text
Private App Subnet A → NAT Gateway A → Internet Gateway
Private App Subnet B → NAT Gateway B → Internet Gateway
```

NAT allows private instances to initiate outbound internet connections without allowing unsolicited inbound internet connections to those instances.

### 8.3 Route Tables

Separate route tables are used for the different tiers.

* Public Route Table
* Private Application Route Table A
* Private Application Route Table B
* Private Database Route Table A
* Private Database Route Table B

This provides granular control over where traffic can be routed.

-----------------------------------------

## 9. Security Architecture

### 9.1 Security Groups

Security Groups provide **stateful, instance-level traffic control**.

#### ALB Security Group (alb-sg)

* Allows HTTP traffic on port **80** from the internet.
* Allows required outbound traffic to the application layer.

```text
Inbound:
TCP 80 → 0.0.0.0/0
```

#### Application Security Group(app-sg)

The application EC2 instances only accept HTTP traffic from the ALB Security Group.

```text
Inbound:
TCP 80 → ALB Security Group
```

This prevents direct public access to the EC2 instances.

#### Database Security Group(db-sg)

The database layer allows PostgreSQL traffic only from the application security group.

```text
Inbound:
TCP 5432 → Application Security Group
```

-------------------------------------------

## 9.2 Network ACLs

Network ACLs provide **stateless subnet-level traffic control**.

Separate NACLs are configured for:

* Public subnets(public - Nacl)
* Application subnets(app - Nacl)
* Database subnets(db - Nacl)

### Public-Nacl

##### Inbound Rules
```text

Rule Number |   Type    |   Protocol    |   Port Range  |   Source  |   Allow/Deny  |
-------------------------------------------------------------------------------------
  100        HTTP(80)      TCP(6)          80               0.0.0.0/0         ALlow      
  110        HTTPS(443)    TCP(6)          443              0.0.0.0/0         Allow
  120        Custom TCP    TCP(6)          1024-65535       0.0.0.0/0         Allow
  130        Custom TCP    TCP(6)          1024-65535       10.0.111.0/24     Allow
  140        Custom TCP    TCP(6)          1024-65535       10.0.112.0/24     Allow
  *          All Traffic    All             All             0.0.0.0/0         Deny
```
#### Outbound Rules 
```text

Rule Number |   Type    |   Protocol    |   Port Range  |   Source  |   Allow/Deny  |
-------------------------------------------------------------------------------------
  100        HTTP(80)      TCP(6)          80                10.0.111.0/24     ALlow      
  110        HTTP(80)      TCP(6)          80                10.0.112.0/24     Allow
  120        HTTPS(443)    TCP(6)          443               0.0.0.0/0         Allow
  130        HTTP(80)      TCP(6)          80                0.0.0.0/0         Allow
  140        Custom TCP    TCP(6)          1024-65535        0.0.0.0/0         Allow
  *          All Traffic    All            All               0.0.0.0/0         Deny
```


### App-Nacl

##### Inbound Rules
```text

Rule Number |   Type    |   Protocol    |   Port Range  |   Source  |   Allow/Deny  |
-------------------------------------------------------------------------------------
  100        HTTP(80)      TCP(6)          80               10.0.101.0/24    ALlow      
  110        HTTP(80)      TCP(6)          80               10.0.102.0/24    Allow
  120        Custom TCP    TCP(6)          1024-65535       10.0.101.0/24    Allow
  130        Custom TCP    TCP(6)          1024-65535       10.0.102.0/24    Allow
  140        Custom TCP    TCP(6)          1024-65535       10.0.121.0/24    Allow
  150        Custom TCP    TCP(6)          1024-65535       10.0.122.0/24    Allow
  160        DNS(UDP)      UDP(17)         53               10.0.0.0/16      Allow
  170        DNS(TCP)      TCP(6)          53               10.0.0.0/16      Allow
  180        Custom TCP    TCP(6)          1024-65535       0.0.0.0/0        Allow  
  *          All Traffic    All             All             0.0.0.0/0        Deny
```

#### Outbound Rules 
```text

Rule Number |   Type    |   Protocol    |   Port Range  |       Source  |       Allow/Deny  |
---------------------------------------------------------------------------------------------
  100        Custom TCP         TCP(6)        1024-65535        10.0.101.0/24     ALlow      
  110        Custom TCP         TCP(6)        1024-65535        10.0.102.0/24     ALlow 
  120        PostgreSQL(5432)   TCP(6)        5432              10.0.121.0/24     Allow
  130        PostgreSQL(5432)   TCP(6)        5432              10.0.122.0/24     Allow
  140        HTTPS(443)         TCP(6)        443               0.0.0.0/0         Allow
  150        HTTP(80)           TCP(6)        80                0.0.0.0/0         Allow
  160        DNS(UDP)           UDP(17)       53                10.0.100.2/32     Allow
  170        DNS(TCP)           TCP(6)        53                10.0.100.2/32     Allow
  180        Custom UDP         UDP(17)       123               10.0.100.2/32     Allow
  *          All Traffic        All           All               0.0.0.0/0         Deny
```


### db-Nacl

##### Inbound Rules
```text

Rule Number |   Type    |   Protocol    |   Port Range  |       Source  |       Allow/Deny  |
---------------------------------------------------------------------------------------------
  100        PostgreSQL(5432)   TCP(6)          5432            10.0.111.0/24     Allow    
  110        PostgreSQL(5432)   TCP(6)          5432            10.0.112.0/24     Allow
  120        Custom TCP         TCP(6)          1024-65535      10.0.111.0/24     Allow
  130        Custom TCP         TCP(6)          1024-65535      10.0.112.0/24     Allow  
  *          All Traffic        All             All             0.0.0.0/0         Deny
```

#### Outbound Rules 
```text

Rule Number |   Type    |   Protocol    |      Port Range  |       Source  |       Allow/Deny  |
------------------------------------------------------------------------------------------------
  100       Custom TCP         TCP(6)          1024-65535      10.0.111.0/24        Allow
  110       Custom TCP         TCP(6)          1024-65535      10.0.112.0/24        Allow  
  *          All Traffic        All           All               0.0.0.0/0           Deny
```

The NACL rules restrict traffic based on source/destination CIDRs, protocols, and ports.

The database NACL, for example, permits PostgreSQL traffic from the application subnets and required return/ephemeral traffic.

----------------------------------------

## 10. Compute & Application Layer

Two EC2 instances are deployed in the private application subnets:

* EC2 Instance A → Application Subnet A
* EC2 Instance B → Application Subnet B

The instances run **Amazon Linux 2023** with Nginx used to serve test web pages.

#### User Data Provided In CloudFormation Template for EC2- A

```text
UserData:
        Fn::Base64: |
          #!/bin/bash

          dnf update -y
          dnf install -y nginx

          systemctl enable nginx
          systemctl start nginx

          cat > /usr/share/nginx/html/index.html <<'EOF'
          <!DOCTYPE html>
          <html>
          <head>
              <title>CloudFormation EC2-A</title>
          </head>
          <body>
              <h1>CloudFormation EC2-A</h1>
              <p>Nginx is running on Application Server A.</p>
          </body>
          </html>
          EOF

```

The instances do not require public IP addresses.

AWS Systems Manager is used for secure management through the IAM role:

`AmazonSSMManagedInstanceCore`

This allows administration without exposing SSH directly to the internet.

-----------------------------------------

## 11. Load Balancing

An **Application Load Balancer (ALB)** is deployed across the public subnets.

The ALB:

1. Receives HTTP requests from clients.
2. Uses a listener on port 80.
3. Forwards requests to the Target Group.
4. Distributes requests between the private EC2 instances.
5. Performs health checks on the targets.

The target group contains both application EC2 instances.

The targets were successfully validated as **healthy** during testing.

-----------------------------------------

## 12. Traffic Flow

#### The primary incoming traffic path is:

```text
Client
  ↓
Internet
  ↓
Internet Gateway
  ↓
Application Load Balancer
  ↓
Target Group
  ↓
Private EC2 Instance
```

The following components control or influence this traffic:

* **Route Tables** determine where packets are routed.
* **NACLs** control traffic at the subnet level.
* **Security Groups** control traffic at the resource level.
* **ALB** distributes traffic between application instances.

#### For outbound internet access from private application instances:

```text
Private EC2
   ↓
Private Route Table
   ↓
NAT Gateway
   ↓
Public Route Table
   ↓
Internet Gateway
   ↓
Internet
```

-------------------------------------

## 13. High Availability & Resilience

The architecture uses two Availability Zones to reduce dependence on a single AZ.

High-availability features include:

* Two public subnets.
* Two private application subnets.
* Two private database subnets.
* EC2 instances distributed across two AZs.
* ALB deployed across multiple AZs.
* NAT Gateway in each AZ.

If one application instance becomes unavailable, the ALB can continue sending traffic to a healthy target in the other Availability Zone.

-----------------------------

## 14. IAM & Access Management

IAM is used to control access to AWS resources.

The EC2 instances use an IAM role containing:

`AmazonSSMManagedInstanceCore`

This allows Systems Manager to establish management access without requiring publicly exposed SSH access.

The architecture follows the principle of **least privilege**, where resources should receive only the permissions and network access required for their function.

----------------------------

## 15. Deployment & Implementation

The architecture was implemented in two stages.

### Stage 1 – Manual Deployment

The VPC and its components were initially created through the AWS Management Console.

The following were configured manually:

* VPC
* Six subnets
* Internet Gateway
* NAT Gateways
* Route Tables
* Security Groups
* Network ACLs
* EC2 instances
* IAM role
* Target Group
* Application Load Balancer

### Stage 2 – Infrastructure as Code

The architecture was reproduced using **AWS CloudFormation**.

CloudFormation defines the infrastructure as code, making the environment:

* Reproducible
* Consistent
* Easier to modify
* Easier to deploy again
* Less dependent on manual configuration

#### Terminal Commands

1. ##### Validate Cloudformation Template

```text

aws-cloudformation validate-template \
    --template-body file://devops-network-cfn.yaml

```
2. ##### Deploy Cloudformation stack

``` text
aws cloudformation deploy \
    --template-file devops-network-cfn.yaml \
    --stack-name cfn-devops-network \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameter-overrides AmazonLinuxAMI=<ami name> \
    --region us-east-1

```
3. ##### Check Cloudformation Stack status

```text
aws cloudformation describe-stacks \
    --stack-name cfn-devops-network \
    --region us-east-1
    --query "Stacks[0].[STackStatus, StackStatusReason]" \
    --output table

```

-------------------------

## 16. Validation & Testing

The following tests were performed to validate the architecture:

### Network Validation

* Confirmed all six subnets were created.
* Verified subnet-to-route-table associations.
* Verified Internet Gateway attachment.
* Verified NAT Gateway configuration.
* Verified appropriate routes.

### Security Validation

* Verified Security Group rules.
* Verified NACL rules.
* Confirmed application instances were not directly publicly accessible.
* Confirmed database access was restricted to the application layer.

### Application Validation

* Confirmed EC2 instances were running.
* Confirmed instances were accessible through AWS Systems Manager.
* Confirmed ALB target health.
* Confirmed ALB successfully served responses from the private EC2 instances.

### Access Validation

The EC2 instances were successfully managed through **SSM/Fleet Manager**, avoiding the need for public SSH access.

### Validation for Cloudformation Version

1. #### Generated Cloudformation Outputs

``` text
aws cloudformation describe-stacks \
    --stack-name cfn-devops-network \
    --region us-east-1 \
    --query "Stacks[0].Outputs[*].[OutputKey,OutputValue]" \
    --output table

```
2. #### Check SSM Connectivity
Show whether EC2 instances aree registered and connected to System Manager.

```text
aws ssm describe-instance-information \
    --region us-east-1 \
    --query "InstanceInformationList[].[InstanceId,PingStatus,PlatformName,AgentVersion]" \
    --output table
```
3. #### Target Group Health

```text

aws elbv2 describe-target-health \
    --target-group-arn YOUR_TARGET_GROUP_ARN \
    --region us-east-1 \
    --query "TargetHealthDescriptions[].[TargetId,Target.Port,TargetHealth.State,TargetHealth.Reason,TargetHealth.Description]" \
    --output table
```
4. #### ALB Verfication

```text

curl  http://YOUR_ALB_DNS_NAME

```

------------------------------

## 17. Cost Considerations

The architecture includes AWS resources that generate ongoing costs, particularly:

* NAT Gateways
* Application Load Balancer
* EC2 instances
* Data transfer
* Other associated AWS services

Two NAT Gateways improve availability but increase cost compared with using a single NAT Gateway.

A billing alarm is configured to provide cost monitoring and help identify unexpected AWS spending.

The Cloudformation Stack is deleted After use and then recreated when needed.

For a production environment, costs should be reviewed against availability and security requirements.

----------------------------

## 18. Conclusion

This project demonstrates the design and implementation of a secure, isolated and highly available **three-tier AWS VPC architecture**.

The architecture separates public, application, and database network layers while using Security Groups, Network ACLs and Route Tables to control communication.

The use of two Availability Zones improves resilience, while the Application Load Balancer provides controlled access to private application servers.

The architecture was first deployed manually to understand the underlying AWS networking components and was subsequently reproduced using CloudFormation to demonstrate Infrastructure as Code and repeatable infrastructure deployment.

-----------------------------

## 19. Future Improvements

Potential future improvements include:

* Implement Auto Scaling for EC2 instances.
* Replace test EC2 web servers with a production application.
* Deploy Amazon RDS across the database subnets.
* Add AWS WAF in front of the ALB.
* Add CloudWatch monitoring and centralized logging.
* Implement automated CI/CD deployment.
* Add additional security controls and automated compliance checks.
* Extend the CloudFormation template to fully automate the environment.
* Evaluate Terraform as an alternative Infrastructure as Code solution.

--------------------------------