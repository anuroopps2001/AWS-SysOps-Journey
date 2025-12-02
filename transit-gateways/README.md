# AWS Transit Gateway (TGW) – Architecture & Implementation (Hands‑On Guide)

This document describes the **exact TGW architecture** we built in the lab: three VPCs connected through a Transit Gateway, routing configured for full communication, and EC2 instances used for connectivity tests.

The goal of this `.md` file is to capture everything — components, steps, diagrams (described), commands, and validation.

---

# 1. **Architecture Overview**

We deployed the following components:

### ✔ Three VPCs (A, B, C)

Each VPC contains:

* A CIDR block
* Two public subnets (for simplicity)
* One EC2 instance in a public subnet

### ✔ A single AWS Transit Gateway (TGW)

The TGW acts as a centralized router.

### ✔ Three VPC Attachments

Each VPC is connected to the TGW using a **VPC attachment**.

### ✔ TGW Route Table

We used the **default TGW route table**, and configured:

* ASSOCIATION: Each VPC is associated with the TGW RT
* PROPAGATION: Each VPC propagates its CIDR to the RT

### ✔ Routing

* Each VPC route table has a route toward other VPC CIDRs via the TGW attachment.
* TGW route table has routes pointing to attachments.

### ✔ EC2 instances

We deployed **one EC2 instance in each VPC** (public subnet) for ping/SSH testing.

### ✔ Security Groups

Each EC2 SG allowed:

* Inbound: ICMP + SSH from anywhere for testing
* Outbound: All

---

# 2. **Final Architecture Diagram (Text Representation)**

```
                     ┌──────────────────────────┐
                     │      Transit Gateway     │
                     │        (TGW)             │
                     └────────────┬─────────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         │                        │                        │
┌────────▼────────┐     ┌─────────▼────────┐     ┌─────────▼────────┐
│    VPC‑A         │     │     VPC‑B        │     │     VPC‑C        │
│ CIDR: 10.0.0.0/16│     │ CIDR: 10.1.0.0/16│     │ CIDR: 10.2.0.0/16│
│  EC2: 10.0.1.10  │     │  EC2: 10.1.1.10  │     │  EC2: 10.2.1.10  │
└──────────────────┘     └───────────────────┘     └───────────────────┘
```

---

# 3. **Step‑by‑Step Implementation**

Below is *exactly* what we configured.

---

## 3.1 **Create Three VPCs**

### Example CIDRs:

* **VPC‑A:** `10.0.0.0/16`
* **VPC‑B:** `10.1.0.0/16`
* **VPC‑C:** `10.2.0.0/16`

### Inside each VPC:

* Public Subnet A (AZ‑1a)
* Public Subnet B (AZ‑1b)
* Auto‑assign public IP: enabled

---

## 3.2 **Create EC2 Instances**

For each VPC:

* Launch Amazon Linux 2 EC2 in **public subnet‑a**.
* Security Group:

  * Inbound:

    * ICMP (Ping): `0.0.0.0/0` (lab only)
    * SSH (22): `0.0.0.0/0` (lab only)
  * Outbound: All

---

## 3.3 **Create Transit Gateway**

Go to: **VPC → Transit Gateways → Create**

* Name: `lab-tgw`
* ASN: 64512 (default)
* Default Route Table Association: ON
* Default Route Table Propagation: ON

This creates:

* TGW
* One default TGW route table

---

## 3.4 **Create TGW Attachments (One per VPC)**
“A TGW attachment is the doorway through which the VPC communicates with the Transit Gateway.”

Without the attachment, the VPC cannot send or receive traffic through the TGW.
For each VPC:

1. **Transit Gateway Attachments → Create**
2. Select:

   * TGW: `lab-tgw`
   * VPC: (A, B, or C)
   * Subnets: choose 1 public subnet per AZ (minimum 1)

This produces three attachments:

* `tgw-attach-a`
* `tgw-attach-b`
* `tgw-attach-c`

---

## 3.5 **Configure TGW Route Table**
**From the TGW attachment listed under Association, you can reach the TGW attachments listed under Propagation.**

Go to: **Transit Gateway Route Tables → default → Routes**

### For each VPC attachment

The TGW automatically learns propagated CIDRs only if propagation = enabled.

We ensured:

* **Propagation: ENABLED** for all 3 attachments
* **Association: ENABLED** for all 3 attachments

So the TGW route table contains:

```
10.0.0.0/16 → tgw-attach-a
10.1.0.0/16 → tgw-attach-b
10.2.0.0/16 → tgw-attach-c
```

---

## 3.6 **Configure VPC Route Tables**

Inside each **VPC's public route table**, add routes:

### On VPC‑A Route Table

```
10.1.0.0/16 → tgw-attach-a
10.2.0.0/16 → tgw-attach-a
```

### On VPC‑B Route Table

```
10.0.0.0/16 → tgw-attach-b
10.2.0.0/16 → tgw-attach-b
```

### On VPC‑C Route Table

```
10.0.0.0/16 → tgw-attach-c
10.1.0.0/16 → tgw-attach-c
```

This allows traffic to exit a VPC toward TGW.

---

# 4. **Testing the Connectivity**

From each EC2:

### Ping VPC‑A → VPC‑B

```
ping 10.1.1.10
```

### Ping VPC‑A → VPC‑C

```
ping 10.2.1.10
```

### Ping VPC‑B → VPC‑C

```
ping 10.2.1.10
```

All should succeed.

### SSH tests (if required):

```
ssh ec2-user@10.1.1.10
```

If ICMP works but SSH fails → SG issue.

---

# 5. **Understanding TGW Concepts (Mapped to Lab)**

### **Transit Gateway (TGW)**

Central router that connects multiple VPCs.

### **Attachment**

Connection between a VPC and the TGW.

### **Association**

Identifies **which route table** should be used for incoming traffic from this VPC.

### **Propagation**

Allows the VPC CIDR to be **added to the TGW route table**.

### **Result in our lab**

All VPCs share the same TGW route table.

So traffic can freely move between A ↔ B ↔ C.

---

# 6. **Why Everything Worked (End‑to‑End Explanation)**

### ✔ Non‑overlapping CIDRs

TGW requires unique CIDRs.

### ✔ Route tables correctly configured

Each VPC knew where other VPC CIDRs were reachable.

### ✔ SGs allowed ICMP

Without SG rules, ping would fail even with correct routing.

### ✔ TGW route table contained propagated routes

This enabled TGW to forward packets.

---

# 7. **Summary of the Architecture**

* Three VPCs
* Three EC2 instances
* One TGW
* Three TGW attachments
* Single TGW route table
* All attachments associated + propagated
* All VPC route tables updated
* Full mesh connectivity achieved

---

## ⭐ Goal Architecture

### we will build this:
```bash
                   +----------------------+
                   |  Shared Services VPC |  (VPC-C)
                   |  (Logging, AD, SSM)  |
                   +----------+-----------+
                              |
                              | TGW
                              |
+-------------------+     +---+-----+     +------------------+
|    VPC-A (App)    |-----|  TGW    |-----|  VPC-B (DB)      |
+-------------------+     +---------+     +------------------+
```

### Rules:

- App ↔ Shared Services = ALLOWED

- DB ↔ Shared Services = ALLOWED

- App ↛ DB = DENIED

- DB ↛ App = DENIED

**This is the whole point of Shared Services architecture.**

### ⭐ Step 1 — Confirm VPC Setup
```bash
# VPC-A (App)
10.0.0.0/16
```

```bash
# VPC-B (DB)
10.1.0.0/16
```
```bash
# VPC-C (Shared-Services)
10.2.0.0/16
```

EC2s exist in each VPC (public for now, fine).

So we go directly to TGW configuration.

### ⭐ Step 2 — Create TWO Transit Gateway Route Tables
Go to:

#### VPC → Transit Gateway Route Tables → Create

1️⃣  `tgw-rt-shared`

Used by Shared Services VPC
Allow all incoming traffic
Return traffic allowed to only required VPCs

2️⃣ `tgw-rt-app-db`

Used by both App and DB VPCs
Allow traffic ONLY towards shared services
Block App ↔ DB cross-talk via routing

### ⭐ Step 3 — Attach TGW Attachments to Correct Route Tables
Our attachments:

- tgw-attach-app (VPC-A)

- tgw-attach-db (VPC-B)

- tgw-attach-ss (VPC-C)

We will associate each to the correct TGW route table.

#### 3.1 — Associate App VPC to tgw-rt-app-db

TGW → Route Tables → `tgw-rt-app-db` → **Associations**
```bash
Attach: tgw-attach-app
```

#### 3.2 — Associate DB VPC to the same route table
```bash
Attach: tgw-attach-db
```

#### 3.3 — Associate Shared Services VPC to `tgw-rt-shared`
```bash
Attach: tgw-attach-ss
```

### ⭐ Step 4 — Configure Propagation Rules
### 🔵 In `tgw-rt-app-db` (App+DB table):

➡ ADD PROPAGATION from:

Shared Services attachment (`tgw-attach-ss`)

➡ REMOVE propagation from:

- App attachment

- DB attachment

This ensures:

- App → Shared = Allowed

- DB → Shared = Allowed

- App → DB = ❌ Not allowed

- DB → App = ❌ Not allowed

### 🟣 In `tgw-rt-shared` (Shared services table):

➡ ADD PROPAGATION from:

- App attachment

- DB attachment

This ensures:

- Shared → App = Allowed

- Shared → DB = Allowed

### ⭐ Step 5 — Update VPC Route Tables

Each **public route table** must send traffic to TGW only where required.
#### 🔵 VPC-A Route Table
```bash
10.2.0.0/16 → tgw-attach-app   (allowed)
10.1.0.0/16 → NO ROUTE        (block)
```
#### 🟢 VPC-B Route Table
```bash
10.2.0.0/16 → tgw-attach-db   (allowed)
10.0.0.0/16 → NO ROUTE        (block)
```

#### 🟣 VPC-C Route Table
```bash
10.0.0.0/16 → tgw-attach-ss   (allowed)
10.1.0.0/16 → tgw-attach-ss   (allowed)
```

### ⭐ Step 6 — TESTING
#### ✔ Test 1 — App → Shared Services
```bash
ping 10.2.x.x → MUST WORK

```

#### ✔ Test 2 — DB → Shared Services
```bash
ping 10.2.x.x → MUST WORK

```

#### ❌ Test 3 — App → DB
```bash
ping 10.1.x.x → MUST FAIL

```


#### ❌ Test 4 — DB → App
```bash
ping 10.0.x.x → MUST FAIL

```


#### ✔ Test 5 — Shared Services → App
```bash
ping 10.0.x.x → MUST WORK
```

#### ✔ Test 6 — Shared Services → DB
```bash
ping 10.1.x.x → MUST WORK
```


#### ✔ Test 6 — Shared Services → DB