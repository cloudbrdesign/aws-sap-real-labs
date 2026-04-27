# AWS Certified Solutions Architect Professional — Real Labs & Deep Dives

This repository contains a hands-on lab series for the AWS Certified Solutions Architect Professional (SAP-C02) exam.

Instead of memorizing answers, we reverse-engineer real exam scenarios into practical AWS labs.

---

## 🎯 Approach

Each episode:
- Starts from a real AWS SAP scenario
- Builds a working solution
- Breaks it intentionally
- Debugs the issue
- Fixes it correctly

---

## 📚 Episodes

### 01 — EC2 Tag Enforcement

👉 [Why Your EC2 Tagging Policy Fails (Even When It’s Correct)](episodes/01-ec2-tag-enforcement)

Concepts:
- IAM condition keys
- aws:RequestTag
- EC2 RunInstances behavior
- Multi-resource authorization

---

### 02 — Route 53 Failover & Disaster Recovery

👉 [How Route 53 Fails Over Between Regions (Real DR Lab)](episodes/02-route53-failover-dr)

Concepts:

- Route 53 failover routing (active-passive)
- DNS health checks and failover behavior
- Application Load Balancer (ALB) multi-AZ behavior
- Auto Scaling Groups with target groups
- Multi-region disaster recovery (DR) patterns
- Why DNS failover depends on health check configuration


---

## 🚀 How to Use This Repository

1. Pick an episode
2. Open the README inside the episode folder
3. Follow the CLI steps
4. Run the lab in your AWS account

---

## ⚠️ Important

- Labs are designed to be low-cost
- Always run cleanup steps after each lab
- Use a sandbox AWS account if possible

---

## 🧠 Goal

This is not about passing the exam.

This is about:
> Thinking like a Solutions Architect under real-world constraints.
