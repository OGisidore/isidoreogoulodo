---
title: Create an EC2 instance on AWS
date: 2026-05-01 20:10:00 +0100
categories: [cloud, aws]
author: isidore
tags: [aws, ec2, vpc, ssh, iam, devops]
image:
  path: /assets/img/headers/aws-ec2.png
pin: false
page_id: post-ec2-aws
permalink: /posts/create-ec2-instance-aws/
lang-exclusive: ['en']
---

Amazon EC2 lets you launch virtual servers in the cloud quickly. In this guide we create an EC2 instance step by step, connect over SSH, and apply basic hardening practices.

### Prerequisites

- An active AWS account
- An IAM user with EC2 permissions (avoid using the root account)
- An SSH key pair (or create one during launch)
- A VPC available (default or custom)

### Step 1: Open the EC2 service

In the AWS console:

1. Go to **EC2**
2. Click **Launch instance**
3. Give the instance a name (e.g. `web-ec2-dev`)

### Step 2: Choose AMI and instance type

- **AMI**: Ubuntu Server 22.04 LTS (or Amazon Linux)
- **Instance type**: `t2.micro` or `t3.micro` for testing

These types are often free-tier eligible depending on account and region.

### Step 3: Configure SSH access

Under **Key pair (login)**:

- Select an existing key, or
- Create a new key (`.pem`) and download it

Keep the private key safe; you need it for SSH.

### Step 4: Network and security

Under **Network settings**:

- Select your **VPC**
- Leave **Auto-assign public IP** enabled if you connect from the internet
- Create or choose a **security group** with at least:
  - SSH (TCP 22) from your IP only (recommended)

Optional for a web server:

- HTTP (TCP 80)
- HTTPS (TCP 443)

### Step 5: Storage and launch

- Disk size: 8–20 GiB (gp3) as needed
- Review the summary
- Click **Launch instance**

When it is running, copy the public IP or public DNS from the instance page.

### Step 6: SSH into the instance

```bash
chmod 400 /path/to/my-key.pem
ssh -i /path/to/my-key.pem ubuntu@<PUBLIC_IP>
```

Notes:

- Use `ubuntu` on Ubuntu AMIs
- Use `ec2-user` on Amazon Linux

### Create via CLI (optional)

You can also launch from the command line:

```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxxxxxxxxxxx \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-xxxxxxxxxxxxxxxxx \
  --subnet-id subnet-xxxxxxxxxxxxxxxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-ec2-dev}]'
```

### After launch

- Update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

- Disable direct root SSH login
- Open only the ports you need
- Prefer an instance IAM role instead of long-lived access keys on the VM
- Enable monitoring (CloudWatch) and backups

With this base you can deploy an API, a web backend, or a test environment quickly.
