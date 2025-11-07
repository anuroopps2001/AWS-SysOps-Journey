# AWS-SysOps-Journey
# 🧭 Day 01 — AWS Account Setup & IAM Foundations

**Date:** 07-11-25  
**Goal:** Secure a new AWS account, create IAM users and groups, configure AWS CLI, and enable billing alerts.

---

## 1️⃣ Root Account Hardening

- Enabled **MFA** for root user  
- Set strong password policy  
- Restricted root usage to account-level tasks only (no daily use)

> **Best Practice:** Root user should only be used for billing, support, and global settings.

---

## 2️⃣ IAM User & Group Setup

| User | Group | Policy | Purpose |
|------|--------|--------|----------|
| `sysops-admin` | `AdminGroup` | `AdministratorAccess` | Full control for SysOps tasks |
| `trainee-readonly` | `ReadOnlyGroup` | `ReadOnlyAccess` | Limited visibility for experiments |

**Commands Used:**
```bash
aws iam create-group --group-name AdminGroup
aws iam attach-group-policy --group-name AdminGroup --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

aws iam create-user --user-name sysops-admin
aws iam add-user-to-group --user-name sysops-admin --group-name AdminGroup
```

```bash
aws cloudwatch describe-alarms --alarm-names "billing-alerts-alarm" --region us-east-1
```
