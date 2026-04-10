# AWS Deployment Guide — EC2, S3 & IAM

A complete reference for deploying a Node.js backend to AWS using EC2, storing files in S3, managing permissions with IAM, and automating deployments with GitHub Actions CI/CD.

---

## What's in This Repository

| File | Description |
|------|-------------|
| [EC2-Deploy.md](EC2-Deploy.md) | How to launch an EC2 instance and deploy a Node.js/Express app with Nginx + PM2 |
| [S3-Bucket.md](S3-Bucket.md) | How to create an S3 bucket and use it for file uploads, static hosting, and pre-signed URLs |
| [IAM.md](IAM.md) | How to create IAM users, roles, and policies to securely manage AWS access |

---

## Overview

### Amazon EC2 — Virtual Server

EC2 (Elastic Compute Cloud) lets you run virtual servers in the cloud. This guide uses it to host a Node.js backend with:

- **Nginx** as a reverse proxy (port 80 → port 8000)
- **PM2** as a process manager (auto-restart, survives reboots)
- **GitHub Actions** for automated deployments on every push to `main`

### Amazon S3 — Object Storage

S3 (Simple Storage Service) is used to store files independently of your EC2 instance:

- User-uploaded files (images, PDFs, documents)
- Static frontend hosting (React/Next.js builds)
- Application backups
- Pre-signed URLs for private file access

### AWS IAM — Access Management

IAM (Identity and Access Management) controls who can do what in your AWS account:

- **IAM User** — for local development and CI/CD pipelines (uses access keys)
- **IAM Role** — for EC2 instances in production (no keys needed, more secure)
- **IAM Policies** — JSON documents that grant least-privilege permissions

---

## Architecture

```
GitHub (push to main)
        │
        ▼
GitHub Actions CI/CD
   ├── Build Node.js app
   ├── SSH into EC2
   ├── Deploy new version
   ├── Verify health check
   └── Rollback on failure
        │
        ▼
EC2 Instance (Ubuntu)
   ├── Nginx (port 80/443) → reverse proxy
   ├── PM2 → runs Node.js app (port 8000)
   └── Node.js App
            │
            ▼
        AWS S3
   └── Stores uploaded files / static assets
```

---

## Quick Start

### 1. Set Up IAM

Create credentials before anything else — both EC2 and GitHub Actions need them.

- Create an **IAM User** with a custom S3 policy (for local dev and CI/CD)
- Create an **IAM Role** and attach it to your EC2 instance (for production — no keys on server)

See [IAM.md](IAM.md) for full step-by-step instructions.

### 2. Create an S3 Bucket

- Go to AWS Console → S3 → **Create bucket**
- Choose the same region as your EC2 instance (e.g., `ap-south-1`)
- Keep **Block all public access** enabled for backend/user uploads
- Configure CORS if your frontend uploads directly to S3

See [S3-Bucket.md](S3-Bucket.md) for full step-by-step instructions.

### 3. Launch and Configure EC2

- Launch a `t2.micro` Ubuntu 22.04 instance (Free Tier eligible)
- Configure security group: ports 22 (SSH), 80 (HTTP), 443 (HTTPS), 8000 (Node.js)
- SSH in and install Node.js, Nginx, and PM2
- Clone your repo to `/var/www/express-app`
- Configure Nginx as a reverse proxy to port 8000

See [EC2-Deploy.md](EC2-Deploy.md) for full step-by-step instructions.

### 4. Set Up CI/CD with GitHub Actions

Add these secrets to your GitHub repo (**Settings → Secrets and variables → Actions**):

| Secret | Value |
|--------|-------|
| `EC2_HOST` | Your EC2 public IP address |
| `EC2_USERNAME` | `ubuntu` |
| `EC2_SSH_KEY` | Full contents of your `.pem` key file |

Create `.github/workflows/deploy.yml` — the workflow automatically:

1. Builds your TypeScript app
2. Packages and transfers it to EC2 via SCP
3. Installs dependencies and reloads PM2
4. Verifies the app is responding
5. Rolls back to the previous version if verification fails

See [EC2-Deploy.md](EC2-Deploy.md) (Step 7–8) for the full workflow file.

---

## Environment Variables

**Local development (`.env`):**

```env
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=your_iam_user_access_key
AWS_SECRET_ACCESS_KEY=your_iam_user_secret_key
S3_BUCKET_NAME=your-app-name-uploads
PORT=8000
```

**EC2 production (using IAM Role — no keys needed):**

```env
AWS_REGION=ap-south-1
S3_BUCKET_NAME=your-app-name-uploads
NODE_ENV=production
PORT=8000
```

> Never commit `.env` to git. Add it to `.gitignore`.

---

## Security Best Practices

- Use **IAM Roles** on EC2 instead of storing access keys on the server
- Apply **least privilege** — grant only the S3 bucket your app actually uses
- Never commit AWS credentials to GitHub — use GitHub Secrets for CI/CD
- Keep SSH access restricted to your IP in the EC2 security group
- Rotate IAM access keys regularly
- Enable MFA on your AWS root account

---

## Tech Stack

- **Server:** Ubuntu 22.04 on AWS EC2 (t2.micro)
- **Runtime:** Node.js (LTS) with Express
- **Reverse Proxy:** Nginx
- **Process Manager:** PM2
- **Storage:** AWS S3
- **Auth/Access:** AWS IAM
- **CI/CD:** GitHub Actions
