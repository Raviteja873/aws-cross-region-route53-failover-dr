# AWS Cross-Region Route 53 Failover & Disaster Recovery

## Project Overview

This project demonstrates a **cross-region, active-passive Disaster Recovery architecture on AWS** for a simple web application.

The primary environment is hosted in **Mumbai (`ap-south-1`)** and the secondary Disaster Recovery environment is hosted in **Hyderabad (`ap-south-2`)**.

Users access the application through:

**http://ravitejaaws.dpdns.org**

Under normal conditions, Amazon Route 53 directs traffic to the Mumbai primary environment. The Mumbai environment is fronted by an internet-facing Application Load Balancer and serves the application through an EC2 instance running Nginx.

A separate Hyderabad environment is maintained as the secondary/DR environment. During the failover test, the Mumbai application is made unhealthy and Route 53 directs traffic to the Hyderabad environment. After Mumbai is restored, the primary path is validated again.

### AWS services used

- Amazon Route 53
- Amazon VPC
- Internet Gateway
- Route Tables
- Public Subnets
- Security Groups
- IAM
- AWS Systems Manager Session Manager
- Amazon EC2
- Application Load Balancer
- Target Groups
- Nginx

### Project scope

This project demonstrates **web-tier cross-region failover**. It does not include database replication or persistent application-data replication.

---

# Architecture

![AWS Cross-Region Route 53 Failover Architecture](images/architecture-diagram.png)

## Architecture Explanation

The architecture consists of two independent AWS regional environments connected through DNS-based failover.

```text
                              Internet
                                  |
                                  v
                       ravitejaaws.dpdns.org
                                  |
                                  v
                       Amazon Route 53
                       Failover Routing
                         /             \
                        /               \
                       v                 v
              Mumbai - Primary     Hyderabad - DR
               ap-south-1           ap-south-2
                    |                    |
                    v                    v
              Internet ALB          Internet ALB
                    |                    |
                    v                    v
              Target Group          Target Group
                    |                    |
                    v                    v
                  EC2                  EC2
                    |                    |
                    v                    v
                 Nginx                Nginx
```

### Normal operation

```text
User
 |
 v
Route 53
 |
 v
Mumbai Primary ALB
 |
 v
Mumbai Target Group
 |
 v
Mumbai EC2
 |
 v
Nginx
```

### Failure and recovery

```text
Mumbai application becomes unhealthy
              |
              v
       Mumbai target unhealthy
              |
              v
        Route 53 failover
              |
              v
      Hyderabad DR ALB
              |
              v
      Hyderabad Target Group
              |
              v
       Hyderabad EC2
              |
              v
            Nginx

              |
        Mumbai recovery
              |
              v
       Mumbai becomes healthy
              |
              v
       Primary path restored
```

---

# Environment Summary

| Component | Mumbai — Primary | Hyderabad — Secondary / DR |
|---|---|---|
| Region | `ap-south-1` | `ap-south-2` |
| VPC | `Mumbai-Primary` | `Hyderabad-Secondary` |
| VPC CIDR | `10.10.0.0/16` | `10.20.0.0/16` |
| Public Subnet 1 | `10.10.1.0/24` | `10.20.1.0/24` |
| Public Subnet 2 | `10.10.2.0/24` | `10.20.2.0/24` |
| EC2 | `Mumbai-Primary-EC2` | `Hyderabad-Secondary-EC2` |
| Target Group | `Mumbai-Primary-TG` | `Hyderabad-Secondary-TG` |
| ALB | `Mumbai-Primary-ALB` | `Hyderabad-Secondary-ALB` |
| Web Server | Nginx | Nginx |
| Health Endpoint | `/health` | `/health` |
| DNS Role | Primary | Secondary |

---

# 1. Mumbai — Primary Environment

The Mumbai region is the main application environment used during normal operation.

## Region and VPC

**Region:** Mumbai — `ap-south-1`

**VPC:** `Mumbai-Primary`

**CIDR:** `10.10.0.0/16`

The VPC contains the public networking required for the internet-facing ALB and EC2 workload.

![Mumbai Region](images/01-mumbai-region.png)

![Mumbai VPC](images/01-mumbai-vpc.png)

![Mumbai VPC Configuration](images/02-mumbai-vpc-and-more-configuration.png)

![Mumbai VPC Resource Map](images/04-mumbai-vpc-resource-map.png)

![Mumbai VPC CIDR](images/05-mumbai-vpc-cidr.png)

---

## Mumbai Public Subnets

Two public subnets are used for the regional load balancer:

- `Mumbai-Public-Subnet-1` — `10.10.1.0/24`
- `Mumbai-Public-Subnet-2` — `10.10.2.0/24`

The subnets provide multi-AZ placement for the internet-facing ALB.

![Mumbai Public Subnets](images/06-mumbai-public-subnets.png)

---

## Mumbai Internet Gateway and Routing

The Mumbai VPC uses an Internet Gateway for internet connectivity.

The public route table contains the internet route:

```text
0.0.0.0/0 → Mumbai-Primary-IGW
```

![Mumbai Internet Gateway](images/07-mumbai-internet-gateway.png)

![Mumbai Public Route Table](images/08-mumbai-public-route-table.png)

![Mumbai Route Table and Subnet Associations](images/09-mumbai-route-table-subnet-associations.png)

---

## Mumbai Security Groups

The ALB and EC2 use separate security groups.

The ALB security group provides the public HTTP entry point, while the EC2 security group allows application traffic from the ALB security group.

![Mumbai ALB Security Group](images/01-mumbai-alb-sg-created.png)

![Mumbai EC2 Security Group](images/02-mumbai-ec2-sg-created.png)

This creates the intended traffic relationship:

```text
Internet
   |
   | HTTP : 80
   v
Mumbai ALB
   |
   | HTTP : 80
   v
Mumbai EC2
```

---

## Mumbai IAM and Session Manager

The EC2 instance uses an IAM role for Systems Manager access.

```text
Mumbai-EC2-SSM-Role
```

The role uses the managed policy:

```text
AmazonSSMManagedInstanceCore
```

Session Manager is used for administration and for the controlled failover test.

![Mumbai EC2 SSM Role](images/01-mumbai-ec2-ssm-role.png)

![Mumbai SSM Session](images/02-mumbai-ssm-session.png)

---

## Mumbai EC2 and Nginx

The primary EC2 instance is:

```text
Mumbai-Primary-EC2
```

It runs Amazon Linux 2023 with Nginx as the web server.

![Mumbai Primary EC2](images/01-mumbai-primary-ec2-running.png)

The application exposes a simple health endpoint:

```text
/health
```

which returns:

```text
OK
```

![Mumbai Nginx Health Check](images/03-mumbai-nginx-health-check.png)

---

## Mumbai Target Group and ALB

The EC2 instance is registered with:

```text
Mumbai-Primary-TG
```

The target group uses HTTP port 80 and checks:

```text
/health
```

![Target Group Configuration](images/01-target-group-configuration.png)

![Mumbai EC2 Registered](images/02-mumbai-ec2-registered.png)

![Mumbai Target Healthy](images/03-mumbai-target-healthy.png)

The target group is connected to:

```text
Mumbai-Primary-ALB
```

The ALB is internet-facing and receives HTTP traffic before forwarding requests to the healthy EC2 target.

![Mumbai ALB Created](images/02-mumbai-alb-created.png)

![Mumbai ALB DNS Name](images/03-mumbai-alb-dns-name.png)

![Mumbai ALB Test](images/04-mumbai-primary-alb-test.png)

---

# 2. Hyderabad — Secondary / Disaster Recovery Environment

The Hyderabad environment provides the secondary application stack used when the primary environment is unavailable.

## Region and VPC

**Region:** Hyderabad — `ap-south-2`

**VPC:** `Hyderabad-Secondary`

**CIDR:** `10.20.0.0/16`

![Hyderabad VPC Configuration](images/01-hyderabad-vpc-and-more.png)

![Hyderabad VPC Resource Map](images/02-hyderabad-vpc-resource-map.png)

---

## Hyderabad Public Subnets

Two public subnets are used:

- `Hyderabad-Public-Subnet-1` — `10.20.1.0/24`
- `Hyderabad-Public-Subnet-2` — `10.20.2.0/24`

![Hyderabad Public Subnets](images/03-hyderabad-public-subnets.png)

---

## Hyderabad Internet Gateway and Routing

The Hyderabad VPC has its own Internet Gateway and public route table.

```text
0.0.0.0/0 → Hyderabad-Secondary-IGW
```

![Hyderabad Internet Gateway](images/04-hyderabad-internet-gateway.png)

![Hyderabad Route Table](images/05-hyderabad-route-table.png)

---

## Hyderabad Security Groups

The secondary environment follows the same application access model:

```text
Internet
   |
   v
Hyderabad ALB
   |
   v
Hyderabad EC2
```

![Hyderabad ALB Security Group](images/01-hyderabad-alb-sg.png)

![Hyderabad EC2 Security Group](images/02-hyderabad-ec2-sg.png)

---

## Hyderabad IAM and Session Manager

The secondary EC2 uses:

```text
Hyderabad-EC2-SSM-Role
```

with Systems Manager permissions.

![Hyderabad SSM Role](images/01-hyderabad-ssm-role.png)

![Hyderabad SSM Policy](images/02-hyderabad-ssm-policy.png)

![Hyderabad SSM Session](images/02-hyderabad-ssm-session.png)

---

## Hyderabad EC2 and Nginx

The DR EC2 instance is:

```text
Hyderabad-Secondary-EC2
```

![Hyderabad Secondary EC2](images/01-hyderabad-secondary-ec2-running.png)

Nginx provides the secondary application response and the `/health` endpoint used for target health monitoring.

![Hyderabad Nginx Health Check](images/Screenshot%202026-10-06%2003-hyderabad-nginx-health-check.png)

---

## Hyderabad ALB

The secondary load balancer is:

```text
Hyderabad-Secondary-ALB
```

![Hyderabad ALB Created](images/02-hyderabad-alb-created.png)

![Hyderabad ALB DNS Name](images/03-hyderabad-alb-dns-name.png)

![Hyderabad ALB Network Mapping](images/01-alb-network-mapping.png)

The secondary application was also tested directly through the ALB before the Route 53 failover test.

![Hyderabad Secondary ALB Test](images/04-hyderabad-secondary-alb-test.png)

---

# 3. Route 53 DNS and Failover

The public application domain is:

```text
ravitejaaws.dpdns.org
```

The domain is delegated to a Route 53 public hosted zone.

![Domain Registration](images/04-route53-domain-registration.png)

![Route 53 Hosted Zone](images/05-route53-hosted-zone.png)

![Route 53 Name Servers](images/06-route53-nameservers.png)

The hosted zone contains two failover A records for the same domain:

```text
ravitejaaws.dpdns.org
        |
        +---- Primary → Mumbai ALB
        |
        +---- Secondary → Hyderabad ALB
```

---

## Primary Record

The primary Route 53 record points to:

```text
Mumbai-Primary-ALB
```

with:

```text
Routing Policy: Failover
Role: Primary
Alias: Yes
Evaluate Target Health: Yes
```

![Route 53 Primary Record](images/07-route53-primary-record.png)

---

## Secondary Record

The secondary record points to:

```text
Hyderabad-Secondary-ALB
```

with:

```text
Routing Policy: Failover
Role: Secondary
Alias: Yes
Evaluate Target Health: Yes
```

![Route 53 Secondary Record](images/08-route53-secondary-record.png)

---

## Complete Failover Configuration

The final Route 53 configuration contains both primary and secondary destinations.

![Both Primary and Secondary Records](images/04-both-primary-secondary-records.png)

![Route 53 Failover Configuration](images/09-route53-failover-complete.png)

---

# 4. Normal Application Traffic

Before testing the Disaster Recovery behavior, DNS resolution and the primary application were validated.

![Route 53 DNS Resolution](images/10-route53-dns-resolution.png)

With Mumbai healthy, the domain reaches the primary application:

```text
http://ravitejaaws.dpdns.org
```

![Mumbai Primary Browser Test](images/11-route53-primary-browser-test.png)

The normal path is:

```text
User
 |
 v
Route 53
 |
 v
Mumbai ALB
 |
 v
Mumbai Target Group
 |
 v
Mumbai EC2
 |
 v
Nginx
```

---

# 5. Disaster Recovery Failover Test

The failover test simulates an application failure in the Mumbai environment.

The primary Nginx service is intentionally stopped through Session Manager. This causes the `/health` endpoint to fail and the target to become unhealthy.

![Failover Trigger](images/12-route53-failover-trigger.png)

![Mumbai Primary Unhealthy](images/13-route53-primary-unhealthy.png)

The failure sequence is:

```text
Mumbai Nginx unavailable
          |
          v
/health check fails
          |
          v
Mumbai target becomes unhealthy
          |
          v
Primary environment is no longer preferred
          |
          v
Route 53 selects Secondary
```

---

# 6. Traffic Served by Hyderabad

The same DNS name is tested after the Mumbai failure.

```text
http://ravitejaaws.dpdns.org
```

Traffic is now served by the Hyderabad environment.

![Route 53 Failover to Secondary](images/14-route53-failover-to-secondary.png)

The failover path becomes:

```text
User
 |
 v
Route 53
 |
 | Primary unhealthy
 v
Hyderabad ALB
 |
 v
Hyderabad Target Group
 |
 v
Hyderabad EC2
 |
 v
Nginx
```

This confirms the core objective of the project: **the secondary regional web environment can serve traffic when the primary environment becomes unhealthy.**

---

# 7. Primary Recovery

After the failover test, the Mumbai application is restored.

The Mumbai Nginx service is started again and its health endpoint is validated.

The target subsequently returns to a healthy state.

![Mumbai Primary Recovery](images/15-route53-primary-recovery.png)

The domain is then tested again after recovery.

![Final Route 53 Recovery](images/16-route53-final-recovery.png)

The final state returns to:

```text
User
 |
 v
Route 53
 |
 v
Mumbai Primary ALB
 |
 v
Mumbai Target Group
 |
 v
Mumbai EC2
 |
 v
Nginx
```

---

# 8. End-to-End Result

The complete project behavior can be summarized in three states.

### State 1 — Normal

```text
Mumbai Healthy
      |
      v
Route 53
      |
      v
Mumbai Primary
```

### State 2 — Failure

```text
Mumbai Unhealthy
      |
      v
Route 53 Failover
      |
      v
Hyderabad Secondary / DR
```

### State 3 — Recovery

```text
Mumbai Healthy Again
      |
      v
Primary Path Available
```

The project therefore demonstrates:

- Multi-region AWS deployment
- Independent regional networking
- Load-balanced web applications
- Application health monitoring
- DNS-based failover
- Disaster Recovery testing
- Secondary-region traffic handling
- Primary-region recovery

---

# 9. Evidence in This Repository

The `images/` directory contains the AWS Console evidence captured during the project.

### Architecture

```text
architecture-diagram.png
architecture-diagram.png.png
```

### Mumbai

```text
01-mumbai-region.png
01-mumbai-vpc.png
02-mumbai-vpc-and-more-configuration.png
04-mumbai-vpc-resource-map.png
05-mumbai-vpc-cidr.png
06-mumbai-public-subnets.png
07-mumbai-internet-gateway.png
08-mumbai-public-route-table.png
09-mumbai-route-table-subnet-associations.png

01-mumbai-alb-sg-created.png
02-mumbai-ec2-sg-created.png
01-mumbai-ec2-ssm-role.png
02-mumbai-ssm-session.png
01-mumbai-primary-ec2-running.png

03-mumbai-nginx-health-check.png
01-target-group-configuration.png
02-mumbai-ec2-registered.png
03-mumbai-target-healthy.png

02-mumbai-alb-created.png
03-mumbai-alb-dns-name.png
04-mumbai-primary-alb-test.png
```

### Hyderabad

```text
01-hyderabad-vpc-and-more.png
02-hyderabad-vpc-resource-map.png
03-hyderabad-public-subnets.png
04-hyderabad-internet-gateway.png
05-hyderabad-route-table.png

01-hyderabad-alb-sg.png
02-hyderabad-ec2-sg.png
01-hyderabad-ssm-role.png
02-hyderabad-ssm-policy.png
02-hyderabad-ssm-session.png

01-hyderabad-secondary-ec2-running.png
02-hyderabad-alb-created.png
03-hyderabad-alb-dns-name.png
01-alb-network-mapping.png
04-hyderabad-secondary-alb-test.png
Screenshot 2026-10-06 03-hyderabad-nginx-health-check.png
```

### Route 53 and failover validation

```text
04-route53-domain-registration.png
05-route53-hosted-zone.png
06-route53-nameservers.png
07-route53-primary-record.png
08-route53-secondary-record.png
09-route53-failover-complete.png
10-route53-dns-resolution.png
11-route53-primary-browser-test.png
12-route53-failover-trigger.png
13-route53-primary-unhealthy.png
14-route53-failover-to-secondary.png
15-route53-primary-recovery.png
16-route53-final-recovery.png
04-both-primary-secondary-records.png
```

---

# Project Outcome

The final implementation provides an **active-passive cross-region web Disaster Recovery design** using Amazon Route 53 Failover Routing.

The project demonstrates the complete lifecycle:

```text
Build
  ↓
Primary application
  ↓
Secondary / DR application
  ↓
DNS configuration
  ↓
Normal traffic
  ↓
Primary failure
  ↓
Route 53 failover
  ↓
Secondary serves traffic
  ↓
Primary recovery
  ↓
Normal traffic restored
```

**Primary Region:** Mumbai — `ap-south-1`  
**Secondary Region:** Hyderabad — `ap-south-2`  
**Domain:** `ravitejaaws.dpdns.org`

---

## Author

**Ravi Teja**

MCA Final-Year Student | AWS Cloud & DevOps Learner
