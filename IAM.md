# 🔐 AWS IAM — Complete Guide

## What is IAM?

**AWS Identity and Access Management (IAM)** AWS ka security system hai jo control karta hai **"kaun kya kar sakta hai"** tumhare AWS account mein.

Real-world analogy:
> Socho tumhara office hai. **IAM** wohi system hai jo decide karta hai:
> - Kon employee kis room mein ja sakta hai (permissions)
> - Har employee ka apna ID card hota hai (credentials)
> - Security guard check karta hai (AWS validates every request)

IAM ke bina, sab ke paas sab kuch access hoga — bahut dangerous!

---

## Why Use IAM?

- **Security** — Har service ko sirf wahi access do jo usse chahiye (**least privilege principle**)
- **Separation** — Root account (admin) ko daily use mat karo
- **Audit** — Track karo ki kis ne kya kiya
- **Automation** — CI/CD pipelines aur apps ke liye separate credentials
- **Team** — Multiple developers ko manage karo with different permissions

---

## IAM Key Concepts

| Concept | Kya hai |
|---------|---------|
| **User** | Ek person ya application — long-term credentials (access keys) |
| **Group** | Users ka collection — ek group ko ek policy assign karo |
| **Role** | EC2/Lambda etc. ke liye identity — temporary credentials (no password/key) |
| **Policy** | JSON document — define karta hai kya allowed/denied hai |
| **Access Key** | Programmatic access ke liye — Key ID + Secret Key pair |
| **Root Account** | AWS account ka owner — almost kabhi use mat karo! |

---

## IAM Policy — Kaise Kaam Karta Hai?

Policy ek **JSON document** hai jo define karta hai permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",          // Allow ya Deny
      "Action": [                 // Kya kar sakta hai
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"  // Kis resource par
    }
  ]
}
```

Simple rule: **Effect + Action + Resource = Permission**

---

## Step-by-Step: Create IAM User for Your App

Yeh sab se common use case hai — backend app ke liye S3 access dena.

#### Step 1: Open IAM Console

AWS Console → Search "IAM" → Click **Users** → **Create user**

#### Step 2: Set User Details

- **Username:** `express-app-s3-user` *(descriptive naam rakho)*
- **Provide user access to AWS Console:** ❌ No *(programmatic access only)*

Click **Next**

#### Step 3: Attach Permissions

**Option A — Attach existing policy (Quick):**

Click **Attach policies directly** → Search `AmazonS3FullAccess` → Select it

> ⚠️ `AmazonS3FullAccess` gives access to ALL S3 buckets. For production, use a custom policy (see below).

**Option B — Create custom policy (Recommended for Production):**

Click **Create policy** → JSON tab → Paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificBucketOnly",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-app-name-uploads",
        "arn:aws:s3:::your-app-name-uploads/*"
      ]
    }
  ]
}
```

> Replace `your-app-name-uploads` with your actual bucket name.

Name the policy: `ExpressAppS3Policy` → Create → Attach to user

#### Step 4: Review & Create User

Click **Create user** ✅

#### Step 5: Create Access Keys

User detail page → **Security credentials** tab → **Create access key**

- **Use case:** Select **Application running outside AWS**
- Click **Next** → **Create access key**

> 🚨 **CRITICAL:** Download the `.csv` file or copy both keys NOW.
> **AWS will NEVER show the Secret Access Key again after this page!**

```
Access Key ID:     AKIAIOSFODNN7EXAMPLE
Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

---

## Step-by-Step: Create IAM Role for EC2

Role EC2 instance ko directly AWS services access karne deta hai — **bina access keys ke!**

> 💡 This is more secure than storing keys in `.env` on EC2.

#### Step 1: Create Role

IAM → **Roles** → **Create role**

- **Trusted entity:** AWS service
- **Use case:** EC2

Click **Next**

#### Step 2: Attach Policy

Search and select: `AmazonS3FullAccess` (or your custom policy)

Click **Next** → Name: `EC2-S3-Access-Role` → **Create role**

#### Step 3: Attach Role to EC2 Instance

EC2 Console → Select your instance → **Actions** → **Security** → **Modify IAM role** → Select `EC2-S3-Access-Role` → **Update IAM role**

Now your EC2 instance can access S3 **without any credentials in code!**

```typescript
// When EC2 has an IAM Role, no credentials needed!
const s3Client = new S3Client({
  region: process.env.AWS_REGION,
  // No accessKeyId or secretAccessKey needed — Role handles it automatically
});
```

---

## IAM Groups — Team Management

Agar multiple developers hain, groups use karo:

#### Create a Group

IAM → **User groups** → **Create group**

- Name: `developers`
- Attach policies: `AmazonS3ReadOnlyAccess`, `AmazonEC2ReadOnlyAccess`

Add users to the group — they automatically inherit group permissions.

```
Developers Group
  ├── Policy: S3 Read Only
  ├── Policy: EC2 Read Only
  ├── User: alice
  ├── User: bob
  └── User: charlie
```

---

## Real Project Setup: What You Need

For a typical Express + S3 project, create these:

### 1. IAM User — for local development & CI/CD

```
User: express-app-s3-user
Policy: ExpressAppS3Policy (custom — only your bucket)
Access: Access Key + Secret Key
Use: .env file locally, GitHub Secrets for CI/CD
```

### 2. IAM Role — for EC2 in production

```
Role: EC2-S3-Access-Role
Policy: ExpressAppS3Policy (same custom policy)
Attached to: Your EC2 instance
Use: No keys needed on server!
```

---

## Environment Variables Setup

Local development mein `.env`:

```env
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
S3_BUCKET_NAME=your-app-name-uploads
```

EC2 par (if using IAM Role — recommended):

```env
AWS_REGION=ap-south-1
S3_BUCKET_NAME=your-app-name-uploads
# No keys needed! Role handles authentication
```

GitHub Secrets for CI/CD (Settings → Secrets):

```
AWS_ACCESS_KEY_ID      → your key id
AWS_SECRET_ACCESS_KEY  → your secret key
AWS_REGION             → ap-south-1
S3_BUCKET_NAME         → your-app-name-uploads
```

---

## IAM Security Best Practices

| ✅ Do | ❌ Don't |
|-------|---------|
| Create individual IAM users | Share credentials between people |
| Use groups for team permissions | Use root account for daily work |
| Least privilege — only needed access | Give `AdministratorAccess` to everything |
| Rotate access keys regularly | Commit keys to GitHub |
| Use IAM Roles for EC2/Lambda | Store long-term keys on servers |
| Enable MFA on root account | Leave root account without MFA |
| Use custom policies for apps | Use `FullAccess` policies in production |

---

## Security Alert: Never Do This! 🚨

```bash
# ❌ NEVER commit to git
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# ❌ NEVER hardcode in source code
const s3Client = new S3Client({
  credentials: {
    accessKeyId: "AKIAIOSFODNN7EXAMPLE",     // ❌
    secretAccessKey: "wJalrXUtn...",          // ❌
  }
});
```

If you accidentally push keys to GitHub:
1. **Immediately** go to IAM → User → Security credentials → **Deactivate** the key
2. Create a new access key
3. Update all places where old key was used
4. Check AWS CloudTrail for unauthorized usage

---

## Quick Reference: Common IAM Policies

| Policy Name | What it allows |
|-------------|----------------|
| `AmazonS3FullAccess` | All S3 operations on all buckets |
| `AmazonS3ReadOnlyAccess` | Read-only access to all S3 |
| `AmazonEC2FullAccess` | Full EC2 management |
| `AmazonEC2ReadOnlyAccess` | View EC2 resources only |
| `AdministratorAccess` | Full AWS access (use ONLY for root/admin) |
| `PowerUserAccess` | Everything except IAM management |

---

## Summary: IAM in Your Project

```
Your AWS Account
│
├── Root Account (only for billing/initial setup)
│
├── IAM User: express-app-s3-user
│   ├── Policy: ExpressAppS3Policy
│   ├── Access Key → stored in .env (local) / GitHub Secrets (CI/CD)
│   └── Used by: local dev machine, GitHub Actions
│
└── IAM Role: EC2-S3-Access-Role
    ├── Policy: ExpressAppS3Policy
    ├── Attached to: EC2 instance
    └── Used by: production server (no keys needed!)
```

---

> 💡 **Connection:** IAM User credentials → `.env` → `S3.md` ka code use karta hai → Files EC2 se S3 mein store hoti hain. Sab connected hai!
