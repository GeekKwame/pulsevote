# PulseVote — Live Polling App
End-to-End Cloud Solution Design & Deployment

A real-time polling web application deployed on AWS using CloudFront, ALB, EC2, S3, ACM, and GitHub Actions CI/CD.

**Live URL:** https://d3qju0bc4vfs1j.cloudfront.net/

---

## 🚀 Project Status Update

### Done So Far
- [x] Create and configure AWS Free Tier account; set up IAM users with least-privilege permissions
- [x] Install and configure AWS CLI locally with named profiles
- [x] Create a GitHub repository with proper branching strategy (`main`, `develop`, feature branches)
- [x] Launch an EC2 instance (`t3.micro`) and configure security groups to allow HTTP/HTTPS traffic
- [x] Create an S3 bucket for static assets with appropriate public access settings
- [x] Document the project environment setup in a `README.md` file in GitHub
- [x] Develop a simple web application (HTML/CSS/JS or lightweight framework) and host it on EC2
- [x] Configure a Target Group pointing to your EC2 instance(s)
- [x] Create an Application Load Balancer (ALB) to distribute traffic to the Target Group
- [x] Configure ALB Listener Rules for HTTP → HTTPS redirect
- [x] Set up Auto Scaling Group to handle traffic spikes
- [x] Upload static assets (images, CSS, JS) to S3 and serve via public URL
- [x] Test ALB health checks and confirm instances are in 'healthy' state
- [x] Push all code to GitHub

### Yet to Do
- [ ] Create a CloudFront distribution with ALB as the origin
- [ ] Attach the ACM certificate to CloudFront to enforce HTTPS on the custom domain
- [ ] Configure CloudFront Behaviors: redirect HTTP to HTTPS, set cache policies for static assets
- [ ] Restrict EC2 security groups to only accept traffic from CloudFront (using AWS-managed prefix lists)
- [ ] Apply IAM policies using principle of least privilege for all AWS resources
- [ ] Enable S3 bucket policies to only allow CloudFront Origin Access Control (OAC)
- [ ] Review and document all open ports and security group rules
- [ ] Test the full HTTPS flow: browser → CloudFront → ALB → EC2
- [ ] Create GitHub Actions workflow to automatically deploy code changes to EC2 on push to `main` branch
- [ ] Set up a deployment script that SSHs into EC2 and pulls latest code from GitHub
- [ ] Configure CloudWatch Alarms for: EC2 CPU utilization > 80%, ALB 5xx error rate > 5%
- [ ] Enable CloudWatch Logs for the web application and set log retention to 7 days
- [ ] Set up AWS Budgets alert to notify when costs approach Free Tier limits
- [ ] Update Trello board to reflect completed tasks and close out backlog items
- [ ] Write a final deployment checklist and update the project README with architecture diagram
- [ ] Conduct a peer code review via GitHub Pull Requests before final merge

---


## Architecture Overview

```
User (Browser)
     │  HTTPS
     ▼
┌─────────────────────┐
│   AWS CloudFront    │  CDN · SSL termination · Global edge
│   (ACM Certificate) │  HTTP → HTTPS redirect
└────────┬────────────┘
         │ HTTPS (origin)
         ▼
┌─────────────────────┐
│  Application Load   │  Traffic distribution
│  Balancer (ALB)     │  Health checks · HTTP → HTTPS redirect
└────────┬────────────┘
         │ HTTP (private)
         ▼
┌─────────────────────┐
│   EC2 (t3.micro)    │  Nginx serves index.html
│   Target Group      │  Security group: CloudFront only
└─────────────────────┘

┌─────────────────────┐
│   Amazon S3         │  Static assets (images, icons)
│   + CloudFront OAC  │  Served via CloudFront only
└─────────────────────┘

┌─────────────────────┐
│   GitHub Actions    │  CI/CD: push to main → deploy to EC2
│   (deploy.yml)      │  via SCP + SSH
└─────────────────────┘
```

---

## Project Structure

```
pulsevote/
│
├── index.html                  ← Full app (HTML + CSS + JS)
├── .gitignore
├── README.md
│
└── .github/
    └── workflows/
        └── deploy.yml          ← CI/CD pipeline
```

---

## AWS Services Used

| Service | Purpose |
|---|---|
| EC2 (t3.micro) | Hosts Nginx web server serving index.html |
| Application Load Balancer | Distributes traffic, HTTP→HTTPS redirect |
| CloudFront | Global CDN, SSL termination, caches static content |
| ACM (Certificate Manager) | Free SSL/TLS certificate for HTTPS |
| S3 | Static asset storage (served via CloudFront OAC) |
| IAM | Least-privilege roles for all resources |
| CloudWatch | CPU and error rate alarms, log retention |
| AWS Budgets | Cost alerts for Free Tier limits |

---

## Phase 1 — Environment Setup

### 1. AWS Account & IAM
- Create AWS Free Tier account
- Create IAM user with least-privilege permissions
- Enable MFA on root account
- Configure AWS CLI:
```bash
aws configure
# Enter: Access Key ID, Secret Key, Region (e.g. us-east-1), Output format (json)
```

### 2. GitHub Repository
```bash
git clone https://github.com/GeekKwame/pulsevote.git
cd pulsevote
```

Branch strategy:
- `main` — production (triggers auto-deploy)
- `develop` — integration branch
- `feature/*` — individual features

### 3. EC2 Instance
- Launch **t3.micro** (Amazon Linux 2 or Ubuntu 22.04)
- Security group: allow **HTTP (80)** and **HTTPS (443)** inbound
- Download and save your `.pem` key file securely

### 4. S3 Bucket
```bash
aws s3 mb s3://pulsevote-assets-YOUR_NAME
```
- Block all public access (CloudFront OAC handles delivery)

### 5. SSL Certificate (ACM)
- Request a public certificate in ACM for your domain
- Validate via DNS (add CNAME record to your domain)
- Wait for status: **Issued**

---

## Phase 2 — EC2 Setup & App Deployment

### Install Nginx on EC2
```bash
# SSH into EC2
ssh -i your-key.pem ec2-user@YOUR_EC2_PUBLIC_IP

# Amazon Linux 2
sudo yum update -y
sudo yum install nginx -y

# Ubuntu
# sudo apt update && sudo apt install nginx -y

# Start and enable Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Configure Nginx
```bash
sudo nano /etc/nginx/nginx.conf
```

Replace the default server block with:
```nginx
server {
    listen 80;
    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Health check endpoint for ALB
    location /health {
        return 200 'healthy';
        add_header Content-Type text/plain;
    }
}
```

```bash
# Create web root and deploy app
sudo mkdir -p /var/www/html
sudo cp index.html /var/www/html/
sudo chown -R nginx:nginx /var/www/html   # Amazon Linux
# sudo chown -R www-data:www-data /var/www/html  # Ubuntu

sudo nginx -t          # Test config
sudo systemctl reload nginx
```

### Manual first deploy
```bash
# From your local machine
scp -i your-key.pem index.html ec2-user@YOUR_EC2_IP:/var/www/html/
```

### ALB Setup
1. Create **Target Group** → target type: Instance → add your EC2
2. Health check path: `/health`
3. Create **Application Load Balancer** → internet-facing → attach target group
4. Add **Listener**: HTTP:80 → redirect to HTTPS:443
5. Add **Listener**: HTTPS:443 → forward to target group → attach ACM cert
6. Confirm target group shows **healthy**

---

## Phase 3 — CloudFront & Security

### CloudFront Distribution
1. Create distribution → origin: your **ALB DNS name**
2. Protocol: HTTPS only
3. Attach ACM certificate
4. Default behavior: redirect HTTP to HTTPS
5. Cache policy: `CachingOptimized` for static content

### Security Hardening
```bash
# Restrict EC2 security group to CloudFront IPs only
# In AWS Console: EC2 → Security Groups → edit inbound rules
# Source: pl-XXXXXXXX (AWS-managed CloudFront prefix list)
```

S3 bucket policy (OAC):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::pulsevote-assets-YOUR_NAME/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT_ID:distribution/DIST_ID"
      }
    }
  }]
}
```

---

## Phase 4 — CI/CD Pipeline Setup

### GitHub Secrets
In your repo → Settings → Secrets and variables → Actions, add:

| Secret | Value |
|---|---|
| `EC2_HOST` | Your EC2 public IP or domain |
| `EC2_USER` | `ec2-user` (Amazon Linux) or `ubuntu` |
| `EC2_SSH_KEY` | Full contents of your `.pem` private key |

### How the pipeline works
1. You push code to `main`
2. GitHub Actions runs `deploy.yml`
3. It SCPs `index.html` directly to `/var/www/html/` on EC2
4. SSH verifies Nginx is running
5. Your live site is updated — no manual steps

### CloudWatch Alarms
```bash
# EC2 CPU > 80%
aws cloudwatch put-metric-alarm \
  --alarm-name "PulseVote-HighCPU" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:REGION:ACCOUNT:YOUR_SNS_TOPIC
```

Log retention:
```bash
aws logs put-retention-policy \
  --log-group-name /pulsevote/nginx \
  --retention-in-days 7
```

### AWS Budget Alert
- Go to AWS Budgets → Create budget
- Type: Cost budget · Threshold: $5
- Alert: email when forecast exceeds threshold

---

## Deployment Checklist

Refer to the **[Project Status Update](#-project-status-update)** section at the top of this document for the complete status of completed and remaining task items.

---

## Local Development

No build tools needed. Just open the file:
```bash
# Option 1 — open directly
open index.html

# Option 2 — serve locally
npx serve .
# then visit http://localhost:3000
```

---

## Security Decisions

**Why CloudFront sits in front of ALB:**
CloudFront absorbs DDoS traffic at the edge before it reaches the origin, and lets us enforce HTTPS globally without exposing the ALB directly.

**Why EC2 only accepts CloudFront traffic:**
The EC2 security group uses the AWS-managed CloudFront prefix list as the inbound source. Direct access to the ALB or EC2 IP is blocked — all traffic must pass through CloudFront.

**Why ACM over self-signed certificates:**
ACM certificates are free, auto-renew, and are trusted by all browsers. They attach directly to CloudFront and the ALB with no server-side configuration.

**Why least-privilege IAM:**
Each AWS resource (EC2 role, GitHub Actions user) has only the permissions it needs — nothing more. This limits blast radius if credentials are ever compromised.

---

## Team

| Name | Role |
|---|---|
| GeekKwame | Cloud Engineer |

---

*AWS Free Tier Capstone Project · Intern Cohort*
