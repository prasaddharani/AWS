# AWS Networking & EC2 Practical Labs 🚀

---

## 🔹 1. Launch EC2 with User Data (Web Server Setup)

### Concept

* **User Data** is a script that runs at the first boot of an EC2 instance.
* We can use it to automatically install packages and configure applications (e.g., web server).
* Uses **EC2 Metadata Service (IMDSv2)** to fetch instance details (Private IP, Public IP, etc.).

### Practical Lab (via Console)

1. Go to **EC2 → Launch Instance**.
2. Choose **Amazon Linux 2 AMI**.
3. Select instance type (t2.micro for free tier).
4. Use default VPC/subnet.
5. Add **Key Pair** for SSH access.
6. Under **User Data** (Advanced details), paste:

```bash
#!/bin/bash
yum update -y
yum install -y httpd

systemctl start httpd
systemctl enable httpd

# Get IMDSv2 token
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Fetch Public IP
PUBLIC_IP=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/public-ipv4)

# Update index.html
echo "<h1>Hello from EC2 Instance with Public IP: $PUBLIC_IP</h1>" > /var/www/html/index.html
```

7. Launch instance.
8. In browser → `http://<EC2-Public-IP>` → You should see:

   ```
   Hello from EC2 Instance with Public IP: <your-ip>
   ```

---

## 🔹 2. Security Groups (SG) vs NACL

### Concept

* **Security Group (SG):** Stateful firewall at instance level.

    * If you allow inbound HTTP (80), return traffic is automatically allowed.
* **NACL (Network ACL):** Stateless firewall at subnet level.

    * Must allow both inbound and outbound rules for a flow to succeed.

### Practical Lab

1. Create EC2 in default VPC.
2. Attach a Security Group:

    * Inbound → allow **SSH (22)**, **HTTP (80)**.
3. Access web page from browser → works ✅
4. Now test NACL:

    * Edit **Outbound Rules** → Deny port 80 → Refresh browser.
    * Request fails (because NACL is stateless).
    * Re-allow port 80 → Works again.

---

## 🔹 3. Application Load Balancer (ALB) with Private EC2s

### Concept

* **ALB** distributes HTTP/HTTPS traffic across instances.
* Best practice: **EC2 instances stay private**, only ALB is public.
* Bastion host is only for **SSH access**, not for web traffic.

### Practical Lab

1. Create **two private EC2s** (in private subnets, no public IP).
   Install Apache with user data (similar to Step 1).
2. Create **ALB**:

    * Public Subnets (e.g., 1a, 1b).
    * Security Group: allow HTTP (80) from `0.0.0.0/0`.
3. Create **Target Group**:

    * Type: **Instance**.
    * Register your two **private EC2s**.
    * Health check: HTTP on `/`.
4. Create **Listener Rule**:

    * Forward port 80 → Target Group.
5. Test: Visit `http://<ALB-DNS-Name>` → You’ll see the EC2 response.

---

## 🔹 4. Instance Metadata Service (IMDSv2)

### Concept

* Used by EC2 to fetch metadata (IP, instance ID, region, etc.).
* **IMDSv2** requires a session token for security.

### Practical Lab

From EC2 SSH:

```bash
# Get Token
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Fetch Private IP
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/local-ipv4

# Fetch Public IP
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/public-ipv4
```

---

## 🔹 5. Bastion Host

### Concept

* Public EC2 instance used to SSH into private EC2s.
* You **don’t** route web traffic through Bastion → ALB connects directly to private EC2s.

### Practical Lab

1. Launch Bastion in Public Subnet.
2. SSH into Bastion → then SSH into Private EC2 using private IP.
3. Test connectivity (`curl <private-ec2-private-ip>`).

---

## 🔹 Key Learnings

* **User Data** automates setup at launch.
* **IMDSv2** is required to safely fetch metadata.
* **SG = Stateful, NACL = Stateless**.
* **ALB should directly talk to private EC2s** (not via Bastion).
* **Bastion Host = SSH only**.
* **Blocking outbound in NACL** shows stateless behavior.

---

✅ With these steps, you can now:

* Launch web apps with user data,
* Secure them with SG/NACL,
* Place them behind ALB,
* Test metadata,
* Use Bastion for SSH.

---