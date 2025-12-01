# AWS PrivateLink (VPC Endpoint Services) – Full Hands‑On Lab

This document contains the **complete, step‑by‑step, detailed implementation** of:

* Provider VPC (Service Provider)
* Consumer VPC (Service Consumer)
* HTTP backend on private EC2
* NLB as service front-end
* Endpoint Service (PrivateLink)
* Interface Endpoint in Consumer
* DNS + testing

This is based on the full architecture we built manually.

---

# ## SECTION 1 — Provider VPC (Service Provider)

The Provider exposes a private service through **PrivateLink**.

---

# ### 1. Create Provider VPC

```
Name: provider-vpc
CIDR: 11.0.0.0/16
DNS Resolution: Enabled
DNS Hostnames: Enabled
```

---

# ### 2. Create Provider Subnets

### **Public Subnet**

```
Name: provider-public-a
CIDR: 11.0.1.0/24
AZ: us-east-1a
Auto-assign public IPv4: Enabled
```

### **Private Subnet**

```
Name: provider-private-a
CIDR: 11.0.3.0/24
AZ: us-east-1a
Auto-assign public IPv4: Disabled
```

---

# ### 3. Create and Attach Internet Gateway

```
Name: provider-igw
Attach to: provider-vpc
```

---

# ### 4. Create Public Route Table

```
Name: provider-public-rt
Route: 0.0.0.0/0 → IGW
Associate: provider-public-a
```

Private subnet has **no IGW route**.

---

# ### 5. Launch Provider Backend EC2 (Private Subnet)

```
AMI: Amazon Linux 2
type: t2.micro
Subnet: provider-private-a
Public IP: Disabled
SG: provider-backend-sg
```

### Security Group: **provider-backend-sg**

Inbound (initially temporary for testing):

```
TCP 8080 → Source: provider-vpc CIDR
```

Outbound:

```
All traffic → 0.0.0.0/0
```

---

# ### 6. Install HTTP Service on Provider Private EC2

Use **SSM Session Manager**.

Run:

```bash
sudo yum update -y
sudo yum install httpd -y
sudo systemctl enable httpd
sudo systemctl start httpd
```

Modify port to 8080:

```bash
sudo sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf
sudo systemctl restart httpd
```

Add webpage:

```bash
echo "<h1>Hello from Provider Private EC2</h1>" | sudo tee /var/www/html/index.html
```

Test:

```bash
curl localhost:8080
```

---

# ## SECTION 2 — Network Load Balancer (Provider)

The PrivateLink service **must** use a Network Load Balancer.

---

# ### 7. Create Target Group

```
Name: provider-tg
type: Instances
Protocol: TCP
Port: 8080
VPC: provider-vpc
```

Register:

* private EC2 → port 8080

---

# ### 8. Create Internal NLB

```
Name: provider-nlb
Scheme: Internal
IP Type: IPv4
Subnets: provider-public-a
```

Listener:

```
Protocol: TCP
Port: 8080
Forward to TG: provider-tg
```

---

# ### 9. Security Group for Backend

Create SG for traffic *from PrivateLink*.

```
Name: provider-nlb-sg
Inbound: TCP 8080 → Source: (added automatically later by VPC endpoint ENI)
Outbound: All → 0.0.0.0/0
```

Attach to **backend EC2**.

---

# ## SECTION 3 — Create Endpoint Service (Provider)

---

# ### 10. Create Endpoint Service

Navigate:

```
VPC → Endpoint Services → Create
```

Select:

```
Load Balancer: provider-nlb
Acceptance Required: Yes
Private DNS: Disabled (optional)
```

Service name generated:

```
com.amazonaws.vpce.us-east-1.vpce-svc-xxxxxxxx
```

Consumer VPC will use this.

---

# ## SECTION 4 — Consumer VPC

The Consumer connects to the service using **Interface Endpoint**.

---

# ### 11. Create Consumer VPC

```
Name: consumer-vpc
CIDR: 21.0.0.0/16
DNS Hostnames: Enabled
DNS Resolution: Enabled
```

---

# ### 12. Create Consumer Subnets

**Public Subnet**

```
Name: consumer-public-a
CIDR: 21.0.1.0/24
Auto Public IP: On
```

**Private Subnet**

```
Name: consumer-private-a
CIDR: 21.0.3.0/24
Auto Public IP: Off
```

---

# ### 13. Route Tables

### Public route table:

```
Name: consumer-public-rt
0.0.0.0/0 → IGW
Associate: consumer-public-a
```

### Private route table:

(NAT optional):

```
Name: consumer-private-rt
0.0.0.0/0 → NAT Gateway (optional)
Associate: consumer-private-a
```

---

# ### 14. Consumer Private EC2

```
Name: consumer-ec2
AMI: Amazon Linux 2
Subnet: consumer-private-a
Public IP: No
SG: consumer-sg
```

### consumer-sg

Inbound:

```
(Nothing required for inbound)
```

Outbound:

```
All → 0.0.0.0/0
```

---

# ## SECTION 5 — Interface Endpoint (Consumer)

This is what actually connects to the Provider’s Endpoint Service.

---

# ### 15. Create Interface Endpoint in Consumer VPC

Go to:

```
VPC → Endpoints → Create Endpoint
```

Choose:

```
Service category: Other endpoint services
Service Name: com.amazonaws.vpce.us-east-1.vpce-svc-xxxxxxxx
VPC: consumer-vpc
Subnets: consumer-private-a
Security Group: consumer-sg
Private DNS: Enabled
```

Create.

---

# ### 16. Provider Accepts Connection

Go to:

```
VPC → Endpoint Services → Connections
```

Click **Accept**.

Status becomes **Available**.

---

# ## SECTION 6 — Testing End‑to‑End

---

# ### 17. DNS Test From Consumer EC2

Inside consumer EC2:

```bash
dig <private-dns-name-of-endpoint>
```

Expected:

* DNS resolves to **interface endpoint ENI private IPs** (21.x.x.x range)

---

# ### 18. Service Access Test

Run:

```bash
curl http://<endpoint-dns-name>:8080
```

Expected Output:

```
<h1>Hello from Provider Private EC2</h1>
```

This confirms:

* PrivateLink routing works
* NLB routing works
* EC2 backend works
* All private, no internet

---

# ## END — Final Architecture Flow

```
Consumer EC2 (21.x)
      ↓ DNS → Interface Endpoint ENI (21.x)
      ↓ AWS PrivateLink
Provider NLB (11.x)
      ↓ Target Group
Provider Private EC2 (11.x:8080)
```

* No VPC Peering
* No IGW
* No NAT required for backend
* 100% private communication

---

# This completes the full VPC Endpoint + Endpoint Service (PrivateLink) Lab.

