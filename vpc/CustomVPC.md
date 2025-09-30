# AWS Multi-AZ VPC Project – Bastion Host, NAT Gateway, Private Web App with ALB

This project demonstrates how to set up a secure, **highly available VPC architecture** in AWS with:

* **Public subnets** for Bastion host, NAT Gateway, and ALB.
* **Private subnets** for application EC2 instances.
* **Bastion host** to SSH into private EC2s securely.
* **NAT Gateway** to allow outbound internet access from private EC2s.
* **Application Load Balancer (ALB)** to expose a private web app.

---

## 1. Architecture Diagram

```
                  +---------------------+
                  |     Internet        |
                  +---------------------+
                           |
                           v
                  +---------------------+
                  |  Application LB     |
                  | (Public Subnets)    |
                  +---------------------+
                  |        |            |
                  v        v            v
       +----------------+    +----------------+
       |  Private EC2   |    |  Private EC2   |
       |  (AZ-1a)       |    |  (AZ-1b)       |
       +----------------+    +----------------+
                ^                    ^
                |                    |
                +--- NAT Gateway(s) -+
                | (Public Subnets)   |
                v
       +----------------+    
       |   Internet     |    
       +----------------+    

   Bastion Host (Public Subnet) → SSH into Private EC2s
```

---

## 2. Prerequisites

* AWS account
* IAM user with permissions: `EC2, VPC, ELB, IAM`
* AWS CLI (optional for automation)
* Key Pair for SSH access

---

## 3. Step-by-Step Implementation

### Step 1: Create a VPC

* Go to **VPC Console** → **Create VPC**.
* CIDR: `10.0.0.0/16`.

---

### Step 2: Create Subnets

* **Public Subnets**:

    * `10.0.1.0/24` in **AZ-1a**
    * `10.0.2.0/24` in **AZ-1b**
* **Private Subnets**:

    * `10.0.3.0/24` in **AZ-1a**
    * `10.0.4.0/24` in **AZ-1b**

Enable **Auto-assign Public IP** = **ON** for Public Subnets.

---

### Step 3: Create Internet Gateway (IGW)

* Create and attach an **Internet Gateway** to the VPC.
* Create **Public Route Table** → Add route:

    * `0.0.0.0/0 → IGW`.
* Associate this RT with **public subnets**.

---

### Step 4: Create NAT Gateways

* Allocate **Elastic IP** for each NAT Gateway.
* Create **NAT Gateway** in **each public subnet**.
* Create **Private Route Tables** → Add route:

    * `0.0.0.0/0 → NAT Gateway (of same AZ)`.
* Associate them with private subnets.

---

### Step 5: Launch Bastion Host

* Launch **EC2 (Amazon Linux/Ubuntu)** in **Public Subnet 1a**.
* Attach Elastic IP.
* Security Group:

    * Allow **SSH (22)** from **your IP only**.

---

### Step 6: Launch Private EC2 Instances (Web App)

* Launch **EC2 (Amazon Linux/Ubuntu)** in each private subnet (1a & 1b).
* User Data Script (installs Nginx sample app):

```bash
#!/bin/bash
sudo yum update -y
sudo amazon-linux-extras install nginx1 -y
echo "<h1>Hello from Private EC2 in $(hostname) </h1>" | sudo tee /usr/share/nginx/html/index.html
sudo systemctl enable nginx
sudo systemctl start nginx
```

* Security Group for Private EC2s:

    * Allow **HTTP (80)** from **ALB SG**.
    * Allow **SSH (22)** only from **Bastion Host SG**.

---

### Step 7: Configure Application Load Balancer (ALB)

* Create ALB in **Public Subnets (1a, 1b)**.
* ALB Security Group: Allow **HTTP (80)** from Internet.
* Create Target Group:

    * Type = EC2 Instances.
    * Register both private EC2s.
* Listener:

    * `HTTP : 80 → Target Group`.

---

### Step 8: Test the Setup

1. **SSH Access:**

    * SSH into Bastion Host → then into Private EC2 using private IP.
    * Verify Nginx is running (`curl localhost`).

2. **Outbound Internet from Private EC2s:**

    * From Bastion → SSH into private EC2.
    * Run: `ping google.com` (works via NAT).

3. **Web Application Access:**

    * From your browser → Open `http://<ALB-DNS-Name>`.
    * You should see:

      ```
      Hello from Private EC2 in ip-10-0-3-xx
      ```
    * Refresh to see traffic alternate between AZ-1a and AZ-1b EC2s.

---

## 4. Security Best Practices

* Restrict SSH access to Bastion only from your IP.
* Do not assign public IPs to private EC2s.
* Use IAM Roles for EC2 instead of hardcoding credentials.
* Enable ALB logging and CloudWatch monitoring.

---

## 5. Cleanup

* Delete ALB
* Delete EC2s (Private + Bastion)
* Delete NAT Gateways (to stop billing)
* Delete IGW and VPC

---

✅ Now you have a **multi-AZ private web app architecture** accessible via **ALB**, with **Bastion + NAT** for secure connectivity.

---

👉 Dharani, do you also want me to **write Terraform/CloudFormation code** for this setup so you can deploy it faster, or should I keep it only as manual console steps?
