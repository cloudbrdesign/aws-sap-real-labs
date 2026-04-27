# AWS SAP: How Route 53 Fails Over Between Regions (Real DR Lab)

This lab is part of the **AWS Certified Solutions Architect Professional — Real Labs & Deep Dives** series.

Instead of memorizing answers, we take AWS SAP-style scenarios and turn them into real, hands-on AWS labs.

---

## Scenario

A company based on the west coast of North America is preparing to launch a new customer-facing web application.

The application will be deployed in the **us-west-1** region using Amazon EC2 instances.

The company has the following requirements:

- The application must be highly available across multiple Availability Zones
- It must automatically scale based on user demand
- It requires a disaster recovery environment in **us-east-1**
- The secondary region should act as a passive failover site
- Traffic should automatically fail over if the primary region becomes unhealthy

---

## Objective

In this lab, you will build:

- A primary web application stack in **us-west-1**
- A secondary disaster recovery stack in **us-east-1**
- Application Load Balancers in both regions
- Auto Scaling Groups behind each ALB
- Route 53 active-passive failover using your domain

For this lab, I use:

```text
clusterforge.net
```
The application endpoint will be:
```
app.clusterforge.net
```

**Architecture**
----------------

```text
app.clusterforge.net
        |
        v
Amazon Route 53 Failover Routing
        |
        |-- PRIMARY: us-west-1
        |       |
        |       +-- Application Load Balancer
        |       +-- Target Group
        |       +-- Auto Scaling Group
        |       +-- EC2 instances running nginx
        |
        |-- SECONDARY: us-east-1
                |
                +-- Application Load Balancer
                +-- Target Group
                +-- Auto Scaling Group
                +-- EC2 instances running nginx
```
**Core Theory**
---------------

An Application Load Balancer is used as the single entry point for application traffic. It distributes incoming requests across registered targets such as EC2 instances, monitors target health, and routes traffic only to healthy targets. AWS documents this as a core Elastic Load Balancing behavior.  

For high availability, an Application Load Balancer should span subnets in multiple Availability Zones. AWS notes that an ALB requires subnets from different Availability Zones, and it is most effective when each enabled zone has at least one registered target.  

Auto Scaling Groups help maintain the desired number of EC2 instances and can automatically register launched instances with an Elastic Load Balancing target group.  

Route 53 failover routing is used for active-passive disaster recovery. The primary record receives traffic while healthy; if its health check fails, Route 53 can return the secondary record instead.  

**Important Cost Warning**
--------------------------

This lab creates billable resources:

*   2 Application Load Balancers
    
*   EC2 instances in two regions
    
*   Auto Scaling Groups
    
*   Route 53 health checks
    
*   Route 53 DNS records
    

Run the cleanup steps immediately after completing the lab.

**Prerequisites**
-----------------

You need:

*   AWS CLI installed and configured
    
*   A Route 53 hosted zone for your domain
    
*   Permissions to create:
    
    *   EC2 resources
        
    *   Security groups
        
    *   Launch templates
        
    *   Auto Scaling Groups
        
    *   Application Load Balancers
        
    *   Route 53 health checks and records
        

This lab assumes your domain is hosted in Route 53:
```bash
clusterforge.net
```

**Step 1 — Define Lab Variables**
=================================

```bash
export DOMAIN_NAME="clusterforge.net"
export RECORD_NAME="app.clusterforge.net"

export PRIMARY_REGION="us-west-1"
export SECONDARY_REGION="us-east-1"

export PROJECT_NAME="sap-episode-02-dr"
```

Confirm:
```bash
echo $DOMAIN_NAME
echo $RECORD_NAME
echo $PRIMARY_REGION
echo $SECONDARY_REGION
echo $PROJECT_NAME
```

**Step 2 — Discover Default VPCs**
==================================

For this lab, we use the default VPC in each region.

In production, you would usually build dedicated VPCs with proper subnet design, routing, NAT, observability, and security controls. For this learning lab, default VPCs keep the focus on the AWS SAP concept: **multi-region active-passive failover**.

**Primary Region VPC**
----------------------

```bash
export PRIMARY_VPC_ID=$(aws ec2 describe-vpcs \
  --region $PRIMARY_REGION \
  --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text)

echo $PRIMARY_VPC_ID
```
**Secondary Region VPC**
------------------------

```bash
export SECONDARY_VPC_ID=$(aws ec2 describe-vpcs \
  --region $SECONDARY_REGION \
  --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text)

echo $SECONDARY_VPC_ID
```

Expected output:
```text
vpc-xxxxxxxx
vpc-yyyyyyyy
```

**Step 3 — Discover Subnets in Two Availability Zones**
=======================================================

An Application Load Balancer requires at least two Availability Zone subnets. Each subnet should be in a different Availability Zone.  

**Primary Region Subnets**
--------------------------

```bash
export PRIMARY_SUBNET_1=$(aws ec2 describe-subnets \
  --region $PRIMARY_REGION \
  --filters "Name=vpc-id,Values=$PRIMARY_VPC_ID" \
  --query "Subnets[0].SubnetId" \
  --output text)

export PRIMARY_SUBNET_2=$(aws ec2 describe-subnets \
  --region $PRIMARY_REGION \
  --filters "Name=vpc-id,Values=$PRIMARY_VPC_ID" \
  --query "Subnets[1].SubnetId" \
  --output text)

echo $PRIMARY_SUBNET_1
echo $PRIMARY_SUBNET_2
```
**Secondary Region Subnets**
----------------------------

```bash
export SECONDARY_SUBNET_1=$(aws ec2 describe-subnets \
  --region $SECONDARY_REGION \
  --filters "Name=vpc-id,Values=$SECONDARY_VPC_ID" \
  --query "Subnets[0].SubnetId" \
  --output text)

export SECONDARY_SUBNET_2=$(aws ec2 describe-subnets \
  --region $SECONDARY_REGION \
  --filters "Name=vpc-id,Values=$SECONDARY_VPC_ID" \
  --query "Subnets[1].SubnetId" \
  --output text)

echo $SECONDARY_SUBNET_1
echo $SECONDARY_SUBNET_2
```

At this point, you should have:

```bash
echo "Primary VPC: $PRIMARY_VPC_ID"
echo "Primary Subnets: $PRIMARY_SUBNET_1 $PRIMARY_SUBNET_2"

echo "Secondary VPC: $SECONDARY_VPC_ID"
echo "Secondary Subnets: $SECONDARY_SUBNET_1 $SECONDARY_SUBNET_2"
```

**Step 4 — Create Security Groups**
=======================================================

We need two security groups in each region:

```text
ALB Security Group
Allows HTTP from the internet

EC2 Security Group
Allows HTTP only from the ALB security group
```

### **Primary ALB Security Group**

```bash
export PRIMARY_ALB_SG_ID=$(aws ec2 create-security-group \
  --region $PRIMARY_REGION \
  --group-name "${PROJECT_NAME}-primary-alb-sg" \
  --description "Primary ALB security group" \
  --vpc-id $PRIMARY_VPC_ID \
  --query "GroupId" \
  --output text)

echo $PRIMARY_ALB_SG_ID
```

Allow HTTP from the internet:

```bash
aws ec2 authorize-security-group-ingress \
  --region $PRIMARY_REGION \
  --group-id $PRIMARY_ALB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
  ```

### **Primary EC2 Security Group**

```bash
export PRIMARY_EC2_SG_ID=$(aws ec2 create-security-group \
  --region $PRIMARY_REGION \
  --group-name "${PROJECT_NAME}-primary-ec2-sg" \
  --description "Primary EC2 security group" \
  --vpc-id $PRIMARY_VPC_ID \
  --query "GroupId" \
  --output text)

echo $PRIMARY_EC2_SG_ID
```

Allow HTTP only from the ALB:

```bash
aws ec2 authorize-security-group-ingress \
  --region $PRIMARY_REGION \
  --group-id $PRIMARY_EC2_SG_ID \
  --protocol tcp \
  --port 80 \
  --source-group $PRIMARY_ALB_SG_ID
```

### **Secondary ALB Security Group**

```bash
export SECONDARY_ALB_SG_ID=$(aws ec2 create-security-group \
  --region $SECONDARY_REGION \
  --group-name "${PROJECT_NAME}-secondary-alb-sg" \
  --description "Secondary ALB security group" \
  --vpc-id $SECONDARY_VPC_ID \
  --query "GroupId" \
  --output text)

echo $SECONDARY_ALB_SG_ID
```

```bash
aws ec2 authorize-security-group-ingress \
  --region $SECONDARY_REGION \
  --group-id $SECONDARY_ALB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

### **Secondary EC2 Security Group**

```bash
export SECONDARY_EC2_SG_ID=$(aws ec2 create-security-group \
  --region $SECONDARY_REGION \
  --group-name "${PROJECT_NAME}-secondary-ec2-sg" \
  --description "Secondary EC2 security group" \
  --vpc-id $SECONDARY_VPC_ID \
  --query "GroupId" \
  --output text)

echo $SECONDARY_EC2_SG_ID
```

```bash
aws ec2 authorize-security-group-ingress \
  --region $SECONDARY_REGION \
  --group-id $SECONDARY_EC2_SG_ID \
  --protocol tcp \
  --port 80 \
  --source-group $SECONDARY_ALB_SG_ID
```

**Step 5 — Create User Data for the Web Application**
=======================================================


Yes — since this is a web application lab, we need the EC2 instances to serve HTTP traffic.

We’ll install **nginx** and return a simple page showing whether the request reached the primary or secondary region.

### **Primary User Data**

```bash
cat > user-data-primary.sh <<'EOF'
#!/bin/bash
dnf update -y
dnf install -y nginx
systemctl enable nginx
systemctl start nginx

cat > /usr/share/nginx/html/index.html <<HTML
<html>
  <head><title>Primary Region</title></head>
  <body style="font-family: Arial; background: #0b1020; color: white;">
    <h1>PRIMARY REGION</h1>
    <h2>us-west-1</h2>
    <p>Traffic is currently served from the primary region.</p>
  </body>
</html>
HTML
EOF
```
### **Secondary User Data**

```bash
cat > user-data-secondary.sh <<'EOF'
#!/bin/bash
dnf update -y
dnf install -y nginx
systemctl enable nginx
systemctl start nginx

cat > /usr/share/nginx/html/index.html <<HTML
<html>
  <head><title>Secondary Region</title></head>
  <body style="font-family: Arial; background: #101010; color: white;">
    <h1>SECONDARY REGION</h1>
    <h2>us-east-1</h2>
    <p>Traffic has failed over to the disaster recovery region.</p>
  </body>
</html>
HTML
EOF
```

**Step 6 — Get AMIs for Both Regions**
=======================================================

```bash
export PRIMARY_AMI_ID=$(aws ec2 describe-images \
  --region $PRIMARY_REGION \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

echo $PRIMARY_AMI_ID
```

```bash
export SECONDARY_AMI_ID=$(aws ec2 describe-images \
  --region $SECONDARY_REGION \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

echo $SECONDARY_AMI_ID
```

**Step 7 — Create Launch Templates**
=======================================================

Auto Scaling groups can use launch templates to define the EC2 configuration, including AMI, instance type, security group, and user data.  

### **Primary Launch Template**

```bash
export PRIMARY_USER_DATA=$(base64 -i user-data-primary.sh)

aws ec2 create-launch-template \
  --region $PRIMARY_REGION \
  --launch-template-name "${PROJECT_NAME}-primary-lt" \
  --launch-template-data "{
    \"ImageId\":\"$PRIMARY_AMI_ID\",
    \"InstanceType\":\"t3.micro\",
    \"SecurityGroupIds\":[\"$PRIMARY_EC2_SG_ID\"],
    \"UserData\":\"$PRIMARY_USER_DATA\"
  }"

  ```
  ### **Secondary Launch Template**

  ```bash
  export SECONDARY_USER_DATA=$(base64 -i user-data-secondary.sh)

aws ec2 create-launch-template \
  --region $SECONDARY_REGION \
  --launch-template-name "${PROJECT_NAME}-secondary-lt" \
  --launch-template-data "{
    \"ImageId\":\"$SECONDARY_AMI_ID\",
    \"InstanceType\":\"t3.micro\",
    \"SecurityGroupIds\":[\"$SECONDARY_EC2_SG_ID\"],
    \"UserData\":\"$SECONDARY_USER_DATA\"
  }"
  ```

  **Step 8 — Create Target Groups**
=======================================================

Target groups route requests to registered targets, and health checks are configured at the target group level. ALBs use these health checks to send traffic only to healthy targets.  

### **Primary Target Group**

```bash
export PRIMARY_TG_ARN=$(aws elbv2 create-target-group \
  --region $PRIMARY_REGION \
  --name "sap-ep02-primary-tg" \
  --protocol HTTP \
  --port 80 \
  --vpc-id $PRIMARY_VPC_ID \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-path "/" \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text)

echo $PRIMARY_TG_ARN
```

### **Secondary Target Group**

```bash
export SECONDARY_TG_ARN=$(aws elbv2 create-target-group \
  --region $SECONDARY_REGION \
  --name "sap-ep02-secondary-tg" \
  --protocol HTTP \
  --port 80 \
  --vpc-id $SECONDARY_VPC_ID \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-path "/" \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text)

echo $SECONDARY_TG_ARN
```

**Step 9 — Create Application Load Balancers**
=======================================================

An ALB should be created in multiple Availability Zones, and each enabled zone should ideally contain at least one registered target.  

### **Primary ALB**

```bash
export PRIMARY_ALB_ARN=$(aws elbv2 create-load-balancer \
  --region $PRIMARY_REGION \
  --name "sap-ep02-primary-alb" \
  --subnets $PRIMARY_SUBNET_1 $PRIMARY_SUBNET_2 \
  --security-groups $PRIMARY_ALB_SG_ID \
  --scheme internet-facing \
  --type application \
  --query "LoadBalancers[0].LoadBalancerArn" \
  --output text)

echo $PRIMARY_ALB_ARN
```

Get DNS name:
```bash
export PRIMARY_ALB_DNS=$(aws elbv2 describe-load-balancers \
  --region $PRIMARY_REGION \
  --load-balancer-arns $PRIMARY_ALB_ARN \
  --query "LoadBalancers[0].DNSName" \
  --output text)

echo $PRIMARY_ALB_DNS
```

Get ALB hosted zone ID:
```bash
export PRIMARY_ALB_ZONE_ID=$(aws elbv2 describe-load-balancers \
  --region $PRIMARY_REGION \
  --load-balancer-arns $PRIMARY_ALB_ARN \
  --query "LoadBalancers[0].CanonicalHostedZoneId" \
  --output text)

echo $PRIMARY_ALB_ZONE_ID
```

### **Secondary ALB**

```bash
export SECONDARY_ALB_ARN=$(aws elbv2 create-load-balancer \
  --region $SECONDARY_REGION \
  --name "sap-ep02-secondary-alb" \
  --subnets $SECONDARY_SUBNET_1 $SECONDARY_SUBNET_2 \
  --security-groups $SECONDARY_ALB_SG_ID \
  --scheme internet-facing \
  --type application \
  --query "LoadBalancers[0].LoadBalancerArn" \
  --output text)

echo $SECONDARY_ALB_ARN
```

```bash
export SECONDARY_ALB_DNS=$(aws elbv2 describe-load-balancers \
  --region $SECONDARY_REGION \
  --load-balancer-arns $SECONDARY_ALB_ARN \
  --query "LoadBalancers[0].DNSName" \
  --output text)

echo $SECONDARY_ALB_DNS
```

```bash
export SECONDARY_ALB_ZONE_ID=$(aws elbv2 describe-load-balancers \
  --region $SECONDARY_REGION \
  --load-balancer-arns $SECONDARY_ALB_ARN \
  --query "LoadBalancers[0].CanonicalHostedZoneId" \
  --output text)

echo $SECONDARY_ALB_ZONE_ID
```

**Step 10 — Create ALB Listeners**
=======================================================

### **Primary Listener**

```bash
aws elbv2 create-listener \
  --region $PRIMARY_REGION \
  --load-balancer-arn $PRIMARY_ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$PRIMARY_TG_ARN
```

### **Secondary Listener**

```bash
aws elbv2 create-listener \
  --region $SECONDARY_REGION \
  --load-balancer-arn $SECONDARY_ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$SECONDARY_TG_ARN
```

**Step 11 — Create Auto Scaling Groups**
=======================================================

Auto Scaling can automatically register launched instances with the target group and deregister them when they are terminated.  

### **Primary Auto Scaling Group**

```bash
aws autoscaling create-auto-scaling-group \
  --region $PRIMARY_REGION \
  --auto-scaling-group-name "${PROJECT_NAME}-primary-asg" \
  --launch-template "LaunchTemplateName=${PROJECT_NAME}-primary-lt,Version=\$Latest" \
  --min-size 1 \
  --max-size 2 \
  --desired-capacity 1 \
  --vpc-zone-identifier "$PRIMARY_SUBNET_1,$PRIMARY_SUBNET_2" \
  --target-group-arns $PRIMARY_TG_ARN \
  --health-check-type ELB \
  --health-check-grace-period 120
```

### **Secondary Auto Scaling Group**

```bash
aws autoscaling create-auto-scaling-group \
  --region $SECONDARY_REGION \
  --auto-scaling-group-name "${PROJECT_NAME}-secondary-asg" \
  --launch-template "LaunchTemplateName=${PROJECT_NAME}-secondary-lt,Version=\$Latest" \
  --min-size 1 \
  --max-size 2 \
  --desired-capacity 1 \
  --vpc-zone-identifier "$SECONDARY_SUBNET_1,$SECONDARY_SUBNET_2" \
  --target-group-arns $SECONDARY_TG_ARN \
  --health-check-type ELB \
  --health-check-grace-period 120
```

Wait a few minutes for instances to launch and pass health checks.

**Step 12 — Validate Target Health**
=======================================================

### **Primary**

```bash
aws elbv2 describe-target-health \
  --region $PRIMARY_REGION \
  --target-group-arn $PRIMARY_TG_ARN \
  --query "TargetHealthDescriptions[].TargetHealth.State" \
  --output table
```

Expected:

```text
healthy
```

### **Secondary**

```bash
aws elbv2 describe-target-health \
  --region $SECONDARY_REGION \
  --target-group-arn $SECONDARY_TG_ARN \
  --query "TargetHealthDescriptions[].TargetHealth.State" \
  --output table
```

Expected:

```text
healthy
```

**Step 13 — Test Both ALBs Directly**
=======================================================

```bash
curl http://$PRIMARY_ALB_DNS
```

Expected:

```text
PRIMARY REGION
us-west-1
```

```bash
curl http://$SECONDARY_ALB_DNS
```

Expected:

```text
SECONDARY REGION
us-east-1
```

**Step 14 — Find Route 53 Hosted Zone**
=======================================================

```bash
export HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name \
  --dns-name $DOMAIN_NAME \
  --query "HostedZones[0].Id" \
  --output text | sed 's|/hostedzone/||')

echo $HOSTED_ZONE_ID
```
**Step 15 — Create a Broken Health Check First**
=======================================================

This is our intentional failure.

We will create a Route 53 health check that points to the primary ALB, but checks a path that does not exist:

```text
/broken-health
```

The ALB itself may be working, but the Route 53 health check will fail.

This teaches the key lesson:

```text
DNS failover depends on the health check you configure.
```

Route 53 supports active-passive failover using primary and secondary records, where the primary record is used when healthy and traffic can fail over to the secondary record when the primary is unhealthy.

```bash
export PRIMARY_HEALTH_CHECK_ID=$(aws route53 create-health-check \
  --caller-reference "${PROJECT_NAME}-primary-broken-$(date +%s)" \
  --health-check-config "{
    \"IPAddress\": null,
    \"Port\": 80,
    \"Type\": \"HTTP\",
    \"ResourcePath\": \"/broken-health\",
    \"FullyQualifiedDomainName\": \"$PRIMARY_ALB_DNS\",
    \"RequestInterval\": 30,
    \"FailureThreshold\": 3
  }" \
  --query "HealthCheck.Id" \
  --output text)

echo $PRIMARY_HEALTH_CHECK_ID
```

