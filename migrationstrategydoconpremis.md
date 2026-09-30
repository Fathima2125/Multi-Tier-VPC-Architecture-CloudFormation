# Migration Strategy: On-Premises Application to AWS EC2 Using the 6Rs.

## 1. Document Information

| Item                        | Details                                                                                          |
| --------------------------- | ------------------------------------------------------------------------------------------------ |
| Document Title              | Migration Strategy – On-Premises Application to AWS EC2 Using the 6Rs                            |
| Source Environment          | On-Premises                                                                                      |
| Target Environment          | AWS                                                                                              |
| Selected Migration Strategy | Rehost (Lift and Shift)                                                                          |
| Target Compute              | Amazon EC2                                                                                       |
| Target Network              | Secure Multi-Tier AWS VPC                                                                        |
| Availability                | Multi-AZ                                                                                         |
| Purpose                     | Define the strategy and technical approach for migrating the application from on-premises to AWS |

---

## 2. Executive Summary

This document defines a migration strategy for moving an existing application from an on-premises environment to AWS.

The application is initially treated as an existing workload running on an on-premises server. Before selecting a migration approach, the application is evaluated using the AWS **6R migration strategies**:

* Rehost
* Replatform
* Repurchase
* Refactor / Re-architect
* Retain
* Retire

Based on the current migration objective, **Rehost (Lift and Shift)** is selected as the initial strategy.

The application will be migrated to **Amazon EC2** with minimal changes to its application architecture. The EC2 instances will be deployed within the private application tier of the AWS VPC, while an Application Load Balancer in the public tier will provide controlled access to the application.

The purpose of this approach is to separate **migration from modernization**. The first objective is to establish the application successfully in AWS. Once the workload is stable, further modernization can be considered, including containerization and migration to Amazon ECS.

---

## 3. Migration Context

### 3.1 Current State

The application is assumed to currently run in an on-premises environment.

The simplified current architecture is:

```text
                    USERS
                      |
                      v
               On-Premises Network
                      |
                      v
              Application Server
                      |
                      v
                 Application
```

The application may have dependencies such as:

* Operating system
* Application runtime
* Libraries and packages
* Database
* Storage
* Network connectivity
* External APIs
* Authentication
* Configuration
* Certificates
* Scheduled processes

These dependencies must be identified before migration.

---

## 4. Migration Objectives

The migration has the following objectives:

1. Move the existing application from on-premises infrastructure to AWS.
2. Minimize changes to the application during the initial migration.
3. Reduce migration complexity and risk.
4. Establish a secure AWS network environment.
5. Deploy the application on Amazon EC2.
6. Keep application servers in private subnets.
7. Provide controlled inbound access through an Application Load Balancer.
8. Apply appropriate IAM and network security controls.
9. Validate application functionality after migration.
10. Maintain a rollback option during the migration period.
11. Establish a foundation for future application modernization.

---

## 5. Scope

### 5.1 In Scope

This migration strategy covers:

* Assessment of the application using the 6Rs.
* Selection of the initial migration strategy.
* AWS target architecture.
* VPC and subnet placement.
* EC2 deployment.
* Application migration.
* IAM.
* Security Groups.
* Network ACLs.
* Routing.
* Application Load Balancer.
* Testing and validation.
* Cutover.
* Downtime considerations.
* Rollback.
* Monitoring.
* Backup and recovery considerations.
* Migration risks and dependencies.
* Future modernization direction.

### 5.2 Out of Scope

The following are not part of the initial Rehost implementation:

* Complete application refactoring.
* Microservices redesign.
* Containerization.
* ECS deployment.
* Kubernetes implementation.
* Major application code changes.
* Full cloud-native redesign.

Containerization and ECS are considered a **subsequent modernization phase**.

---

## 6. AWS 6R Assessment

The AWS 6R framework is used to evaluate possible migration approaches.

### 6.1 Rehost

Rehost means moving the application to AWS with minimal modification.

It is commonly referred to as **lift and shift**.

For this migration:

```text
On-Premises Application
          |
          | Minimal changes
          v
      AWS EC2
```

#### Assessment

Rehost is selected for the initial migration because the immediate requirement is to move the existing application to AWS without simultaneously redesigning the application.

---

### 6.2 Replatform

Replatform means making selected optimizations while moving the application.

Examples could include:

* Moving a self-managed database to Amazon RDS.
* Changing part of the infrastructure to a managed AWS service.
* Making limited runtime or platform changes.

#### Assessment

Replatform may be appropriate for future stages, but it is not the selected strategy for the initial migration because the current objective is a minimal-change migration.

---

### 6.3 Repurchase

Repurchase means replacing the existing application with a different commercial or SaaS solution.

#### Assessment

This is not applicable to the current migration objective because the existing application is intended to be retained and moved to AWS.

---

### 6.4 Refactor / Re-architect

Refactoring involves substantially changing the application architecture to take advantage of cloud-native capabilities.

Examples include:

* Microservices
* Containers
* Serverless architecture
* Managed AWS services
* Event-driven architecture

#### Assessment

Refactoring is not selected for the initial migration because it would significantly increase the scope and complexity of the migration.

It can be considered after the application has been successfully migrated and stabilized.

---

### 6.5 Retain

Retain means keeping the application in the existing environment.

#### Assessment

Retain is not selected because the objective of this exercise is to migrate the application to AWS.

---

### 6.6 Retire

Retire means removing an application that is no longer required.

#### Assessment

Retire is not selected because the application is assumed to remain required by the organization.

---

## 7. 6R Decision

| 6R Strategy | Decision      | Reason                                   |
| ----------- | ------------- | ---------------------------------------- |
| Rehost      | **Selected**  | Move application with minimal changes    |
| Replatform  | Future option | Could optimize selected components later |
| Repurchase  | Not selected  | Existing application is being retained   |
| Refactor    | Future option | Can modernize after stabilization        |
| Retain      | Not selected  | Migration to AWS is required             |
| Retire      | Not selected  | Application is still required            |

#### Selected Strategy

**Rehost — Lift and Shift**

The application will first be moved to EC2 with minimal modification.

---

## 8. Why Rehost is Appropriate

Rehost is appropriate for the initial migration for several reasons.

#### 8.1 Minimal application changes

The existing application can largely retain its current architecture and runtime.

#### 8.2 Lower initial migration complexity

The migration focuses primarily on infrastructure rather than simultaneously changing the application architecture.

#### 8.3 Faster path to AWS

The organization can establish the workload in AWS before undertaking modernization.

#### 8.4 Reduced migration variables

Changing the application architecture and infrastructure simultaneously makes troubleshooting more difficult.

With Rehost:

```text
First:
On-Premises → EC2

Then:
EC2 → Containerized/ECS
```

#### 8.5 Provides a modernization foundation

After the application is stable on AWS, modernization can be considered as a separate initiative.

AWS also identifies rehost as a migration pattern that can be implemented using migration tooling such as AWS Transform MGN, while another approach is to redeploy an application using an AMI and deployment pipeline when state replication is not required.

---

## 9. Target AWS Architecture

The target environment uses the secure multi-tier VPC architecture.

```text
                         INTERNET
                            |
                            v
                    Internet Gateway
                            |
             +--------------+--------------+
             |                             |
        Public Subnet A               Public Subnet B
             |                             |
             +--------------+--------------+
                            |
                            v
                 Application Load Balancer
                            |
                 +----------+----------+
                 |                     |
                 v                     v
          Private App A          Private App B
          10.0.11.0/24           10.0.12.0/24
                 |                     |
               EC2-A                 EC2-B
                 |                     |
                 +----------+----------+
                            |
                            v
                     Private DB Tier
```

The EC2 application servers are located in private application subnets.

The Application Load Balancer is located in public subnets and provides the controlled entry point for users.

---

## 10. Target Network Design

The VPC uses the following network structure:

| Tier        | Availability Zone A | Availability Zone B |
| ----------- | ------------------- | ------------------- |
| Public      | `10.0.101.0/24`       | `10.0.102.0/24`       |
| Application | `10.0.111.0/24`      | `10.0.112.0/24`      |
| Database    | `10.0.121.0/24`      | `10.0.122.0/24`      |

VPC CIDR:

```text
10.0.0.0/16
```

This provides logical separation between public, application, and database workloads.

AWS recommends that the future-state design identify the target Region, VPCs, Availability Zones, components being migrated, new components, and interactions with other services.

---

## 11. Target Components

| Component          | Current State                   | Target State                              | Migration Treatment         |
| ------------------ | ------------------------------- | ----------------------------------------- | --------------------------- |
| Application Server | On-premises                     | Amazon EC2                                | Rehost                      |
| Application        | On-premises                     | EC2                                       | Rehost                      |
| Network            | On-premises network             | AWS VPC                                   | New AWS infrastructure      |
| Load Balancer      | Existing/on-premises assumption | AWS ALB                                   | New AWS component           |
| Application Subnet | On-premises                     | Private AWS subnet                        | New AWS infrastructure      |
| Database           | Existing dependency             | Retained/migrated according to assessment | To be determined            |
| Administration     | On-premises access              | AWS Systems Manager                       | New AWS capability          |
| Monitoring         | Existing monitoring             | CloudWatch                                | AWS operational capability  |
| IAM                | On-premises access model        | AWS IAM                                   | New AWS security capability |

The actual treatment of individual components must be confirmed during detailed assessment because an application can contain components requiring different migration strategies.

---

## 12. Migration Phases

The migration will be performed in controlled phases.

```text
1. Discover
      ↓
2. Assess
      ↓
3. Design
      ↓
4. Prepare AWS
      ↓
5. Build EC2
      ↓
6. Migrate Application
      ↓
7. Test
      ↓
8. Cutover
      ↓
9. Validate
      ↓
10. Stabilize
      ↓
11. Future Modernization
```

---

## 13. Phase 1 – Discovery

The existing application must first be understood.

The discovery process should identify:

* Application servers
* Operating systems
* CPU and memory usage
* Storage requirements
* Application ports
* Runtime dependencies
* Databases
* External APIs
* DNS dependencies
* Authentication
* Certificates
* Scheduled jobs
* File dependencies
* Network dependencies
* Current monitoring
* Backup requirements

The purpose is to avoid discovering critical dependencies during the migration itself.

---

## 14. Phase 2 – Application Assessment

The collected information is used to assess whether the application can run on EC2 with minimal changes.

Questions include:

* Is the operating system supported?
* What EC2 resources are required?
* Does the application require persistent storage?
* What ports must be opened?
* What database does it use?
* Does it require outbound internet access?
* Does it depend on internal on-premises services?
* Does it require specific software licenses?
* Does it require a particular runtime version?
* Are there hard-coded IP addresses or hostnames?
* Are there security or compliance requirements?

The answers influence the final target design.

---

## 15. Phase 3 – AWS Environment Preparation

Before migrating the application, the AWS environment should be prepared.

Required infrastructure includes:

* VPC
* Public subnets
* Private application subnets
* Private database subnets
* Internet Gateway
* NAT Gateways
* Route Tables
* Security Groups
* Network ACLs
* IAM roles
* EC2
* Application Load Balancer
* Target Group
* Monitoring

The AWS foundation should be established before application cutover.

AWS notes that foundational elements such as accounts, networking, and core services may already exist; otherwise, the foundational design should be developed alongside the application design.

---

## 16. Phase 4 – EC2 Provisioning

EC2 instances are provisioned in the private application subnets.

Example:

```text
AZ-A                         AZ-B
 |                            |
Private App A                Private App B
 |                            |
EC2-A                        EC2-B
```

Each instance should be configured with:

* Appropriate AMI
* Appropriate instance type
* EBS storage
* Application Security Group
* IAM instance role
* Private IP address
* Required application runtime

The configuration should be based on the assessment of the existing on-premises workload.

---

## 17. Phase 5 – Application Installation and Migration

The application is transferred from the on-premises environment to EC2.

Depending on the application, this may include:

* Application binaries/source
* Configuration files
* Dependencies
* Runtime
* Static files
* Certificates
* Required data
* Scheduled tasks

The application is then configured to run on the EC2 instance.

The objective is to reproduce the existing application behavior with minimal architectural changes.

---

## 18. Phase 6 – Data Migration

Application data must be migrated separately from application code where applicable.

The appropriate method depends on the existing database and data architecture.

Possible approaches include:

* Backup and restore
* Replication
* Database migration tools
* File synchronization
* Application-level export/import

The migration plan must ensure data consistency before cutover.

If the database is eventually moved to a managed AWS service such as Amazon RDS, that would represent a later modernization/replatforming decision rather than pure Rehost.

---

## 19. Phase 7 – Security Configuration

Security is applied before the application becomes accessible.

### Security Groups

Example:

```text
Internet
   |
   v
ALB Security Group
   |
   v
Application Security Group
   |
   v
EC2
```

The EC2 application servers should accept application traffic from the ALB Security Group rather than unrestricted internet sources.

### NACLs

NACLs provide subnet-level traffic control.

### Private subnets

EC2 application instances do not require public IP addresses.

### IAM

EC2 instances use IAM roles rather than embedded AWS credentials.

---

### 7.1 IAM Strategy

IAM follows the principle of least privilege.

The EC2 instance should receive an IAM role containing only the permissions required by the workload and management processes.

For administration, AWS Systems Manager can provide secure access to private instances.

```text
Administrator
      |
      v
Systems Manager
      |
      v
Private EC2
```

This avoids the need to expose management access directly to the public internet.

AWS identifies IAM for secure access control and Systems Manager for patching, remote access, and maintenance as common components of a migrated application architecture.

---

### 7.2 Application Load Balancer

The Application Load Balancer provides the entry point for external users.

Traffic flow:

```text
User
 |
 v
Internet
 |
 v
ALB
 |
 v
Target Group
 |
 +---- EC2-A
 |
 +---- EC2-B
```

The ALB performs health checks on the EC2 instances.

Only healthy instances should receive normal application traffic.

---

### 7.3 Outbound Connectivity

Private EC2 instances may require outbound connectivity for:

* OS updates
* Package downloads
* External APIs
* AWS service communication where required

The application subnets therefore use NAT Gateways.

```text
Private EC2
     |
     v
Route Table
     |
     v
NAT Gateway
     |
     v
Internet Gateway
     |
     v
Internet
```

The NAT Gateway provides outbound connectivity without making the EC2 instances directly internet-facing.

---

## 20. Phase 8 -  Monitoring and Operations

Monitoring should be established before cutover.

CloudWatch can be used for:

* EC2 metrics
* Application logs
* System logs
* ALB metrics
* Target health
* Alarms

Operational readiness should also be considered, including who will support the application, incident response procedures, expected service levels, separation of duties, and team responsibilities. AWS specifically recommends considering these operational questions during application design.

---

## 21. Phase 9 - Backup and Recovery

Before migration:

* Verify existing backups.
* Create a recoverable copy of application data.
* Confirm the restoration process.
* Document recovery responsibilities.

After migration:

* Configure EC2/EBS backup requirements.
* Monitor backup success.
* Test restoration where practical.

The original on-premises environment should remain available until the AWS environment is sufficiently validated.

---
## 22. Phase 10 -  Rollback Strategy

Rollback must be defined before cutover.

The original on-premises application should remain available during the initial AWS validation period.

If a critical issue occurs:

```text
AWS
 |
 | Failure
 v
Rollback Decision
 |
 v
Redirect Traffic
 |
 v
On-Premises Application
```

Possible rollback conditions include:

* Application failure
* Data inconsistency
* Critical performance problems
* Network failure
* Security issue
* Unresolved dependency
* Unexpected application behavior

The rollback decision should have clear ownership and predefined criteria.

---

## 23.  Phase 11 -  Testing Strategy

Testing should occur before production cutover.

### Functional Testing

Verify:

* Application starts.
* Users can access required features.
* Application dependencies work.
* Database connectivity works.

### Network Testing

Verify:

* ALB → EC2 works.
* EC2 → database works where required.
* EC2 → external services works where required.
* NAT connectivity works.

### Security Testing

Verify:

* EC2 is not directly internet accessible.
* Only required ports are open.
* Security Groups are correctly configured.
* NACLs are correctly configured.
* IAM permissions are sufficient but not excessive.

### Performance Testing

Compare application performance against the existing environment where practical.

---

## 24. Phase 12 -  Cutover Strategy

The cutover moves production traffic from the on-premises environment to AWS.

A controlled cutover should follow:

```text
On-Premises
    |
    | Final synchronization
    v
AWS EC2
    |
    | Final validation
    v
AWS ALB
    |
    v
Users
```

Before cutover:

* AWS environment is ready.
* Application has been tested.
* Data is synchronized.
* Monitoring is active.
* Rollback plan is ready.
* Stakeholders are informed.

AWS recommends documenting cutover considerations and anticipated testing, audit, validation, and proof-of-concept requirements as part of the migration design.

---

## 25. Phase 13 - Downtime Considerations
The amount of downtime depends on the application's architecture and the method used to migrate its data.

Potential downtime can occur during:

* Final data synchronization
* Application shutdown
* Final configuration changes
* DNS changes
* Application startup
* Validation

Downtime can be minimized by:

1. Preparing the AWS environment in advance.
2. Installing the application before the migration window.
3. Performing data synchronization before cutover.
4. Scheduling cutover during a low-traffic period.
5. Keeping the on-premises environment available during validation.

The exact downtime cannot be determined until the application's current architecture and data dependencies are assessed.

---

## 26. Phase 14 - Risks, Assumptions, Issues and Dependencies

AWS recommends explicitly documenting unresolved risks, assumptions, issues, and dependencies and assigning ownership to them.

### Risks

| Risk                            | Impact                          | Mitigation                               |
| ------------------------------- | ------------------------------- | ---------------------------------------- |
| Unknown application dependency  | Application failure             | Perform discovery and dependency mapping |
| Incorrect EC2 sizing            | Performance issues              | Analyze current resource utilization     |
| Data migration failure          | Data loss/inconsistency         | Backup and validate data                 |
| Incorrect network configuration | Connectivity failure            | Test routes, SGs and NACLs               |
| DNS/cutover issue               | Users cannot access application | Pre-test DNS and maintain rollback       |
| Security misconfiguration       | Unauthorized access             | Least-privilege security review          |
| Unexpected application behavior | Service disruption              | Perform functional testing               |
| Insufficient monitoring         | Slow issue detection            | Enable monitoring before cutover         |

### Assumptions

* The application can run on an AWS-supported operating system.
* The application can be hosted on EC2 without major code changes.
* Required application dependencies can be transferred or recreated.
* Required data can be migrated.
* AWS networking can support the application's connectivity requirements.

### Dependencies

* Application dependency assessment
* Database/data migration
* DNS
* Network connectivity
* IAM
* Security configuration
* Application owners
* Infrastructure team
* Migration team

---

## 27. Phase 15 - Cost Considerations

The target AWS architecture will introduce AWS infrastructure costs.

Potential cost areas include:

* EC2
* EBS
* Application Load Balancer
* NAT Gateways
* Elastic IPs where applicable
* CloudWatch
* Data transfer
* Database services if migrated

AWS recommends estimating the target architecture's running cost using the AWS Pricing Calculator and including required software licensing costs.

A detailed cost estimate should be performed after the EC2 sizing and data-transfer requirements are known.

---

## 28. Phase 16 -  Migration Validation

The migration is considered successful when the following have been validated.

### Infrastructure

* [ ] VPC is operational.
* [ ] Required subnets exist.
* [ ] Route tables are correct.
* [ ] NAT connectivity works.
* [ ] EC2 instances are running.

### Security

* [ ] Security Groups are correct.
* [ ] NACLs are correct.
* [ ] EC2 instances are private.
* [ ] IAM roles are attached.
* [ ] Systems Manager access works where required.

### Application

* [ ] Application starts successfully.
* [ ] Required dependencies are available.
* [ ] ALB targets are healthy.
* [ ] Application is accessible through the ALB.
* [ ] Functional testing passes.

### Data

* [ ] Required data is available.
* [ ] Data integrity is verified.
* [ ] Database connectivity works.

### Operations

* [ ] Monitoring is active.
* [ ] Logs are available.
* [ ] Backups are configured.
* [ ] Rollback remains possible.

---

## 29. Phase 17 -  Post-Migration Stabilization

After cutover, the application should enter a stabilization period.

During this period:

* Monitor application performance.
* Monitor EC2 health.
* Review application logs.
* Monitor ALB target health.
* Monitor AWS costs.
* Resolve migration-related issues.
* Confirm backups.
* Validate user experience.

The original environment should only be decommissioned after the agreed validation and rollback period.

---

## 30.  Phase 18 - Future Modernization

Rehost is the first stage rather than necessarily the final target architecture.

Once the application is stable on EC2, it can be modernized.

The next planned stage is:

```text
                    MIGRATION
                       |
                       v
              On-Premises → EC2
                    Rehost
                       |
                       v
              Stable AWS Workload
                       |
                       v
                 Containerize
                       |
                       v
                 Docker Image
                       |
                       v
                     ECR
                       |
                       v
                     ECS
```

This creates a controlled progression:

**Migrate → Stabilize → Modernize**

The second document will therefore address how the application running on EC2 can be containerized and deployed using Amazon ECS.

---

## 31.  Architectural Decision

### Decision

**Rehost the application to Amazon EC2 as the initial migration strategy.**

### Reason

The immediate objective is to move the existing application from on-premises infrastructure to AWS while minimizing application changes.

### Alternatives Considered

* Replatform
* Refactor
* Repurchase
* Retain
* Retire

### Rationale

Rehost provides a relatively direct migration path and allows the organization to separate the risk of infrastructure migration from the risk of application modernization.

Future modernization can then be performed after the workload is established and validated in AWS.

---

## 32. Final Migration Strategy

The complete migration strategy is:

```text
┌──────────────────────────┐
│     ON-PREMISES          │
│                          │
│ Existing Application     │
│ Existing Dependencies    │
└────────────┬─────────────┘
             │
             │ 1. Discover
             │ 2. Assess
             │ 3. Evaluate 6Rs
             │
             ▼
      ┌───────────────┐
      │  REHOST       │
      │ Lift & Shift  │
      └───────┬───────┘
              │
              │ 4. Prepare AWS
              │ 5. Provision EC2
              │ 6. Migrate App
              │ 7. Test
              │
              ▼
┌─────────────────────────────────┐
│             AWS                 │
│                                 │
│       Secure VPC                │
│          │                      │
│       ALB                       │
│          │                      │
│    Private EC2                  │
│          │                      │
│    Database Tier                │
└───────────────┬─────────────────┘
                │
                │ 8. Cutover
                │ 9. Validate
                │ 10. Stabilize
                │
                ▼
       ┌──────────────────┐
       │ FUTURE PHASE     │
       │                  │
       │ Docker → ECR     │
       │       → ECS      │
       └──────────────────┘
```

---

##  33. Conclusion

The proposed migration follows a **6R-based assessment** and selects **Rehost (Lift and Shift)** as the initial strategy for moving the application from on-premises infrastructure to AWS.

The application will be deployed on Amazon EC2 within the private application tier of a secure, multi-AZ VPC. An Application Load Balancer will provide controlled external access, while Security Groups, Network ACLs, IAM, private subnets, NAT Gateways, monitoring, and backup mechanisms provide the supporting security and operational controls.

The migration will be performed in stages:

**Discover → Assess → Design → Prepare → Migrate → Test → Cutover → Validate → Stabilize**

The original on-premises environment will remain available during the agreed rollback period.

After successful stabilization, the application can move to the next modernization phase: **containerization and deployment using Amazon ECS**.

The overall strategy is therefore:

> **First migrate the application to AWS with minimal change. Then stabilize it. Then modernize it.**

This staged approach keeps the initial migration focused while creating a clear path toward a more cloud-native architecture.
