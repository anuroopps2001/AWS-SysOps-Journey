## 🟦 Introduction

This document covers full AWS load balancing and scaling workflow:

- #### ALB (Application Load Balancer)

- #### Target Groups

- #### Auto Scaling Group (ASG)

You will understand:

- How clients reach your application

- How traffic flows from ALB → TG → EC2

- How ASG replaces unhealthy instances

- How scaling policies work

- How to debug issues

## 🟧 Application Load Balancer (ALB)
🔹 What is ALB?

Application Load Balancer distributes HTTP/HTTPS traffic across multiple EC2s, containers (ECS), or IP-based targets.

It works at Layer 7 (HTTP/HTTPS).

Supports:

- Path-based routing (/api/*, /images/*)

- Host-based routing (admin.example.com, app.example.com)

- Weighted routing (20/80 split)

- HTTP → HTTPS redirection

- TLS termination (SSL offloading)

### 🔹 ALB Features
#### 1. Listeners

A listener is:
```bash
Protocol + Port
```

example:
```bash
HTTP : 80
HTTPS : 443
```

Listener rules decide how ALB forwards traffic.

On what port and Protocol, ALB dns should be accessed is decided by Listeners

#### 2. Security Groups

ALB security group must allow:

| Rule        | Recommendation |
| ----------- | -------------- |
| Inbound 80  | 0.0.0.0/0      |
| Inbound 443 | 0.0.0.0/0      |
| Outbound    | Allow all      |


Instance SG should allow traffic on port where application is running from ALB SG

#### 3. Cross-Zone Load Balancing

Ensures even distribution across AZs.

#### Enabled by default for ALB.


#### 4. Access Logs

ALB can store logs in S3:

- Client IP

- Request URL

- Latency

- Target response code

- User-agent

Enable via CLI:
```bash
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn <alb-arn> \
  --attributes Key=access_logs.s3.enabled,Value=true \
               Key=access_logs.s3.bucket,Value=my-log-bucket \
               Key=access_logs.s3.prefix,Value=alb-logs
```

## 🟩 Target Groups (TG)

Target Group = collection of backend targets that the ALB sends traffic to.

.

### 🔹 Target Types
| Type         | Used When                          |
| ------------ | ---------------------------------- |
| **Instance** | EC2 instances                      |
| **IP**       | ECS “Awsvpc” mode, on-prem servers |
| **Lambda**   | HTTP → Lambda routing              |


Most commonly for EC2:
```bash
Instance mode
Protocol: HTTP
Port: 80
```

### 🔹 Health Checks

TG health checks decide if instance is healthy.

Example configuration:
```bash
Protocol: HTTP
Port: traffic port (80)
Path: /health or /status.html
Healthy threshold: 5
Unhealthy threshold: 2
Timeout: 5 sec
Interval: 30 sec
```

If instance returns non-200, TG marks it unhealthy.

### 🔹 Weighted Target Groups

ALB can forward requests to multiple TGs with a specific weight:

Example:

TG1 → 70% traffic

TG2 → 30% traffic

This supports:

- Blue/Green deployments

- Canary releases

- Version-based routing

## 🟦 ALB → TG Routing Flow
```java
Clients
   │
   ▼
Application Load Balancer (listener 80/443)
   │ applies rules
   ▼
Target Group (health checks, port 80)
   │
   ▼
EC2 Instances
```

## 🟥 Auto Scaling Group (ASG)

ASG ensures:

- Automatic scaling (increase/decrease EC2 count)

- Automatic healing (replace unhealthy nodes)

- Efficient cost control

## 🔹 Launch Template (LT)

LT contains:

- AMI ID

- Instance type

- Security groups

- Key pair

- IAM instance profile

- User data (install Apache etc.)

User data example:
```bash
#!/bin/bash
yum install httpd -y
systemctl enable httpd
echo "Hello from ASG instance" > /var/www/html/index.html
systemctl start httpd
```

### 🔹 ASG Capacity Settings
| Parameter           | Meaning                       |
| ------------------- | ----------------------------- |
| **MinSize**         | Lowest number of EC2s allowed |
| **DesiredCapacity** | Target number of EC2s         |
| **MaxSize**         | Upper limit of EC2s           |


For auto-healing tests:
```bash
Min = 2
Desired = 2
Max = 4
```

### 🔹 Health Check Types
| Type    | Behavior                     |
| ------- | ---------------------------- |
| **EC2** | Only instance status checks  |
| **ELB** | Uses ALB Target Group health |


Use ELB for auto-healing:
```bash
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --health-check-type ELB \
  --health-check-grace-period 60
```

### 🔹 Auto-Healing Process

1. You stop Apache:
```bash
sudo systemctl stop httpd
```
2. ALB marks instance unhealthy

3. ASG sees "unhealthy" (because health check type = ELB)

4. ASG terminates instance

5. ASG launches a new instance

6. New instance becomes healthy and is added to TG

### 🔹 Scaling Policies

#### 1. Target Tracking

Maintains a metric at a target value.

Example:
```bash
Target CPU = 70%
```
Behavior:

- CPU > 70% → scale OUT

- CPU < 70% → scale IN

`Important: Target tracking ALWAYS scales both directions, unless you disable scale-in.`

#### 2. Step Scaling

Example:
```bash
If CPU > 80% for 2 minutes → add 1 instance  
If CPU > 90% for 5 minutes → add 2 instances  
```

#### 3. Scheduled Scaling

Useful for predictable peaks:

- Weekday morning spikes

- Black Friday

- Month-end batch jobs


## 🟥 Troubleshooting Guide
### 1. Target = Unhealthy

Health check path incorrect

Security group blocks ALB → EC2

Application not listening on correct port

Wrong protocol

### 2. Target = Unused

TG attached to wrong listener

ALB is in different AZ

ASG using wrong TG

### 3. ASG not launching new instance

DesiredCapacity = 1

Scaling policy scaled IN

Instance profile missing

Subnets not available

### 4. Auto-healing not working

HealthCheckType = EC2 (wrong)

Should be ELB

### 5. Weighted routing not applying

You modified wrong listener

Rule priority conflict

Cached DNS (browser)
