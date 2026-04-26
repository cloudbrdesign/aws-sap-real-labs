# AWS SAP: Why Your EC2 Tagging Policy Fails (Even When It’s Correct)

This lab is part of my **AWS Certified Solutions Architect Professional** preparation series.

Instead of memorizing answers, we take real exam scenarios and turn them into **hands-on AWS labs**.

---

# 🧠 Scenario

A large company operates multiple AWS accounts.

Engineers launch EC2 instances and EBS volumes daily.

Over time:
- Most resources are **untagged**
- Accounts hit **service limits**
- No one knows **who owns what**
- Cleanup becomes **impossible**

Management introduces a rule:

> All EC2 resources must include mandatory tags before creation.

---

# 🎯 Objective

- Enforce tagging using IAM policies
- Understand why a **correct-looking policy fails**
- Fix the policy using proper scoping

---

# 🧠 Core Concept (Theory)

## IAM Tag-Based Access Control

AWS allows you to enforce tagging using condition keys such as:

- `aws:RequestTag/<tag-key>`
- `aws:TagKeys`

Example:

```json
"Condition": {
  "Null": {
    "aws:RequestTag/Owner": "true"
  }
}
```

This means:

    Deny the request if the “Owner” tag is missing.

**🚨 The Hidden Problem**
-------------------------

When you run:
```
aws ec2 run-instances
```
AWS does NOT only evaluate the EC2 instance.

It evaluates multiple resources:

*   EC2 instance
    
*   Network interface
    
*   EBS volumes
    
*   Security groups
    
*   Subnet
    
*   AMI
    

👉 This is critical.

**❌ Why the Policy Fails**
--------------------------

If you write:
```
"Resource": "*"
```
You are applying the condition to **ALL resources involved**.

But:

*   Network interfaces do NOT have aws:RequestTag/Owner
    
*   So the condition fails
    
*   AWS returns:

```
UnauthorizedOperation ... explicit deny ... network-interface/*
```

**✅ Correct Approach**
----------------------

Scope the policy to only resources that support tagging:
```
arn:aws:ec2:*:*:instance/*
arn:aws:ec2:*:*:volume/*
```
**⚙️ Lab Setup**
================

**1\. Set Environment Variables**
---------------------------------

```
export AWS_REGION=af-south-1
```

Get Account ID:
```
export ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)
```

Get latest AMI:
```
export AMI_ID=$(aws ec2 describe-images \
  --region $AWS_REGION \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" \
  "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)
```
Get subnet:
```
export SUBNET_ID=$(aws ec2 describe-subnets \
  --region $AWS_REGION \
  --query "Subnets[0].SubnetId" \
  --output text)
```

**2\. Create IAM User**
-----------------------

```
aws iam create-user \
  --user-name tag-enforcement-lab-user
```

Create access key:

```
aws iam create-access-key \
  --user-name tag-enforcement-lab-user
```
**3\. Configure CLI Profile**
-----------------------------

```
aws configure --profile taglab-demo
```

Enter:

*   Access Key
    
*   Secret Key
    
*   Region
    

**4\. Verify Profile**
----------------------

```
aws sts get-caller-identity \
  --profile taglab-demo
```

**🚀 Lab Execution**
====================

**Step 1 — Allow EC2 Access**
-----------------------------

Create policy:

```
cat > allow.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*"
    }
  ]
}
EOF
```

Create in AWS:

```
aws iam create-policy \
  --policy-name AllowEC2Demo \
  --policy-document file://allow.json
```

Attach:

```
aws iam attach-user-policy \
  --user-name tag-enforcement-lab-user \
  --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AllowEC2Demo
```

**Step 2 — Baseline Test**
--------------------------

```
aws ec2 run-instances \
  --profile taglab-demo \
  --region $AWS_REGION \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $SUBNET_ID \
  --count 1
```

✅ This works (no tagging required)

**Step 3 — Add Deny Policy (WRONG)**
------------------------------------

```
cat > deny.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestTag/Owner": "true"
        }
      }
    }
  ]
}
EOF
```

Create:
```
aws iam create-policy \
  --policy-name DenyUntaggedEC2 \
  --policy-document file://deny.json
```

Attach:

```
aws iam attach-user-policy \
  --user-name tag-enforcement-lab-user \
  --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/DenyUntaggedEC2
```

**Step 4 — Test Without Tags**
------------------------------

```
aws ec2 run-instances \
  --profile taglab-demo \
  --region $AWS_REGION \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $SUBNET_ID \
  --count 1
```

❌ Fails (expected)

**Step 5 — Test With Tags**
---------------------------

```
aws ec2 run-instances \
  --profile taglab-demo \
  --region $AWS_REGION \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $SUBNET_ID \
  --count 1 \
  --tag-specifications \
  'ResourceType=instance,Tags=[{Key=Owner,Value=Hudson}]'
```

❌ Still fails

**🔍 Observe the Error**
------------------------

You will see something like:

```
not authorized to perform ec2:RunInstances on resource network-interface/*
```

👉 This is the key learning moment.

**Step 6 — Fix the Policy**
---------------------------

```
cat > fixed.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "Null": {
          "aws:RequestTag/Owner": "true"
        }
      }
    }
  ]
}
EOF
```

Update policy:

```
aws iam create-policy-version \
  --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/DenyUntaggedEC2 \
  --policy-document file://fixed.json \
  --set-as-default
```

**Step 7 — Final Test**
-----------------------

```
aws ec2 run-instances \
  --profile taglab-demo \
  --region $AWS_REGION \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $SUBNET_ID \
  --count 1 \
  --tag-specifications \
  'ResourceType=instance,Tags=[{Key=Owner,Value=Hudson}]'
```

✅ Works

**🧠 Exam Takeaway**
====================

*   EC2 RunInstances evaluates **multiple resources**
    
*   Using "Resource": "\*" can break policies
    
*   Use **tag condition keys carefully**
    
*   Scope policies to relevant resources
    

**🧹 Cleanup**
==============

Terminate instances:

```
aws ec2 terminate-instances \
  --instance-ids <INSTANCE_ID>
```

Detach policies:

```
aws iam detach-user-policy ...
```

Delete policies:

```
aws iam delete-policy ...
```

**🎯 Final Insight**
====================

The logic was correct. The scope was wrong.

That’s the level AWS SAP expects you to think at.