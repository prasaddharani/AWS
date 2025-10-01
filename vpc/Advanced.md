# Quick index (jump to any section)

1. NAT Gateway
2. NACL & Security Groups (SG)
3. VPC Endpoints (types + hands-on)
4. VPC Peering
5. VPC Flow Logs

---

## 1) NAT Gateway — quick objective

**Goal:** create/verify NAT Gateways (multi-AZ), confirm private instances have outbound internet via NAT, and see NAT’s private/EIP mapping.
Key fact: NAT Gateway must be created in a **public subnet** and is given an Elastic IP for internet access. ([AWS Documentation][1])

### Console steps (create + verify)

1. VPC Console → **NAT Gateways** → **Create NAT Gateway**.

    * Select **Subnet** = one of your public subnets (AZ-specific).
    * **Elastic IP allocation**: create/choose an Elastic IP and attach.
2. Repeat for the second AZ if you want HA (one NAT per AZ recommended).
3. Update the **Route Table** for each private subnet:

    * VPC Console → **Route Tables** → select private subnet’s route table → **Edit routes** → add `0.0.0.0/0` → target = *NAT Gateway id (in same AZ)*.
4. Verify: VPC Console → **NAT Gateways** → open NAT GW details → see **Private IP** and **Elastic IP** listed. (NAT’s ENI holds a private IP while the EIP is the public identity.)
5. Test from a private EC2 (SSH via bastion):

   ```bash
   curl -s https://ifconfig.me   # returns NAT Elastic IP when going out to internet
   curl -I https://google.com    # should return HTTP headers
   ```
6. CLI checks:

   ```bash
   aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=<vpc-id>"
   ```

### Troubleshooting

* If `curl` times out: ensure NAT is in public subnet and public route table has IGW. ([AWS Documentation][2])
* Check NAT quota (EIPs per NAT, connections): docs mention limits and how to request adjustments. ([AWS Documentation][3])

---

## 2) NACL (Network ACL) and Security Groups (SG) — quick objective

**Goal:** review differences, practice editing rules, and demonstrate SG stateful vs NACL stateless behavior (ephemeral ports test).

**Key facts:**

* **Security groups are stateful** (return traffic is automatically allowed).
* **NACLs are stateless** (you must explicitly allow both inbound and outbound). ([AWS Documentation][4])

### Console steps — Security Groups (instance level)

1. EC2 Console → **Security Groups** → **Create security group**.

    * Example SGs: `sg-alb` (HTTP 80 from 0.0.0.0/0), `sg-private-web` (allow 80 from `sg-alb`), `sg-bastion` (SSH from your IP).
2. Attach SG to ALB/EC2/Bastion via EC2/ELB console.

### Console steps — NACLs (subnet level)

1. VPC Console → **Network ACLs** → select or **Create network ACL**.
2. Edit **Inbound rules** and **Outbound rules** (rules are processed by rule number; remember they are stateless).

    * Example for stateless test:

        * Inbound allow TCP 80 from `0.0.0.0/0`
        * Outbound DENY TCP 1024–65535 (ephemeral range) → you will see inbound requests reach instance, but responses blocked (timeout) — demonstrates statelessness.

### Tests / Demonstrations

* On instance: `curl localhost` → always works (loopback).
* From Bastion: `curl http://<private-ip>` → works if SG+NACL inbound allow.
* From Internet (or ALB): request a page: with NACL outbound ephemeral blocked, requests will arrive but responses will be dropped (timeout) — shows NACL is stateless and replies use ephemeral ports.

### CLI checks

```bash
aws ec2 describe-network-acls --filter Name=vpc-id,Values=<vpc-id>
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=<vpc-id>"
```

### Troubleshooting & best practice

* Use SGs to control instance access (instance-level firewall). Use NACLs as a secondary subnet boundary (DDoS-like blocks, explicit subnet denies).
* Remember rule ordering for NACLs (lower rule number first).

---

## 3) VPC Endpoints — types & hands-on

**Goal:** create and test Interface and Gateway endpoints; understand when to use each.

**Key facts:**

* **Gateway endpoints**: route-based endpoints for **Amazon S3 and DynamoDB** (do not use PrivateLink). ([AWS Documentation][5])
* **Interface endpoints**: ENI-based, use **AWS PrivateLink** — common for many AWS services and partner services (creates ENIs in your subnets). ([AWS Documentation][6])
* **Gateway Load Balancer endpoints**: used to insert inspection appliances (GWLB) in path (advanced).

### Console steps — Gateway endpoint (S3) (free)

1. VPC Console → **Endpoints** → **Create endpoint**.
2. Type: **Gateway** → Service Name: `com.amazonaws.<region>.s3` (or DynamoDB).
3. Select VPC → Select route tables (the private subnets’ route tables will get a route to the endpoint).
4. Create → confirm route table entries updated automatically.

**Test:** from a private EC2 (no IGW or NAT) try:

```bash
aws s3 ls s3://aws-public-data-...   # should work if role/credentials allowed
curl https://s3.<region>.amazonaws.com/<object>  # if using path-style, pay attention to DNS
```

### Console steps — Interface endpoint (PrivateLink)

1. VPC Console → **Endpoints** → **Create endpoint**.
2. Type: **Interface** → choose the AWS service or partner service (e.g., `com.amazonaws.<region>.logs` for CloudWatch logs).
3. Select subnets (ENIs will be created in chosen subnets) and security group(s) that control who can connect to the endpoint.
4. Create → check ENIs in **Network Interfaces**.

**Test:** from EC2: attempt to reach the service’s private DNS name (Private DNS is usually enabled by default for supported services) and observe traffic going over endpoint ENI.

### CLI checks

```bash
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=<vpc-id>"
```

### Notes & recommendations

* Use **gateway endpoints** for heavy S3/DynamoDB traffic (lower cost, no ENIs).
* Use **interface endpoints (PrivateLink)** for private access to other AWS services or partner APIs; secure with endpoint SGs.

---

## 4) VPC Peering — quick objective

**Goal:** create a peering connection between two VPCs, add routes, and test connectivity; confirm non-transitive nature.

**Key facts:**

* VPC peering **does not support transitive peering** (A↔B and A↔C does not allow B↔C via A). ([AWS Documentation][7])

### Console steps (create & accept)

1. VPC Console → **Peering Connections** → **Create Peering Connection**.

    * Choose requester and accepter VPC (can be same account or another account).
2. In accepter account (or same account), **Accept** the peering.
3. Update **Route Tables** in both VPCs: add a route for the peer CIDR → target = peering connection ID.
4. Update **Security Groups** to allow traffic from the peer CIDR (security groups are per instance).
5. Test: from an instance in VPC A `ping` or `curl` private IP in VPC B (if ICMP/HTTP allowed).

### Checks & CLI

```bash
aws ec2 describe-vpc-peering-connections --filters "Name=requester-vpc-info.vpc-id,Values=<vpc-id>"
```

### Limitations / gotchas

* CIDR blocks must not overlap.
* No transitive routing — if you need hub-and-spoke or many-to-many, consider Transit Gateway.
* Update route tables carefully; responses must route back via peering (symmetric routes).

---

## 5) VPC Flow Logs — quick objective

**Goal:** enable Flow Logs, direct logs to CloudWatch Logs or S3, and query / inspect traffic to help troubleshooting.

**Key fact:** VPC Flow Logs capture IP traffic to/from network interfaces in your VPC; they can publish to CloudWatch Logs, S3, or Kinesis Data Firehose. ([AWS Documentation][8])

### Console steps (create flow log)

1. VPC Console → select **Your VPCs / Subnets / Network Interfaces** → **Actions → Create flow log**.
2. Configure:

    * **Filter:** All / Accept / Reject
    * **Destination:** CloudWatch Logs group or S3 bucket (or Kinesis Data Firehose)
    * **IAM role**: allow VPC to publish logs.
3. Create → wait ~10–15 minutes for first logs.

### Test & query

* Generate traffic (e.g., curl from/to instance) to create flows.
* CloudWatch Logs → log group set above → use **CloudWatch Logs Insights** to query (use fields like srcAddr, dstAddr, srcPort, dstPort, action).
* If stored in S3, use **AWS Athena** to query large sets (docs show recommended schema). ([AWS Documentation][9])

### Useful queries

* Show rejected traffic (Filter = Reject) to detect blocked flows.
* Query by ENI (interface-id) to map flows to a specific instance.

### CLI checks

```bash
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=<vpc-id>"
```

### Best practices

* Start with **All** if investigating, then switch to **Reject** for ongoing monitoring to save costs.
* Partition S3 logs by time (hour/day) for easier Athena queries.
* Be aware of cost: CloudWatch Logs ingestion/storage or S3 storage costs apply.

---

## Quick troubleshooting checklist (one-page)

* NAT: If private EC2 cannot reach internet → NAT in public subnet? (has EIP?) → private route table points to NAT? ([AWS Documentation][1])
* ALB→EC2: ALB SG allows 0.0.0.0/0 → EC2 SG allows inbound from ALB SG on health port (80)?
* NACLs: If traffic inconsistent → remember statelessness (allow both inbound/outbound, ephemeral ports for responses). ([AWS Documentation][4])
* Endpoints: If S3 access failing from private PE → ensure gateway endpoint or interface endpoint configured and route tables are updated. ([AWS Documentation][5])
* Peering: If ping fails → check route tables in both VPCs and SGs allow peer CIDR; ensure no overlapping CIDRs and remember no transitive routing. ([AWS Documentation][7])
* Flow logs: Enable at VPC/subnet/ENI level, wait ~10–15 min, then query in CloudWatch/Athena. ([AWS Documentation][8])

---

## Quick copy-paste CLI helpers

```bash
# NAT gateways
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=<vpc-id>"

# Security Groups
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=<vpc-id>"

# NACLs
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=<vpc-id>"

# VPC Endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=<vpc-id>"

# VPC Peering
aws ec2 describe-vpc-peering-connections --filters "Name=requester-vpc-info.vpc-id,Values=<vpc-id>"

# Flow Logs
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=<vpc-id>"
```

---
Which of those would you like next?

[1]: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html?utm_source=chatgpt.com "NAT gateways - Amazon Virtual Private Cloud"
[2]: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html?utm_source=chatgpt.com "Troubleshoot NAT gateways - Amazon Virtual Private Cloud"
[3]: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-working-with.html?utm_source=chatgpt.com "Work with NAT gateways - Amazon Virtual Private Cloud"
[4]: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html?utm_source=chatgpt.com "Control subnet traffic with network access control lists"
[5]: https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html?utm_source=chatgpt.com "Gateway endpoints - Amazon Virtual Private Cloud"
[6]: https://docs.aws.amazon.com/vpc/latest/privatelink/create-interface-endpoint.html?utm_source=chatgpt.com "Access an AWS service using an interface VPC endpoint"
[7]: https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html?utm_source=chatgpt.com "How VPC peering connections work"
[8]: https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html?utm_source=chatgpt.com "Logging IP traffic using VPC Flow Logs"
[9]: https://docs.aws.amazon.com/athena/latest/ug/vpc-flow-logs.html?utm_source=chatgpt.com "Query Amazon VPC flow logs - Amazon Athena"
