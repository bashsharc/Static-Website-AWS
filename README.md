# Host a Static Website on AWS — Three-Tier Architecture

A production-grade deployment of a static HTML website on AWS, built for high 
availability, fault tolerance, and scalability using a three-tier VPC architecture 
across multiple Availability Zones.

---

## 📋 Overview

This project involved building a robust AWS infrastructure to host a static HTML 
website (Jupiter). Rather than a single-server setup, the site runs on multiple EC2 
instances behind a load balancer, distributed across two Availability Zones, with 
auto scaling to handle traffic spikes automatically.

**Key design goals:**
- No single point of failure across compute and network
- Web servers never directly exposed to the internet
- Automatic scaling based on demand
- Free, auto-renewing SSL via AWS Certificate Manager

---

## 🏗️ Architecture

<div align="center">
  <img width="616" height="384" alt="Three-Tier AWS Architecture" src="https://github.com/user-attachments/assets/932400b2-83bb-4ed9-88a3-7b283d261f1a" />
</div>

---

## 🛠️ Tools & Technologies

| Category | Tools |
|---|---|
| **Compute** | Amazon EC2, Auto Scaling Groups, Launch Templates |
| **Networking** | Amazon VPC, Internet Gateway, NAT Gateway, Route Tables, Security Groups |
| **Load Balancing** | Application Load Balancer, Target Groups |
| **DNS & SSL** | Amazon Route 53, AWS Certificate Manager (ACM) |
| **Monitoring** | Amazon CloudWatch, Amazon SNS |
| **Web Server** | Apache HTTP Server |
| **Scripting** | Bash (EC2 User Data automation) |

---

## 🔐 Security Design

The Internet Gateway is the single entry point into the VPC from the public internet. 
All inbound traffic flows through it to reach the ALB — web servers sit in private 
subnets and are never directly accessible:

| Security Group | Open Ports | Allowed Source |
|---|---|---|
| ALB SG | 80, 443 | Anywhere (0.0.0.0/0 via IGW) |
| SSH SG | 22 | My IP only |
| Webserver SG | 80, 443, 22 | ALB SG / SSH SG |

---

## 🚀 Deployment Steps

### 1. Build the Three-Tier VPC

- Created **Dev VPC** → enabled DNS hostnames
- Attached **Internet Gateway** — the single connection point between the VPC and 
  the public internet; required for the ALB to receive inbound traffic and for the 
  Bastion Host to be reachable via SSH
- Created **2 public subnets** (one per AZ) — auto-assign public IPv4 enabled
- Created **public route table** — added route `0.0.0.0/0 → Internet Gateway` → 
  associated with both public subnets
- Created **4 private subnets** (2 app, 2 data) — use the default private route 
  table (local traffic only)

### 2. Set Up NAT Gateways

Deployed one NAT Gateway per AZ (in public subnets) with Elastic IPs, and created 
private route tables routing `0.0.0.0/0 → NAT Gateway` for each AZ. This gives 
private instances outbound internet access (for updates/installs) without exposing 
them inbound.

### 3. Configure Security Groups

Three security groups in a chain — each tier only accepts traffic from the tier 
directly above it (see table above).

### 4. Launch EC2 Instances & ALB

Launched two EC2 instances in private app subnets using this User Data script:

```bash
#!/bin/bash
sudo su
yum update -y
yum install -y httpd
cd /var/www/html
wget https://github.com/azeezsalu/jupiter/archive/refs/heads/main.zip
unzip main.zip
cp -r jupiter-main/* /var/www/html/
rm -rf jupiter-main main.zip
systemctl enable httpd
systemctl start httpd
```

Created a **Target Group** and **Application Load Balancer** (internet-facing, 
public subnets). Verified site loads via ALB DNS name.

### 5. Configure Custom Domain & SSL

- Registered domain in **Route 53** → created A Record aliased to the ALB
- Requested SSL certificate in **ACM** (DNS validation via Route 53)
- Added HTTPS listener (port 443) to ALB → HTTP listener redirects to HTTPS

### 6. Set Up Auto Scaling Group

Created a **Launch Template** from a working instance, then an **Auto Scaling Group**:
- Subnets: Private App AZ1 + AZ2
- Target Group: Dev-TG
- Health Checks: EC2 + ELB
- Capacity: Desired 2 | Min 1 | Max 4
- SNS notifications for scaling events

---

## 📸 Screenshots

### Final Architecture
![Final Architecture](https://github.com/user-attachments/assets/932400b2-83bb-4ed9-88a3-7b283d261f1a)

### VPC & Subnets
![VPC Architecture](https://github.com/user-attachments/assets/c477bcd5-4090-4f25-9339-d20576af1e94)

### NAT Gateway Setup
![NAT Gateway](https://github.com/user-attachments/assets/b747679e-3473-4cf1-b2da-ee512635ddb4)

### Security Groups
![Security Groups](https://github.com/user-attachments/assets/853a654a-565b-4930-9ab1-51ce50fbfa51)

### Live Website
![Live Site](https://github.com/user-attachments/assets/0c2c87b5-f14f-4809-a6b9-b589a0118510)

---

## 💡 Key Lessons Learned

- **The Internet Gateway is what makes a subnet truly "public."** The IGW must be 
  both attached to the VPC *and* referenced in the public route table 
  (`0.0.0.0/0 → IGW`). Without both pieces, traffic has nowhere to go.
- **IGW vs NAT Gateway serve opposite directions.** The IGW handles both inbound 
  and outbound for public resources. The NAT Gateway handles outbound-only for 
  private resources — letting EC2 instances reach the internet for updates while 
  staying invisible to inbound traffic.
- **Security Groups chain together.** The Webserver SG only trusts the ALB SG, 
  so even if someone finds a server's private IP, they can't reach it directly.
- **User Data scripts make EC2 provisioning repeatable.** The same script that 
  builds the manual instances feeds directly into the Launch Template, so Auto 
  Scaling instances come up identically every time.
- **Auto Scaling + Load Balancing work as a team.** The ALB distributes traffic 
  evenly; the ASG keeps the right number of healthy servers running with no manual 
  intervention.

---

## 🧹 Cost Management

Resources that incur hourly charges even when idle: **NAT Gateways** (biggest 
cost — ~$32/month each), **ALB**, and **running EC2 instances**. Resources free 
even when idle: VPC, subnets, route tables, Internet Gateway, security groups, 
ACM certificate.

Teardown order: ASG → ALB/Target Group → NAT Gateways → Elastic IPs → Security 
Groups → Internet Gateway → VPC.
