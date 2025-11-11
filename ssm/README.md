## 📘 AWS Systems Manager (SSM)
🧩 Overview

AWS Systems Manager (SSM) provides unified visibility and control of AWS infrastructure.
It allows remote management of EC2 instances and on-prem systems without SSH access, automates patching and configuration, and securely manages application parameters and secrets.

### ⚙️ Core Components

| Feature                             | Purpose                                                              |
| ----------------------------------- | -------------------------------------------------------------------- |
| **SSM Agent**                       | Installed on EC2, enables communication with SSM service             |
| **Managed Instances**               | EC2 systems registered and online under Systems Manager              |
| **Session Manager**                 | Connect to EC2 without SSH keys or inbound ports                     |
| **Run Command**                     | Execute shell commands/scripts remotely across one or many instances |
| **Parameter Store**                 | Securely store and retrieve configuration values or secrets          |
| **Tag-based Targeting**             | Run commands or automation on multiple instances based on tags       |
| **Automation Documents (SSM Docs)** | AWS or custom scripts for automated workflows                        |
| **Ansible Integration**             | Use `AWS-Apply                                                       |


## 🧩 Step-by-Step Labs
1️⃣ SSM Agent Setup

- Attached IAM role SSM-EC2-Role with AmazonSSMManagedInstanceCore policy.

- Verified SSM agent running:

```bash
sudo systemctl status amazon-ssm-agent
```
Confirmed instance registration:
```bash
aws ssm describe-instance-information --region us-east-1
```

## 2️⃣ Session Manager

- Connected via AWS Console → Systems Manager → Session Manager → Start Session

- Verified shell access (whoami, hostname, uptime).

- No SSH key or inbound rules required — secure tunnel via AWS.

## 3️⃣ Run Command

- Used AWS-RunShellScript document:
```bash
uptime
sudo systemctl restart httpd
```

- Tested conditional execution using run command:
```bash
sudo systemctl stop httpd && echo "Stopped" || echo "Failed"
```

- Ran commands on multiple instances using tags:
```bash
Key=Environment,Value=Dev
```

## 4️⃣ Parameter Store

- Created parameters:
```bash
aws ssm put-parameter --name "/myapp/db_host" --value "db.example.com" --type "String"
aws ssm put-parameter --name "/myapp/db_password" --value "MyPass@123" --type "SecureString"
```

- Retrieved inside EC2 (via SSM session):
```bash
DB_HOST=$(aws ssm get-parameter --name "/myapp/db_host" --query 'Parameter.Value' --output text)
DB_PASS=$(aws ssm get-parameter --name "/myapp/db_password" --with-decryption --query 'Parameter.Value' --output text)
```
- Integrated parameters inside SSM Run Command scripts for automation.

## 5️⃣ Ansible Integration

Understood AWS-ApplyAnsiblePlaybooks document for executing playbooks from:

- S3 bucket

- GitHub repository

- Local path

Example:
```bash
SourceType = S3
SourceInfo = {"path":"https://s3.amazonaws.com/mybucket/playbooks/site.yml"}
PlaybookFile = site.yml
```

## 🧠 Key Takeaways

- Manage EC2s without SSH (secure & auditable).

- Execute commands on hundreds of instances simultaneously.

- Use Parameter Store for configuration and secrets — fetched securely at runtime.

- Scale operations by tag-based targeting.

- Integrate Ansible, Patch Manager, and Automation Docs for full-stack management.

## 🔒 IAM & Security Notes
| Action            | Required Permission            |
| ----------------- | ------------------------------ |
| Use SSM features  | `AmazonSSMManagedInstanceCore` |
| Read parameters   | `ssm:GetParameter`             |
| Read SecureString | `ssm:GetParameter` + `         |


## ✅ 1️⃣ Base Permissions (Required for SSM)

Attach the AWS-managed policy to the ec2 through role:
```json
AmazonSSMManagedInstanceCore
```
This gives your EC2 instance permission to:

- Communicate with SSM endpoints.

- Execute Run Command, Session Manager, Patch Manager, etc.

- Upload logs and metadata to SSM backend.

## 🧩 2️⃣ Custom Policy — Access to Parameter Store

Attach a custom inline policy (or a new IAM policy) if your EC2 instance needs to read parameters (plain or encrypted) from Parameter Store.

🔸 Read Plaintext Parameters
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath",
        "ssm:DescribeParameters"
      ],
      "Resource": "arn:aws:ssm:us-east-1:785818313570:parameter/myapp/*"
    }
  ]
}
```

This allows the instance to read all parameters under the /myapp/ path.
✅ You can narrow this to `/prod/myapp/` or `/dev/myapp/` for least privilege.


## 🔐 3️⃣ Custom Policy — SecureString (KMS Decrypt)
If you use SecureString parameters (encrypted with KMS), you must grant permission to use that KMS key.

🔸 If you use AWS-managed KMS key:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:us-east-1:785818313570:key/*"
    }
  ]
}
```

🔸 If you use a Customer Managed KMS Key, replace the ARN:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDecryptForParameterStore",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:785818313570:key/1234abcd-12ab-34cd-56ef-1234567890ab"
    }
  ]
}
```

💡 Note:
The principal (EC2 instance role) must also be included in the KMS key policy like this:
```json
{
  "Sid": "AllowSSMInstanceRoleUseKey",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::785818313570:role/SSM-EC2-Role"
  },
  "Action": [
    "kms:Decrypt",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

🔸 Combined Example Policy

If you want one combined inline policy for both Parameter Store + KMS access:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath",
        "ssm:DescribeParameters"
      ],
      "Resource": "arn:aws:ssm:us-east-1:785818313570:parameter/myapp/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:785818313570:key/*"
    }
  ]
}
```

## 🧠 4️⃣ Validation Commands

Once attached, validate from your EC2 instance:
```bash
aws ssm get-parameter --name "/myapp/db_host" --query 'Parameter.Value' --output text
aws ssm get-parameter --name "/myapp/db_password" --with-decryption --query 'Parameter.Value' --output text
```
If permissions are correct, both return values successfully.
If not, you’ll see:
```bash
AccessDeniedException: User: arn:aws:sts::... is not authorized to perform: ssm:GetParameter
```

## ✅ 5️⃣ Security Recommendations

Use least privilege — restrict to parameter paths needed (/prod/myapp/ only).

Use customer-managed KMS keys for production secrets.

Include CloudWatch logging for all SSM activity (already configured during CloudWatch labs).

Rotate keys periodically.
