# 🔐 Day 02 — IAM Groups, Policies & Roles

**Date:** YYYY-MM-DD  
**Goal:** Implement IAM best practices — manage access with groups, attach managed and custom policies, and use roles for EC2 service permissions.

---

## 1️⃣ Groups and Managed Policies

| Group | Policy | Purpose |
|--------|---------|----------|
| `AdminGroup` | `AdministratorAccess` | Full administrative control |
| `ReadOnlyGroup` | `ReadOnlyAccess` | View-only access for monitoring or audit users |

**Commands Used:**
```bash
aws iam create-group --group-name AdminGroup
aws iam create-group --group-name ReadOnlyGroup
aws iam attach-group-policy --group-name AdminGroup --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam attach-group-policy --group-name ReadOnlyGroup --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
aws iam list-attached-group-policies --group-name ReadOnlyGroup
```

### Custom Policy — Least-Privilege S3 Access
Created a custom policy to allow an EC2 instance or user to access only one S3 bucket with limited actions.

```bash
aws iam create-policy --policy-name S3RestrictedAccess --policy-document file://s3-restricted-policy.json
aws iam attach-group-policy --group-name ReadOnlyGroup --policy-arn arn:aws:iam::<account-id>:policy/S3RestrictedAccess
```

✅ Confirmed policy attachment via aws iam list-attached-group-policies.


### IAM Role for EC2 → S3 Access
Use case: Allow EC2 instances to access specific S3 buckets without using access keys.

```bash
aws iam create-role --role-name EC2S3AccessRole --assume-role-policy-document file://ec2-trust.json
aws iam attach-role-policy --role-name EC2S3AccessRole --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

✅ Verified trust relationship with EC2 service and permissions via get-role and list-attached-role-policies.

### IAM Roles — Quick Notes
Definition:
An IAM Role is a temporary identity in AWS with a set of permissions that can be assumed by trusted entities (like EC2, Lambda, users, or other AWS accounts).
Roles have no username or password, and no long-term access keys — they rely on temporary security credentials from AWS STS.


| Concept              | Explanation                                                           |
| -------------------- | --------------------------------------------------------------------- |
| **Who uses it**      | AWS services (EC2, Lambda), IAM users, or external entities (via STS) |
| **Purpose**          | Grant permissions *without* sharing long-term credentials             |
| **Credentials**      | Temporary (AccessKeyId, SecretKey, SessionToken) via STS              |
| **Attached Policy**  | Defines *what actions* and *on which resources* the role can perform  |
| **AssumeRole**       | The act of “taking on” the role — generates temporary credentials     |
| **Instance Profile** | Container that lets an EC2 instance use a role                        |



🧰 Common Use Cases

- EC2 accessing S3 or DynamoDB securely

- Lambda function writing to CloudWatch Logs

- Cross-account access (account A assumes a role in account B)

- Federated users via SSO or AWS Organizations

🤝 Trust Relationship (Assume Role Policy)

The trust policy defines who is allowed to assume the role — essentially, “who can wear this role’s hat.”

It’s attached inside the role and separate from its permission policy.

Example (EC2 Role Trust Policy):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Explanation:

`"Principal"` = the entity trusted to assume the role

`"Action"`: "sts:AssumeRole" = the operation allowing it

`"Service"`: "ec2.amazonaws.com" = means any EC2 instance can assume it



🔐 Two-Part Rule
| Policy Type           | Defines                                    |
| --------------------- | ------------------------------------------ |
| **Trust Policy**      | *Who* can assume the role                  |
| **Permission Policy** | *What* the role can do after being assumed |

✅ Both must allow the action for access to succeed.

🧭 Quick Example Flow (EC2 → S3 Access)

EC2 instance starts with a role attached.

EC2’s metadata service retrieves temporary credentials from STS.

Role’s trust policy allows EC2 service → sts:AssumeRole.

Role’s permission policy allows s3:GetObject, s3:PutObject.

Instance uses those temporary creds to access S3 — no keys stored.

```bash
aws iam get-role --role-name EC2S3AccessRole
aws iam get-role-policy --role-name EC2S3AccessRole --policy-name S3RestrictedAccess
```
