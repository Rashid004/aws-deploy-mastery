# 🪣 AWS S3 — Complete Guide

## What is Amazon S3?

**Amazon Simple Storage Service (S3)** is AWS ka object storage service hai. Isko samjho ek **infinitely large cloud hard drive** ki tarah — jahan tum files store kar sakte ho aur globally access kar sakte ho.

S3 mein tum kuch bhi store kar sakte ho:
- Images, videos, PDFs
- Static frontend files (HTML, CSS, JS)
- Backups and archives
- User-uploaded content (avatars, documents)
- Application logs

---

## Why Use S3?

| Problem | S3 Solution |
|--------|-------------|
| EC2 mein files store karo — instance delete ho to sab gone | S3 **independent** hai, EC2 se alag rehta hai |
| Large files EC2 ki memory khaate hain | S3 mein **unlimited storage**, EC2 load-free |
| User uploads directly server par aate hain | S3 handles uploads **directly from browser** |
| Static site ke liye bhi EC2 run karna padta hai | S3 static sites **bina server ke** host karta hai |

Key benefits:
- **Durability** — 99.999999999% (11 nines!) data durability
- **Scalability** — No storage limits
- **Cost** — Pay only for what you use (very cheap)
- **Global** — Access from anywhere

---

## S3 Key Concepts

- **Bucket** — Top-level container (folder jaisa) — globally unique name hona chahiye
- **Object** — Har file jo tum store karte ho (with key, value, metadata)
- **Key** — File ka path/name inside bucket (e.g., `uploads/avatar/user123.jpg`)
- **ACL (Access Control List)** — Who can read/write
- **Pre-signed URL** — Temporary URL to access private files
- **Bucket Policy** — JSON rules defining who can access what

---

## Step-by-Step: Create an S3 Bucket

#### Step 1: Open S3 Console

AWS Console → Search "S3" → Click **Create bucket**

#### Step 2: Configure Bucket

- **Bucket name:** `your-app-name-uploads` *(must be globally unique)*
- **Region:** Same region as your EC2 (e.g., `ap-south-1` for Mumbai)

#### Step 3: Block Public Access Settings

Two scenarios:

**Scenario A — Private bucket (backend uploads, user files):**
```
✅ Block all public access → Keep ENABLED (default)
```

**Scenario B — Public bucket (static website, public images):**
```
❌ Uncheck "Block all public access"
✅ Acknowledge the warning
```

#### Step 4: Create Bucket

Click **Create bucket**. Done! 🎉

---

## Use Case 1: Static Frontend Hosting on S3

React/Next.js ka build S3 par host karo — bina EC2 ke!

#### Step 1: Build your frontend

```bash
npm run build
# Output: /dist or /build folder
```

#### Step 2: Enable Static Website Hosting

S3 Bucket → **Properties** tab → **Static website hosting** → Enable

- **Index document:** `index.html`
- **Error document:** `index.html` *(important for SPA routing)*

#### Step 3: Set Bucket Policy (Public Read)

S3 Bucket → **Permissions** tab → **Bucket policy** → Paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```

> Replace `YOUR-BUCKET-NAME` with your actual bucket name.

#### Step 4: Upload Build Files

```bash
# Install AWS CLI first if not installed
# Then configure it (IAM credentials needed — see IAM.md)
aws s3 sync ./dist s3://YOUR-BUCKET-NAME --delete
```

Your site is now live at:
```
http://YOUR-BUCKET-NAME.s3-website.REGION.amazonaws.com
```

> 💡 For custom domain + HTTPS, use **CloudFront** in front of S3.

---

## Use Case 2: File Uploads from Backend (Node.js)

Users jo files upload karte hain (images, PDFs) — unhe directly S3 mein store karo.

#### Step 1: Install AWS SDK

```bash
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
```

#### Step 2: Add Environment Variables

`.env` file mein:

```env
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
S3_BUCKET_NAME=your-app-name-uploads
```

> ⚠️ **Never commit `.env` to GitHub!** Add it to `.gitignore`.
> These keys come from IAM — see `IAM.md` for how to create them.

#### Step 3: Create S3 Helper (s3.service.ts)

```typescript
import {
  S3Client,
  PutObjectCommand,
  DeleteObjectCommand,
  GetObjectCommand,
} from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

// Initialize S3 client
const s3Client = new S3Client({
  region: process.env.AWS_REGION!,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
  },
});

const BUCKET_NAME = process.env.S3_BUCKET_NAME!;

// ─── Upload a file ───────────────────────────────────────────────
export async function uploadToS3(
  fileBuffer: Buffer,
  fileName: string,
  mimeType: string,
  folder: string = "uploads"
): Promise<string> {
  const key = `${folder}/${Date.now()}-${fileName}`;

  await s3Client.send(
    new PutObjectCommand({
      Bucket: BUCKET_NAME,
      Key: key,
      Body: fileBuffer,
      ContentType: mimeType,
    })
  );

  // Return the S3 URL
  return `https://${BUCKET_NAME}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`;
}

// ─── Delete a file ───────────────────────────────────────────────
export async function deleteFromS3(fileUrl: string): Promise<void> {
  // Extract key from URL
  const key = fileUrl.split(".amazonaws.com/")[1];

  await s3Client.send(
    new DeleteObjectCommand({
      Bucket: BUCKET_NAME,
      Key: key,
    })
  );
}

// ─── Generate Pre-signed URL (for private files) ─────────────────
// This gives a temporary URL (expires in X seconds) to access private files
export async function getPresignedUrl(
  key: string,
  expiresIn: number = 3600 // 1 hour
): Promise<string> {
  const command = new GetObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key,
  });

  return getSignedUrl(s3Client, command, { expiresIn });
}
```

#### Step 4: Use in Express Route with Multer

```bash
npm install multer @types/multer
```

```typescript
import express from "express";
import multer from "multer";
import { uploadToS3, deleteFromS3 } from "./s3.service";

const router = express.Router();

// Use memory storage — we'll send buffer directly to S3
const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB limit
  fileFilter: (req, file, cb) => {
    if (file.mimetype.startsWith("image/")) {
      cb(null, true);
    } else {
      cb(new Error("Only images are allowed"));
    }
  },
});

// POST /upload — upload image to S3
router.post("/upload", upload.single("image"), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: "No file uploaded" });
    }

    const url = await uploadToS3(
      req.file.buffer,
      req.file.originalname,
      req.file.mimetype,
      "avatars" // folder inside bucket
    );

    res.json({ success: true, url });
  } catch (error) {
    res.status(500).json({ error: "Upload failed" });
  }
});

// DELETE /upload — delete file from S3
router.delete("/upload", async (req, res) => {
  try {
    const { fileUrl } = req.body;
    await deleteFromS3(fileUrl);
    res.json({ success: true, message: "File deleted" });
  } catch (error) {
    res.status(500).json({ error: "Delete failed" });
  }
});

export default router;
```

---

## Use Case 3: Pre-signed URLs (Private File Access)

Kuch files private rakho — user ko direct URL mat do. Pre-signed URL generate karo jo **expire ho jaye**:

```typescript
// User requests access to their private document
router.get("/document/:key", async (req, res) => {
  try {
    // Generate URL valid for 15 minutes only
    const url = await getPresignedUrl(req.params.key, 900);
    res.json({ url, expiresIn: "15 minutes" });
  } catch (error) {
    res.status(500).json({ error: "Could not generate access URL" });
  }
});
```

---

## CORS Configuration (Important for Frontend Uploads)

Agar frontend directly S3 par upload karta hai, CORS configure karna padega:

S3 Bucket → **Permissions** → **Cross-origin resource sharing (CORS)** → Edit:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
    "AllowedOrigins": [
      "http://localhost:3000",
      "https://yourdomain.com"
    ],
    "ExposeHeaders": ["ETag"]
  }
]
```

---

## S3 Bucket Structure (Best Practice)

```
your-app-name-uploads/
├── avatars/
│   └── 1703123456789-profile.jpg
├── documents/
│   └── 1703123456789-resume.pdf
├── products/
│   └── 1703123456789-product-image.jpg
└── backups/
    └── 20240101_120000-backup.tar.gz
```

---

## Quick Reference

```bash
# AWS CLI commands (useful for CI/CD)

# Upload folder to S3
aws s3 sync ./dist s3://bucket-name --delete

# Upload single file
aws s3 cp ./file.txt s3://bucket-name/path/file.txt

# List bucket contents
aws s3 ls s3://bucket-name/

# Delete file
aws s3 rm s3://bucket-name/path/file.txt

# Download file
aws s3 cp s3://bucket-name/path/file.txt ./local-file.txt
```

---

> 💡 **Next Step:** S3 ke liye proper credentials chahiye — unke liye **IAM.md** padho!
