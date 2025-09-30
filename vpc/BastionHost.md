# AWS VPC – Private EC2, Bastion, NAT Gateway, Elastic IP, and ENI Explained

This document explains step by step why a private EC2 cannot reach the internet directly, how Bastion and NAT Gateway solve different problems, and what role ENI and Elastic IP play. Written in a beginner-friendly way.

---

## 1. Problem Statement
- You have:
    - **Bastion EC2** → deployed in **public subnets (1a, 1b)**.
    - **Private EC2** → deployed in **private subnets (1a, 1b)**.
- From Bastion, you can SSH into the Private EC2 (✅ works).
- But **pinging google.com from Private EC2 times out** (❌ internet not reachable).

---

## 2. Why This Happens
- Private EC2s do not have **direct internet access**.
- To give them outbound-only internet, you need a **NAT Gateway**.
- NAT Gateway must be in a **public subnet** (not private).

---

## 3. Fix – Adding NAT Gateway
1. **Create NAT Gateway**
    - Place in **public subnet (e.g., 1a)**.
    - Attach an **Elastic IP**.
    - For high availability, create one NAT Gateway per AZ.

2. **Update Private Subnet Route Tables**
    - Private subnet 1a → route `0.0.0.0/0 → NAT Gateway in 1a`.
    - Private subnet 1b → route `0.0.0.0/0 → NAT Gateway in 1b`.

3. **Check Security**
    - Private EC2 SG → allow outbound traffic (default allows all).
    - Bastion EC2 SG → allow inbound SSH.

---

## 4. Why NAT Gateway Must Be in a Public Subnet
- **Public subnet** = a subnet whose route table has a **route to the IGW (Internet Gateway)**.
- NAT Gateway needs IGW access to send traffic out to the internet.
- If you place NAT in a private subnet, it has no IGW route → internet access fails.

---

## 5. Why Elastic IP is Required for NAT Gateway
- Internet Gateway only understands **public IPs**.
- Private EC2s only have **private IPs (10.x.x.x, 172.16.x.x, 192.168.x.x)**.
- NAT Gateway provides a **public identity** (Elastic IP) so responses from the internet know where to go back.
- Flow:
    1. Private EC2 (10.0.2.15) → NAT Gateway.
    2. NAT rewrites source IP → Elastic IP (e.g., 52.x.x.x).
    3. IGW sends to internet.
    4. Response → Elastic IP → NAT → Private EC2.

---

## 6. Bastion Public IP vs NAT Elastic IP
| Feature                | Bastion Public IP                | NAT Gateway Elastic IP          |
|-------------------------|----------------------------------|---------------------------------|
| **Who uses it**        | Admins (SSH/RDP)                 | Private EC2s (automatic use)    |
| **Direction**          | Inbound (Internet → VPC)         | Outbound (VPC → Internet)       |
| **Subnet type**        | Public subnet                    | Public subnet                   |
| **Used for**           | SSH into VPC                     | Software updates, external APIs |
| **Identity visible**   | Bastion EC2 itself               | All private EC2s share this EIP |

👉 Analogy:
- Bastion = **office door** (lets you in).
- NAT = **office phone line** (lets you call out, but hides desk phones).

---

## 7. Does NAT Gateway Have Both Private IP and Elastic IP?
✅ Yes.
- **Private IP** → assigned from the subnet (e.g., 10.0.1.50).
- **Elastic IP** → public identity for internet.

### Flow:
1. Private EC2 → NAT’s **private IP**.
2. NAT → translates to its **Elastic IP**.
3. Internet → replies to Elastic IP.
4. NAT → translates back to EC2 private IP.

So NAT has **both legs**:
- One in the **private world (private IP)**.
- One in the **public world (Elastic IP)**.

---

## 8. Why NAT’s Private IP Isn’t Public, Even in a Public Subnet
- Subnet type doesn’t decide whether an IP is public or private.
- Subnet IPs are always **private (10.x, 172.16.x, 192.168.x)**.
- A **public IP (Elastic IP)** is a separate AWS-managed resource mapped to the ENI.
- So NAT Gateway in a public subnet still gets a **private IP** from subnet CIDR + an Elastic IP for internet.

---

## 9. What is ENI (Elastic Network Interface)?
- **ENI = AWS’s virtual network card**.
- Just like your laptop’s Wi-Fi card gives it an IP, ENI gives AWS resources (EC2, NAT, Load Balancer) their IPs.

### ENI properties:
- Has a **private IP** (from subnet).
- Can have a **public/Elastic IP** mapped.
- Linked to **security groups** and **routing**.

### NAT Gateway ENI example:
- Private IP = 10.0.1.50 (internal use).
- Elastic IP = 54.12.34.56 (public identity).
- Private EC2 talks to 10.0.1.50, internet sees 54.12.34.56.

---

## 10. Analogy Recap
- **Network Card (NIC)** in a laptop = gives it IP and lets it connect to router/internet.
- **ENI in AWS** = virtual NIC that gives EC2/NAT/etc. an IP inside a subnet.
- **Elastic IP** = like your SIM card’s phone number that outsiders can dial.
- **Bastion** = office door (inbound access).
- **NAT Gateway** = office phone line (outbound access).

---
