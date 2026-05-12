# AWS SAP: Confused Deputy — Why External ID Matters

This lab is part of the **AWS Certified Solutions Architect Professional — Real Labs & Deep Dives** series.

Instead of memorizing exam answers, we turn AWS SAP-style scenarios into real, hands-on labs.

---

## Scenario

A technology company is preparing for a financial audit.

The company plans to use a third-party web application that needs limited AWS access to discover EC2 resources running in the company’s AWS account.

The company’s internal security policy requires:

- Outside access must follow least privilege
- The third-party vendor must not receive long-term AWS access keys for the customer account
- The credentials used by the vendor must not be usable by another third party
- The vendor has provided:
  - Its AWS account number
  - A unique customer ID

---

## Question

How should the Solutions Architect securely grant the third-party vendor access while preventing the confused deputy problem?

---

## Correct Architecture Pattern

Create an IAM role for the third-party vendor.

The role should have:

1. A **permissions policy** that allows only the required AWS actions  
2. A **trust policy** that allows the vendor to assume the role  
3. A condition requiring the correct `sts:ExternalId`

The important part is this condition:

```json
"Condition": {
  "StringEquals": {
    "sts:ExternalId": "unique-customer-id"
  }
}
```

## Core Theory — The Confused Deputy Problem

Before we touch AWS, let’s understand the *real problem* we are solving.

---

### 🎭 The Story

Let’s define the actors in this scenario:

- **You (The Customer)**  
  You own the AWS account and your EC2 resources.

- **The Vendor (The Deputy)**  
  A third-party application that acts *on your behalf*.

- **Another Customer (The Attacker)**  
  Someone else using the same vendor platform.

---

### 🧠 What Is a “Deputy”?

A **deputy** is simply:

> *A system or service that has permission to act on behalf of someone else.*

In AWS:
- The **third-party application** is the deputy  
- It uses your IAM Role to perform actions for you  

---

### ⚠️ Where Things Go Wrong

The confused deputy problem happens when:

> The deputy performs an action using **its own permissions**,  
instead of verifying **who it is acting for**

---

### 🧾 Real-World Analogy

Imagine this:

You give an **AI assistant** permission to:
- access your AWS account
- analyze EC2 resources
- generate reports

Now imagine this AI serves **multiple customers**.

---

#### 🟢 Normal Behavior

You ask:

```text
“Analyze my EC2 resources”
```

AI uses:

-   your role
-   your context
-   your permissions

Everything works correctly.

* * *

#### **🔴 The Attack**

Another customer says:

```text
“Use this role ARN and analyze MY resources”
```

But they pass **your role ARN**.

The AI:

-   doesn’t verify context
-   trusts the request blindly
-   assumes your role

💥 Result:  
The AI accesses **your account**, but thinks it’s working for someone else.

* * *

### **🎯 That’s the Confused Deputy**

The deputy (AI / vendor):

-   is **not malicious**
-   but is **confused about who it is serving**

It becomes a **bridge for privilege escalation**

* * *

### **🧩 Mapping This to AWS**

| Concept | AWS Equivalent |
| :--- | :--- |
| Deputy | Third-party vendor app |
| You | AWS account owner |
| Attacker | Another vendor customer |
| Authority | IAM Role |
| Action | STS AssumeRole |

## **How We Simulate This With One AWS Account**

In the real world, this uses two AWS accounts:

```text
Customer AWS Account
        |
        | IAM Role with ExternalId trust condition
        v
Third-Party Vendor AWS Account
```

For this lab, we simulate the same concept inside one AWS account:

```text
Your AWS Account
        |
        | IAM Role: audit-discovery-role
        |
        | trusted by
        v
IAM User: vendor-app-user
```
The IAM user represents the third-party vendor application.

This is not a perfect replacement for true cross-account access, but it lets us test the important behavior:

```text
AssumeRole without ExternalId       → should fail after fix
AssumeRole with wrong ExternalId    → should fail
AssumeRole with correct ExternalId  → should succeed
```

## **What You Will Build**

## 

You will create:

-   A simulated third-party IAM user
-   An IAM role representing the customer-side access role
-   A least-privilege EC2 discovery policy
-   An unsafe trust policy first
-   A fixed trust policy using `sts:ExternalId`
-   STS `AssumeRole` tests

* * *

## **Lab Objective**

## 

By the end of this lab, you will understand:

-   Why sharing IAM access keys with third parties is risky
-   How IAM roles provide temporary credentials
-   What the confused deputy problem means in AWS
-   Why `ExternalId` protects third-party access
-   How to test whether a role is misconfigured
-   How this appears in the AWS SAP exam

* * *

# **Step 1 — Set Environment Variables**

```bash
export AWS_REGION="af-south-1"
export LAB_USER="vendor-app-user"
export ROLE_NAME="audit-discovery-role"
export POLICY_NAME="audit-ec2-discovery-policy"
export EXTERNAL_ID="clusterforge-audit-customer-001"
export WRONG_EXTERNAL_ID="attacker-customer-999"
```
Get your AWS account ID:
```bash
export ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)

echo $ACCOUNT_ID

```
Create useful ARNs:
```bash
export LAB_USER_ARN="arn:aws:iam::${ACCOUNT_ID}:user/${LAB_USER}"
export ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/${ROLE_NAME}"

echo $LAB_USER_ARN
echo $ROLE_ARN
```

# **Step 2 — Create the Simulated Vendor IAM User**

## 

This user represents the third-party vendor’s web application.

```bash
aws iam create-user \
  --user-name $LAB_USER
```

Create access keys for the simulated vendor user:

```bash
aws iam create-access-key \
  --user-name $LAB_USER
```

Save the output temporarily. You will need:

```text
AccessKeyId
SecretAccessKey
```

Configure a local AWS CLI profile:

```bash
aws configure --profile vendor-demo
```

Use:

```text
AWS Access Key ID:     <AccessKeyId>
AWS Secret Access Key: <SecretAccessKey>
Default region:        af-south-1
Default output:        json
```

Verify the profile:

```bash
aws sts get-caller-identity \
  --profile vendor-demo
```

Expected output should show the IAM user:

```text
arn:aws:iam::<account-id>:user/vendor-app-user
```

# **Step 3 — Create a Least-Privilege Permission Policy**

# 

The third-party application only needs to discover EC2 resources.

Create the permissions policy:

```bash
cat > ec2-discovery-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2DiscoveryOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeVolumes",
        "ec2:DescribeTags",
        "ec2:DescribeRegions"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```

# 

This is least privilege because it only allows read-only EC2 discovery actions.

* * *

# **Step 4 — Create an Unsafe Trust Policy First**

# 

This is the intentional mistake.

The trust policy allows the simulated vendor user to assume the role, but it does **not** require an external ID.

```bash

cat > trust-policy-unsafe.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TrustVendorWithoutExternalId",
      "Effect": "Allow",
      "Principal": {
        "AWS": "$LAB_USER_ARN"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

Create the IAM role:

```bash
aws iam create-role \
  --role-name $ROLE_NAME \
  --assume-role-policy-document file://trust-policy-unsafe.json
```

Attach the EC2 discovery policy:

```bash
aws iam put-role-policy \
  --role-name $ROLE_NAME \
  --policy-name $POLICY_NAME \
  --policy-document file://ec2-discovery-policy.json
```

Wait briefly for IAM propagation:

```bash
sleep 10
```

# 

* * *

# **Step 5 — Test the Unsafe Role**

# 

Now attempt to assume the role **without** an external ID.

```bash
aws sts assume-role \
  --profile vendor-demo \
  --role-arn $ROLE_ARN \
  --role-session-name vendor-without-external-id
```

Expected result:

```text
Success
```

# 

This is the problem.

The vendor can assume the role without proving which customer context it is acting for.

* * *

## **Why This Is Dangerous**

# 

The role ARN is not a secret.

If another customer or attacker can cause the vendor application to use this role ARN, the vendor could be tricked into assuming your role in the wrong customer context.

That is the confused deputy issue.

The role works, but it works too easily.

* * *

# **Step 6 — Save Temporary Credentials From the Unsafe AssumeRole**

```bash
aws sts assume-role \
  --profile vendor-demo \
  --role-arn $ROLE_ARN \
  --role-session-name vendor-without-external-id \
  > unsafe-creds.json
```

Export the temporary credentials:

```bash
export AWS_ACCESS_KEY_ID=$(cat unsafe-creds.json | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(cat unsafe-creds.json | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(cat unsafe-creds.json | jq -r '.Credentials.SessionToken')
```

Verify the assumed role identity:

```bash
aws sts get-caller-identity
```

Expected output:

```text
arn:aws:sts::<account-id>:assumed-role/audit-discovery-role/vendor-without-external-id
```

# **Step 7 — Validate Least Privilege**

# 

The role should be able to discover EC2 resources:

```bash
aws ec2 describe-instances \
  --region $AWS_REGION \
  --query "Reservations[].Instances[].InstanceId" \
  --output table
```

But it should not have broader permissions.

Test an IAM action:

```bash
aws iam list-users
```

Expected result:
```text
AccessDenied
```
This proves that the permissions policy is limited.

Clear temporary credentials before continuing:

```bash
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
```

# **Step 8 — Fix the Trust Policy With ExternalId**

# 

Now we update the trust policy so the role can only be assumed when the correct external ID is supplied.

```bash
cat > trust-policy-fixed.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TrustVendorWithExternalId",
      "Effect": "Allow",
      "Principal": {
        "AWS": "$LAB_USER_ARN"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "$EXTERNAL_ID"
        }
      }
    }
  ]
}
EOF
```

Update the role trust policy:

```bash
aws iam update-assume-role-policy \
  --role-name $ROLE_NAME \
  --policy-document file://trust-policy-fixed.json
```

Wait for IAM propagation:

```bash
sleep 10
```

# **Step 9 — Test Without ExternalId**

# 

Try to assume the role again without an external ID:

```bash
aws sts assume-role \
  --profile vendor-demo \
  --role-arn $ROLE_ARN \
  --role-session-name vendor-missing-external-id
```

Expected result:

```text
AccessDenied
```

# 

This is good.

The role now rejects assume-role attempts that do not include the customer-specific external ID.

* * *

# **Step 10 — Test With the Wrong ExternalId**

# 

Now simulate the vendor acting on behalf of the wrong customer.

```bash
aws sts assume-role \
  --profile vendor-demo \
  --role-arn $ROLE_ARN \
  --role-session-name vendor-wrong-customer \
  --external-id $WRONG_EXTERNAL_ID
```

Expected result:

```text
AccessDenied
```

# 

This is the core protection.

The vendor principal is trusted, but only when it provides the correct customer context.

* * *

# **Step 11 — Test With the Correct ExternalId**

# 

Now assume the role with the correct external ID:

```bash
aws sts assume-role \
  --profile vendor-demo \
  --role-arn $ROLE_ARN \
  --role-session-name vendor-correct-customer \
  --external-id $EXTERNAL_ID
```
Expected result:

```text
Success
```

Save the credentials:

```bash
aws sts assume-role \
  --profile vendor-demo \
  --role-arn $ROLE_ARN \
  --role-session-name vendor-correct-customer \
  --external-id $EXTERNAL_ID \
  > fixed-creds.json
```

Export them:

```bash
export AWS_ACCESS_KEY_ID=$(cat fixed-creds.json | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(cat fixed-creds.json | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(cat fixed-creds.json | jq -r '.Credentials.SessionToken')
```

Verify identity:

```bash
aws sts get-caller-identity
```

Expected output:

```text
assumed-role/audit-discovery-role/vendor-correct-customer
```

# **Step 12 — Validate Final Access**

# 

The role can still perform the intended EC2 discovery:

```bash
aws ec2 describe-instances \
  --region $AWS_REGION \
  --query "Reservations[].Instances[].InstanceId" \
  --output table
```

But it still cannot perform unrelated actions:

```bash
aws iam list-users
```

Expected:

```text
AccessDenied
```

Clear credentials:

```bash
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
```

# 

* * *

# **What We Proved**

# 

Before the fix:

```text
Vendor principal + Role ARN = AssumeRole succeeds
```

After the fix:

```text
Vendor principal + Role ARN + correct ExternalId = AssumeRole succeeds
```

And:

```text
Vendor principal + Role ARN + missing/wrong ExternalId = AccessDenied
```

# 

That is the confused deputy protection.

* * *

# **Exam Takeaway**

# 

For the AWS Certified Solutions Architect Professional exam, the correct pattern is:

```text
IAM role for third-party access
Least-privilege permissions policy
Trust policy allowing the vendor principal
sts:ExternalId condition in the trust policy
Temporary credentials through STS AssumeRole
```

# 

Common wrong answers:

-   Share IAM access keys with the vendor
-   Create a highly privileged IAM user for the vendor
-   Trust the vendor account without `ExternalId`
-   Use only permissions policies and ignore the trust policy
-   Let the customer invent the external ID without vendor-side uniqueness

The key phrase to remember:

```text
External ID prevents another customer from tricking the third-party vendor into using your role.
```

# **Cleanup**

# 

Unset any temporary role credentials:

```bash
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
```

Detach/delete inline role policy:

```bash
aws iam delete-role-policy \
  --role-name $ROLE_NAME \
  --policy-name $POLICY_NAME
```

Delete the role:

```bash
aws iam delete-role \
  --role-name $ROLE_NAME
```

List access keys for the simulated vendor user:

```bash
aws iam list-access-keys \
  --user-name $LAB_USER
```

Delete the access key:

```bash
aws iam delete-access-key \
  --user-name $LAB_USER \
  --access-key-id <ACCESS_KEY_ID>
```

Delete the user:

```bash
aws iam delete-user \
  --user-name $LAB_USER
```

Delete local files:

```bash
rm -f ec2-discovery-policy.json
rm -f trust-policy-unsafe.json
rm -f trust-policy-fixed.json
rm -f unsafe-creds.json
rm -f fixed-creds.json
```

# **Final Insight**

# 

The mistake is not trusting a third party.

The mistake is trusting the third party without forcing it to prove **which customer it is acting for**.

That is what `sts:ExternalId` gives you.
