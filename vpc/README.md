# AWS VPC, Peering, DNS, and Route53 Resolver – Complete Deep Dive

(Work in progress — we will fill each section step‑by‑step)

---

## 1. **VPC Fundamentals (Step-by-Step Creation Guide)**

Below is the **complete practical workflow** we followed to build a fully functional VPC networking setup.

---

## **1.1 Create a VPC**

### **Steps:**

1. Go to **VPC Console → Create VPC**
2. Choose **VPC only**
3. Provide CIDR:

   ```
   10.0.0.0/16
   ```
4. Enable:

   * DNS Resolution = ✔ On
   * DNS Hostnames = ✔ On
5. Name the VPC:

   ```
   main-vpc
   ```

### **Explanation:**

* VPC is your private network boundary inside AWS.
* CIDR determines maximum IPs.
* DNS support is required for private DNS, EC2 hostnames, and Route53.

---

## **1.2 Create Public Subnets**

### **Steps:**

1. Go to **Subnets → Create subnet**
2. Select **main-vpc**
3. Add two subnets:

   ```
   public-a → 10.0.1.0/24 (us-east-1a)
   public-b → 10.0.2.0/24 (us-east-1b)
   ```
4. Enable:

   * Auto-assign public IP = ✔ Yes

### **Explanation:**

* Public subnets allow EC2 to have public IPs to access the internet.
* They must be connected to an Internet Gateway.

---

## **1.3 Create Private Subnets**

### **Steps:**

1. Create:

   ```
   private-a → 10.0.10.0/24 (us-east-1a)
   private-b → 10.0.11.0/24 (us-east-1b)
   ```
2. Auto-assign public IP = ❌ No

### **Explanation:**

* Private subnets cannot access the internet directly.
* Must use NAT Gateway for outgoing traffic.

---

## **1.4 Create and Attach an Internet Gateway (IGW)**

### **Steps:**

1. **IGW → Create**
2. Attach to `main-vpc`
3. Name: `main-igw`

### **Explanation:**

* IGW provides internet to public subnets.
* Required for EC2 with public IP.

---

## **1.5 Create Public Route Table**

### **Steps:**

1. Create `public-rt`
2. Add route:

   ```
   0.0.0.0/0 → Internet Gateway
   ```
3. Associate to subnets:

   * public-a
   * public-b

### **Explanation:**

* Outbound traffic goes to the internet.
* Makes subnets *public*.

---

## **1.6 Create NAT Gateway**

### **Steps:**

1. Allocate Elastic IP → `nat-eip`
2. Create NAT Gateway in **public-a** subnet.
3. Name: `nat-gw`

### **Explanation:**

* Private subnets use NAT for outbound internet.
* NAT must be inside a public subnet.

---

## **1.7 Create Private Route Table**

### **Steps:**

1. Create `private-rt`
2. Add route:

   ```
   0.0.0.0/0 → NAT Gateway
   ```
3. Associate with subnets:

   * private-a
   * private-b

### **Explanation:**

* Makes private subnets able to reach the internet **without being reachable from internet**.

---

## **1.8 Security Groups (SG)**

### **Created:**

#### `public-sg`

* Allow SSH (22) from 0.0.0.0/0
* Allow HTTP (80)

#### `private-sg`

* Allow SSH **only from public subnet CIDRs**
* All outbound allowed

### **Explanation:**

* SGs are stateful firewalls applied to EC2.
* Required to control traffic per instance.

---

## **1.9 NACLs (Optional)**

### Public NACL:

* Inbound: Allow all
* Outbound: Allow all

### Private NACL:

* Inbound: Allow all
* Outbound: Allow all

### **Explanation:**

* NACLs are subnet-level stateless firewalls.
* Good for blocking a CIDR entirely.

---

## **2. VPC Peering Setup**

### **Steps:**

1. Go to **VPC Peering → Create Peering Connection**
2. Select:

   * Requester: VPC-A
   * Accepter: VPC-B
3. Accept the connection in VPC-B.
4. Add routes:

### In VPC-A route table:

```
10.1.0.0/16 → pcx-xxxx
```

### In VPC-B route table:

```
10.0.0.0/16 → pcx-xxxx
```

5. Update SGs:

* VPC-A SG allow inbound from VPC-B CIDR
* VPC-B SG allow inbound from VPC-A CIDR

### **Explanation:**

* Peering connects two VPCs using private IP.
* Required for cross-VPC communication.

---

## **3. Private Hosted Zones (PHZ) Setup**

### **Steps:**

1. Route53 → Create Hosted Zone
2. Type: **Private Hosted Zone**
3. Domain:

```
dev.internal
```

4. Associate with VPC-A only.

5. Create record:

```
app.dev.internal → 10.0.10.50
```

### **Test inside VPC-A:**

```
dig app.dev.internal → SUCCESS
```

### **Test inside VPC-B:**

```
dig app.dev.internal → FAIL
```

### **Explanation:**

PHZs do NOT cross VPC peering.
We need Route53 Resolver.

---

## **4. Route53 Resolver Endpoints (Cross-VPC DNS)**

---

## **4.1 Inbound Endpoint (VPC-A)**

### Steps:

1. Route53 Resolver → Inbound Endpoint → Create
2. VPC: VPC-A
3. Subnet: private-a
4. Security Group must allow:

```
TCP/UDP 53 FROM VPC-B CIDR
```

This acts as DNS **server**.

---

## **4.2 Outbound Endpoint (VPC-B)**

### Steps:

1. Route53 Resolver → Outbound Endpoint → Create
2. VPC: VPC-B
3. Subnet: private-b
4. SG rules:

   * Inbound: allow TCP/UDP 53 from VPC-B CIDR
   * Outbound: allow ALL

This acts as DNS **client/forwarder**.

---

## **4.3 Create Forwarding Rule**

### Steps:

1. Route53 Resolver → Rules → Create rule
2. Rule Type: **Forward**
3. Domain:

```
dev.internal
```

4. Forward to inbound endpoint IPs.
5. Associate the rule to **VPC-B**.

---

## **4.4 Final Testing**

### From VPC-B private EC2:

```
dig app.dev.internal → SUCCESS
```

### Explanation:

* DNS request goes to outbound endpoint.
* Forwarded to inbound endpoint.
* Resolved by PHZ in VPC-A.
* Response returned to VPC-B EC2.

This achieves cross-VPC PHZ resolution.

---

## **End of extended section****

### **What is a VPC?**

* A logically isolated network inside AWS.
* You control IP ranges, routing, subnets, and networking.

### **Key components**

* **VPC CIDR** – Base IP range (example: `10.0.0.0/16`).
* **Subnets** – Smaller network segments inside a VPC.

  * Public subnet → Connected to Internet Gateway
  * Private subnet → Uses NAT Gateway for outbound internet
* **Route Tables** – Define where traffic goes.
* **Internet Gateway (IGW)** – Enables internet access for public subnets.
* **NAT Gateway** – Allows private subnet instances to reach the internet.
* **NACL** – Stateless, subnet-level firewall.
* **Security Groups** – Stateful instance-level firewall.

---

## 2. **Public vs Private Subnets**

### **Public Subnet**

* Route table has: `0.0.0.0/0 → IGW`.
* EC2s can have public IPs.

### **Private Subnet**

* No direct internet route.
* Uses NAT Gateway inside a public subnet for outbound-only connectivity.

---

## 3. **Security Groups vs NACLs**

### **Security Groups (SG)**

* Stateful
* Attached to ENI/EC2
* Only allow rules

### **Network ACLs (NACL)**

* Stateless
* Attached to subnets
* Allow and deny rules
* Rules processed in order (lowest rule number first).

### Example test understanding:

* If inbound ephemeral ports are DENIED → SSH breaks.

---

## 4. **VPC Peering**

### **What is VPC Peering?**

* Connects two VPCs privately using AWS backbone.
* Supports cross-VPC communication using **private IPs only**.

### **Requirements**

* Non-overlapping CIDRs.
* Routes added manually in both VPCs.
* SGs must allow traffic from peer VPC CIDR.

### **What works across peering?**

✔ ICMP (ping)
✔ SSH
✔ HTTP/HTTPS
✔ EC2 internal DNS (`ip-10-0-1-23.ec2.internal`)

### **What does NOT work across peering?**

❌ Security-group referencing between VPCs
❌ Transitive routing (VPC-A → VPC-B → VPC-C)
❌ Route53 private zones (needs Resolver)

---

## 5. **Private ↔ Private EC2 Connectivity Across VPC Peering**

### Steps performed:

1. Two VPCs created: VPC-A and VPC-B.
2. Private subnets in both.
3. Private EC2s launched.
4. Public EC2 used as a bastion host.
5. SGs updated:

   * VPC-A SG inbound allows `10.1.0.0/16`.
   * VPC-B SG inbound allows `10.0.0.0/16`.
6. Private EC2 in A could ping + SSH private EC2 in B.

This validated:

* Route tables are correct
* Peering is active
* SGs allow cross-VPC traffic

---

## 6. **EC2 Internal DNS Across VPC Peering**

### Important fact:

AWS internal DNS names like:

```
ip-10-0-1-53.ec2.internal
```

**WORK across VPC peering by default**, as long as:

* `enableDnsSupport = true`
* `enableDnsHostnames = true`

No extra settings needed.

---

## 7. **Private Hosted Zones (PHZ)**

### What is a PHZ?

A private DNS zone that is visible ONLY inside its associated VPC.
Example:

```
dev.internal
```

### What we did:

1. Created PHZ in VPC-A: `dev.internal`
2. Added record: `app.dev.internal → 10.0.1.50` (private EC2 in VPC-A)
3. Tested:

   * Works inside VPC-A ✔
   * Fails inside VPC-B ❌

### Why?

Private Hosted Zones **do NOT cross VPC peering**.
This is an AWS limitation.

---

## 8. **Route53 Resolver – Cross‑VPC DNS Architecture**

To fix the PHZ limitation, we built enterprise-grade DNS:

### Components created:

✔ **Inbound endpoint (VPC-A)** – acts like DNS server
✔ **Outbound endpoint (VPC-B)** – acts like DNS client
✔ **Forwarding rule** – `dev.internal → inbound endpoints`
✔ **Rule association with VPC-B**

### Security Groups created:

**Inbound SG (VPC-A):**

* Allow TCP/UDP 53 from VPC-B CIDR

**Outbound SG (VPC-B):**

* Allow inbound TCP/UDP 53 from VPC-B
* Outbound: allow all (or TCP/UDP 53 to inbound endpoint IPs)

### Result:

From private EC2 in VPC-B:

```
dig app.dev.internal  → SUCCESS
```

Now VPC-B can resolve private records in VPC-A.

This is how enterprises implement:

* Shared Services DNS
* Hybrid DNS
* Centralized Route53 DNS

---

## 9. **Why This Architecture Is Needed**

Used when:

* Multiple VPCs must reach shared services.
* Microservices spread across VPCs.
* Private DNS required across accounts.
* Hybrid (on-prem ↔ AWS) DNS.
* Need DNS without exposing servers to internet.

DNS forwarding ≠ connectivity.
You still need proper routing + SG.

---

## 10. **Key Takeaways**

* EC2 internal DNS works over peering.
* Private Hosted Zones DO NOT cross VPCs.
* Route53 Resolver is required for cross-VPC PHZ resolution.
* Inbound endpoint = DNS server.
* Outbound endpoint = DNS forwarder.
* SGs must allow TCP/UDP 53.
* Perfect flow:

  * VPC-B EC2 → Outbound endpoint → Inbound endpoint → PHZ in VPC-A.

---

## 11. AWS VPC Flow Logs – Deep Dive & Analysis

VPC Flow Logs capture network traffic metadata (no packet payload) flowing through:

- VPC

- Subnets

- ENIs (network interfaces)

They are essential for troubleshooting, auditing, and real-world debugging.

### 12. What VPC Flow Logs Capture

Each log entry shows:

- Source IP

- Destination IP

- Ports

- Protocol

- Packets & bytes

- ACCEPT or REJECT

- Whether the flow was allowed or denied

- Flow Logs do not capture:

- Packet contents

- TLS payload

- Application-level data

### 13. Flow Log Levels

AWS flow logs can be enabled at:

#### 13.1 **VPC Level**

Captures traffic for the entire VPC (all subnets + ENIs).

#### 13.2 **Subnet Level**

Captures traffic inside a specific subnet.

#### 13.3 **ENI Level**

Captures traffic for a single EC2 instance, Lambda ENI, ECS task, or EKS pod ENI.

Most precise and commonly used for debugging.

### 14. Creating VPC Flow Logs
Step-by-Step:

1. Go to **VPC Console → Your VPC → Flow Logs → Create Flow Log**


2. Choose Resource:
✔ Entire VPC

✔ A specific Subnet

✔ A specific ENI



3. Choose filter:

- `ALL` (best for debugging)

- `ACCEPT`

- `REJECT`

4. Choose Destination:

- CloudWatch Log Group

- S3 bucket

5. IAM role auto-created

6. Click Create

Flow logs populate within ~2 minutes.

### 15. Flow Log Record Format

Example Flow Log line:
```bash
2 785818313570 eni-0f755f4799146a4dd 35.203.210.229 10.0.1.93 52714 52977 6 1 44 1764066773 1764066781 ACCEPT OK
```

| Field               | Description                  |
| ------------------- | ---------------------------- |
| 2                   | Log version                  |
| 785818313570        | AWS Account ID               |
| eni-0f75…           | Network interface ID         |
| 35.203.210.229      | Source IP                    |
| 10.0.1.93           | Destination IP               |
| 52714               | Source Port (ephemeral)      |
| 52977               | Destination Port             |
| 6                   | Protocol (TCP = 6, UDP = 17) |
| 1                   | Packets                      |
| 44                  | Bytes                        |
| timestamps          | Start & end of flow          |
| **ACCEPT / REJECT** | Allowed or denied            |
| OK                  | Log status                   |


### 16. How to Analyze Flow Logs

#### 16.1 Scenario: SSH Fails Because of NACL

Flow Log shows:
```
10.0.1.10  → 10.0.2.10 22 REJECT
```

Cause:
NACL outbound ephemeral ports not allowed.

Fix:
```
Allow 1024–65535 outbound
Allow 22 inbound
```

#### 16.2 Scenario: Private EC2 Cannot Reach the Internet

Flow Log:
```
SRC=10.0.10.50 DST=1.1.1.1 ACTION=REJECT
```

Possible reasons:

- NAT Gateway missing

- NAT in wrong AZ

- Private RT missing NAT route

- NACL denies outbound


