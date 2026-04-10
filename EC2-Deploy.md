# 🚀 EC2 Deployment Guide — Node.js Backend

## What is Amazon EC2?

**Amazon Elastic Compute Cloud (EC2)** is a web service provided by AWS (Amazon Web Services) that allows you to run virtual servers — called **instances** — in the cloud. Instead of buying and maintaining physical hardware, EC2 lets you rent computing capacity on demand, paying only for what you use.

Key concepts:
- **Instance** — A virtual machine running on AWS infrastructure
- **AMI (Amazon Machine Image)** — A pre-configured OS template used to launch instances (e.g., Ubuntu 22.04)
- **Security Group** — A virtual firewall that controls inbound/outbound traffic to your instance
- **Key Pair** — SSH credentials used to securely connect to your instance
- **Elastic IP** — A static public IP address you can assign to your instance

EC2 is commonly used to host web servers, APIs, databases, and CI/CD workloads — making it a great fit for deploying a Node.js backend.

---

## How to Create an EC2 Instance

#### Pre-Step 1: Sign in to AWS Console

Go to [https://aws.amazon.com/console](https://aws.amazon.com/console) and sign in to your AWS account. If you don't have one, create a free tier account.

#### Pre-Step 2: Navigate to EC2

In the AWS Console, search for **EC2** in the top search bar and click on it, then click **Launch Instance**.

#### Pre-Step 3: Configure Your Instance

Fill in the following settings:

- **Name:** Give your instance a name (e.g., `express-app-server`)
- **AMI:** Select **Ubuntu Server 22.04 LTS** (Free Tier eligible)
- **Instance Type:** Select `t2.micro` (Free Tier eligible)
- **Key Pair:** Click **Create new key pair**, name it (e.g., `ec2-deploy-key`), choose `.pem` format, and download it. **Keep this file safe — you cannot download it again.**

#### Pre-Step 4: Configure Security Group

Under **Network settings**, click **Edit** and add the following inbound rules:

| Type  | Protocol | Port | Source    |
|-------|----------|------|-----------|
| SSH   | TCP      | 22   | My IP     |
| HTTP  | TCP      | 80   | Anywhere  |
| HTTPS | TCP      | 443  | Anywhere  |
| Custom TCP | TCP | 8000 | Anywhere (for direct Node.js access) |

#### Pre-Step 5: Launch the Instance

Click **Launch Instance**. Wait 1–2 minutes for the instance to start. Once running, go to your instance details and copy the **Public IPv4 address**.

#### Pre-Step 6: Connect to Your Instance via SSH

On your local machine, navigate to where your `.pem` key file is saved and run:

```bash
# Set correct permissions on the key file
chmod 400 ec2-deploy-key.pem

# Connect to your EC2 instance
ssh -i ec2-deploy-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

You are now connected to your EC2 instance and ready to set it up for deployment.

---

## Deploy the Node.js Backend to EC2

After setting up the EC2 instance and configuring security groups, follow these steps:

#### Step 1: Update & Upgrade Packages

> A fresh EC2 instance may have outdated package lists. Running update & upgrade ensures your system has the latest security patches before installing anything.

```bash
sudo apt update && sudo apt upgrade -y
```

#### Step 2: Install Node.js and npm

> Your Express app runs on Node.js, so it must be installed on the server. We use the `n` version manager to install the **LTS** version — recommended for production.

```bash
sudo apt-get install npm -y
sudo npm i -g n
sudo n lts
# Or install a specific version: sudo n 22.0.1
```

> ⚠️ **After installing Node.js, exit and re-login to your EC2 instance** so the new version is picked up:

```bash
exit
ssh -i ec2-deploy-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
node -v   # confirm new version
```

#### Step 3: Install Nginx (Reverse Proxy)

> Nginx acts as a **reverse proxy** — it sits in front of your Node.js app and forwards incoming HTTP requests (port 80) to your app (port 8000). This keeps your app protected and handles SSL, load balancing, and static file serving efficiently.

```bash
sudo apt install nginx -y

# Start and enable Nginx (auto-starts on reboot)
sudo systemctl start nginx
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

#### Step 4: Setup Deployment Directory

> We create `/var/www/express-app` to store application files. This is the Linux convention for web-served content — organized and separate from system files. Your CI/CD pipeline will deploy code here on every push.

```bash
sudo mkdir -p /var/www/express-app
cd /var/www/express-app

# Clone your GitHub repo into current directory
# Note: The "." at the end clones directly into this folder (no subfolder created)
sudo git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git .
```

> 💡 **The `.` (dot) at the end is important!** Without it, git creates a subfolder like `/var/www/express-app/your-repo-name/`. With `.`, all files go directly into `/var/www/express-app/`.

#### Step 5: Configure Nginx as Reverse Proxy

> We tell Nginx how to forward traffic to our Node.js app — listen on port 80, proxy all requests to `localhost:8000`.

```bash
sudo nano /etc/nginx/sites-available/express-app
```

Paste the following configuration:

```nginx
server {
    listen 80;
    server_name YOUR_EC2_PUBLIC_IP;  # Replace with your IP or domain

    location / {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

Enable the configuration:

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/express-app /etc/nginx/sites-enabled/

# Remove default configuration
sudo rm /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

#### Step 6: Install PM2 — Process Manager

> If your Node.js app crashes or EC2 reboots, your server goes offline without a process manager. **PM2** keeps your app running in the background, auto-restarts on crashes, and revives on system reboot.

```bash
sudo npm install -g pm2

# Generate startup script so PM2 starts on reboot
sudo pm2 startup
```

**Option A — Start with `ecosystem.config.cjs` (Recommended)**

If your project has an `ecosystem.config.cjs` file, use it — it defines your app name, script path, environment variables, etc.:

```bash
sudo pm2 start ecosystem.config.cjs
sudo pm2 save
```

**Option B — Start directly without ecosystem file (Also works!)**

If you don't have an `ecosystem.config.cjs`, you can start the app directly:

```bash
# For a compiled app (dist folder)
sudo pm2 start dist/index.js --name "express-app"

# For a plain JS app
sudo pm2 start index.js --name "express-app"

# Save so PM2 restores on reboot
sudo pm2 save
```

> Both options work fine. The ecosystem file just gives you more control over environment variables and clustering. Use whichever fits your project.

**Useful PM2 Commands:**

```bash
pm2 list              # See all running apps
pm2 logs express-app  # View logs
pm2 restart express-app
pm2 stop express-app
pm2 delete express-app
```

---

## CI/CD with GitHub Actions

#### Step 7: Create GitHub Secrets

> Your GitHub Actions workflow needs to SSH into EC2 to deploy. Never hardcode sensitive values — use **GitHub Secrets** which are encrypted and only visible at runtime.

Go to: **GitHub Repo → Settings → Secrets and variables → Actions → New repository secret**

Add these secrets:

| Secret Name | Value |
|-------------|-------|
| `EC2_HOST` | Your EC2 public IP address |
| `EC2_USERNAME` | `ubuntu` (default for Ubuntu AMI) |
| `EC2_SSH_KEY` | Contents of your `.pem` file |

To copy your `.pem` key content:

```bash
# On your local machine
cat ec2-deploy-key.pem

# Copy ALL content including header and footer:
# -----BEGIN RSA PRIVATE KEY-----
# ... key content ...
# -----END RSA PRIVATE KEY-----
```

#### Step 8: Create GitHub Actions Workflow

> This YAML file automates your entire deployment. Every time you push to `main`, GitHub Actions:
> 1. **Builds** your code and packages it
> 2. **Deploys** it to EC2 via SSH
> 3. **Verifies** the app is running
> 4. **Rolls back** automatically if something fails

Create the file: `.github/workflows/deploy.yml`

```yml
name: Deploy to EC2

on:
  push:
    branches: [main]
  workflow_dispatch: # allows manual trigger from GitHub UI

jobs:
  # ─────────────────────────────────────────
  # JOB 1: Build the application
  # ─────────────────────────────────────────
  build:
    name: Build Application
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20.13.1"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Build TypeScript
        run: npm run build

      - name: Create deployment package (without node_modules)
        run: |
          mkdir -p deploy
          cp -r dist deploy/
          cp -r src deploy/
          cp package*.json deploy/
          # Copy ecosystem config if it exists
          [ -f ecosystem.config.cjs ] && cp ecosystem.config.cjs deploy/ || echo "No ecosystem.config.cjs found"
          # Copy .env.example if exists
          [ -f .env.example ] && cp .env.example deploy/ || echo "No .env.example found"
          tar -czf deploy.tar.gz -C deploy .

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: deployment-package
          path: deploy.tar.gz
          retention-days: 1

  # ─────────────────────────────────────────
  # JOB 2: Deploy to EC2
  # ─────────────────────────────────────────
  deploy:
    name: Deploy to EC2
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: deployment-package

      - name: Deploy to EC2 via SSH
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
          EC2_USERNAME: ${{ secrets.EC2_USERNAME }}
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
        run: |
          echo "$SSH_PRIVATE_KEY" > private_key.pem
          chmod 600 private_key.pem

          # Copy deployment package to EC2
          scp -i private_key.pem -o StrictHostKeyChecking=no \
            deploy.tar.gz ${EC2_USERNAME}@${EC2_HOST}:/tmp/

          # SSH into EC2 and deploy
          ssh -i private_key.pem -o StrictHostKeyChecking=no \
            ${EC2_USERNAME}@${EC2_HOST} << 'EOF'

            cd /var/www/express-app

            # Fix ownership
            echo "Fixing directory ownership..."
            sudo chown -R $USER:$USER /var/www/express-app

            # Backup current version
            if [ -d "dist" ]; then
              timestamp=$(date +%Y%m%d_%H%M%S)
              mkdir -p backups
              tar -czf backups/backup_${timestamp}.tar.gz dist package.json ecosystem.config.cjs .env 2>/dev/null || true
              # Keep only last 5 backups
              ls -t backups/backup_*.tar.gz 2>/dev/null | tail -n +6 | xargs -r rm
            fi

            # Extract new version
            echo "Extracting new deployment package..."
            tar --overwrite -xzf /tmp/deploy.tar.gz -C /var/www/express-app
            rm /tmp/deploy.tar.gz

            # Fix ownership after extraction
            sudo chown -R $USER:$USER /var/www/express-app

            # Install production dependencies
            echo "Installing production dependencies..."
            npm ci

            # Create .env if it doesn't exist (first deploy only)
            if [ ! -f .env ]; then
              echo "Creating default .env file..."
              echo "NODE_ENV=production" > .env
              echo "PORT=8000" >> .env
            fi

            mkdir -p logs

            # Start or reload with PM2
            # If ecosystem.config.cjs exists, use it — otherwise use direct start
            if [ -f ecosystem.config.cjs ]; then
              if pm2 describe express-app > /dev/null 2>&1; then
                echo "Reloading PM2 app (zero-downtime)..."
                sudo pm2 reload ecosystem.config.cjs --update-env
              else
                echo "Starting new PM2 process with ecosystem..."
                sudo pm2 start ecosystem.config.cjs
              fi
            else
              if pm2 describe express-app > /dev/null 2>&1; then
                echo "Reloading PM2 app..."
                sudo pm2 restart express-app --update-env
              else
                echo "Starting new PM2 process..."
                sudo pm2 start dist/index.js --name "express-app"
              fi
            fi

            sudo pm2 save
          EOF

          rm private_key.pem

  # ─────────────────────────────────────────
  # JOB 3: Verify deployment
  # ─────────────────────────────────────────
      - name: Verify deployment
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
        run: |
          echo "Waiting for application to start..."
          sleep 10

          for i in {1..2}; do
            echo "Attempt $i of 2..."
            response=$(curl -s -o /dev/null -w "%{http_code}" http://${EC2_HOST} 2>/dev/null || echo "000")

            if [ "$response" = "200" ] || [ "$response" = "301" ] || [ "$response" = "302" ]; then
              echo "✅ Deployment successful! Status: $response"
              exit 0
            fi

            echo "Got response: $response, waiting 5 seconds..."
            sleep 5
          done

          echo "❌ Deployment verification failed"
          echo "Check PM2 logs on EC2: pm2 logs express-app"
          exit 1

  # ─────────────────────────────────────────
  # JOB 4: Rollback on failure
  # ─────────────────────────────────────────
      - name: Rollback on failure
        if: failure()
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
          EC2_USERNAME: ${{ secrets.EC2_USERNAME }}
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
        run: |
          echo "🔄 Attempting rollback to previous version..."

          echo "$SSH_PRIVATE_KEY" > private_key.pem
          chmod 600 private_key.pem

          ssh -i private_key.pem -o StrictHostKeyChecking=no \
            ${EC2_USERNAME}@${EC2_HOST} << 'EOF'

            cd /var/www/express-app
            sudo chown -R $USER:$USER /var/www/express-app

            latest_backup=$(ls -t backups/backup_*.tar.gz 2>/dev/null | head -1)

            if [ -n "$latest_backup" ]; then
              echo "Found backup: $latest_backup"
              tar -xzf "$latest_backup" -C /var/www/express-app
              npm ci
              pm2 reload ecosystem.config.cjs 2>/dev/null || pm2 restart express-app
              echo "✅ Rolled back successfully"
            else
              echo "⚠️ No backup found, cannot rollback"
            fi
          EOF

          rm private_key.pem
```

---

> ✅ **You're done!** Every push to `main` will now automatically build and deploy your app to EC2 with zero manual steps.
