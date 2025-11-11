## ☁️ 1️⃣ CloudTrail — “Who did what in AWS”
🔹 Purpose

CloudTrail tracks API calls and console actions made inside your AWS account — every action by users, roles, or services.
It’s the audit trail of your AWS environment.

Think of it as:
```bash
🧾 “Who did what, when, from where, and how.”
```

🔹 Examples of CloudTrail events
| Action                       | Example CloudTrail event |
| ---------------------------- | ------------------------ |
| EC2 instance launched        | `RunInstances`           |
| IAM policy modified          | `PutUserPolicy`          |
| S3 bucket deleted            | `DeleteBucket`           |
| User logged into AWS console | `ConsoleLogin`           |
| CloudWatch alarm created     | `PutMetricAlarm`         |


Each event records:
```json
{
  "eventTime": "2025-11-11T09:32:10Z",
  "eventSource": "ec2.amazonaws.com",
  "eventName": "RunInstances",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "sysops-admin"
  },
  "sourceIPAddress": "10.192.7.45"
}
```

🔹 Where CloudTrail stores data

By default:

- Events are stored in Event History (for 90 days, no setup required).

- You can enable a Trail to deliver events to:

- An S3 bucket (for long-term archival)

- A CloudWatch Log Group (for live monitoring & alerts)


🔹 Why CloudTrail matters

- Security & compliance audits

- Investigate unauthorized activity

- Detect configuration changes

- Monitor access patterns

- Trigger alarms on suspicious actions (e.g., root login, failed ConsoleLogin)

## 📊 2️⃣ CloudWatch Logs — “What happened inside systems and apps”

🔹 Purpose

CloudWatch Logs collects log files and application logs from your servers, containers, or AWS services.

Think of it as:
```bash
🪵 “What’s happening inside the machine or application.”
```

🔹 Examples of CloudWatch logs
| Source        | Example logs                                                        |
| ------------- | ------------------------------------------------------------------- |
| EC2           | `/var/log/messages`, `/var/log/syslog`, `/var/log/httpd/access.log` |
| Lambda        | Function execution logs (automatically sent)                        |
| VPC Flow Logs | Network traffic logs                                                |
| RDS           | Database error & slow query logs                                    |
| CloudTrail    | API audit events (if forwarding enabled)                            |


Each log group can have multiple streams (one per instance or application).

🔹 Key things CloudWatch Logs enables

- Centralized log storage from multiple sources

- Search, filter, and query logs (via Logs Insights)

- Create metric filters (e.g., count “ERROR” lines)

- Build dashboards or alarms on those metrics

- Send logs to Grafana or SIEM tools for analysis

## 🔄 3️⃣ How CloudTrail and CloudWatch Logs work together

When you enable CloudTrail logging to CloudWatch Logs, all CloudTrail events (API calls) start flowing into a log group like /aws/cloudtrail/AccountTrail.
Now you can:

- Run queries (Logs Insights) on AWS activity

- Create metric filters — e.g.:

 ConsoleLogin failures

  Root account logins

  IAM policy changes

- Visualize those metrics in Grafana or set alarms

👉 So CloudTrail = AWS actions,
👉 CloudWatch Logs = System & app events,
and together they give you full observability + auditability.
