```markdown
# Mavica Gallery — S3/MinIO Setup Guide

Complete reference for connecting the Mavica Gallery frontend to a MinIO S3-compatible bucket via a Node.js API backend.

---

## Architecture

```
Browser (gallery)  →  API Backend (Node.js)  →  MinIO (S3-compatible)
   GET /images          list objects              ListBucket
   POST /upload-url     presign PUT               PutObject
   PUT presigned URL  ───────────────────────→  direct upload to MinIO
```

The browser never touches S3 credentials. The backend generates short-lived presigned URLs for each upload. Images are served directly from MinIO via a public-read bucket policy.

---

## 1. Create the MinIO Bucket

### Install MinIO Client

```bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/
```

### Configure Alias

Replace `YOUR_PROXY_URL`, `YOUR_ACCESS_KEY`, and `YOUR_SECRET_KEY` with your actual values:

```bash
mc alias set myminio https://your-proxy-url \
  YOUR_MINIO_ACCESS_KEY \
  YOUR_MINIO_SECRET_KEY
```

### Create Bucket

```bash
mc mb myminio/mavica-gallery
```

### Set Public Read Policy

This allows images to be loaded directly in `<img>` tags without authentication:

```bash
mc anonymous set download myminio/mavica-gallery
```

Or as explicit JSON:

```bash
mc anonymous set-json myminio/mavica-gallery << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::mavica-gallery/*"]
    }
  ]
}
EOF
```

### Verify

```bash
mc ls myminio/mavica-gallery
mc anonymous get myminio/mavica-gallery
```

---

## 2. Create the API Backend

### Project Setup

```bash
mkdir -p /opt/mavica-api
cd /opt/mavica-api
```

### package.json

```json
{
  "name": "mavica-api",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "@aws-sdk/client-s3": "^3.700.0",
    "@aws-sdk/s3-request-presigner": "^3.700.0",
    "express": "^4.21.0",
    "cors": "^2.8.5"
  }
}
```

### server.js

```javascript
import express from 'express';
import cors from 'cors';
import { S3Client, ListObjectsV2Command } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { PutObjectCommand } from '@aws-sdk/client-s3';

// ─── Configuration ──────────────────────────────────────
const PORT = process.env.PORT || 3090;

const S3 = new S3Client({
  region: 'us-east-1',
  endpoint: process.env.S3_ENDPOINT,
  credentials: {
    accessKeyId: process.env.S3_ACCESS_KEY,
    secretAccessKey: process.env.S3_SECRET_KEY,
  },
  forcePathStyle: true,
});

const BUCKET = process.env.S3_BUCKET || 'mavica-gallery';
const PUBLIC_URL = process.env.PUBLIC_URL;

// ─── Express App ────────────────────────────────────────
const app = express();
app.use(cors());
app.use(express.json());

// Health check
app.get('/', (req, res) => res.json({ status: 'ok', bucket: BUCKET }));

// ─── List Images ────────────────────────────────────────
// GET /images
// Returns: { images: [{ url, filename, size, lastModified }] }
app.get('/images', async (req, res) => {
  try {
    const command = new ListObjectsV2Command({
      Bucket: BUCKET,
      MaxKeys: 1000,
    });
    const data = await S3.send(command);

    const images = (data.Contents || []).map(obj => ({
      url: `${PUBLIC_URL}/${BUCKET}/${encodeURIComponent(obj.Key)}`,
      filename: obj.Key.split('/').pop(),
      size: obj.Size,
      lastModified: obj.LastModified,
    }));

    images.sort((a, b) => b.lastModified - a.lastModified);

    res.json({ images });
  } catch (err) {
    console.error('List failed:', err);
    res.status(500).json({ error: err.message });
  }
});

// ─── Presigned Upload URL ───────────────────────────────
// POST /upload-url
// Body: { filename, contentType }
// Returns: { presignedUrl, fileUrl, key }
app.post('/upload-url', async (req, res) => {
  try {
    const { filename, contentType } = req.body;
    if (!filename) return res.status(400).json({ error: 'filename required' });

    const safeName = filename.replace(/[/\\]/g, '_').replace(/\s+/g, '-');
    const now = new Date();
    const prefix = `${now.getFullYear()}-${String(now.getMonth()+1).padStart(2,'0')}`;
    const key = `${prefix}/${Date.now()}-${safeName}`;

    const command = new PutObjectCommand({
      Bucket: BUCKET,
      Key: key,
      ContentType: contentType || 'image/jpeg',
    });

    const presignedUrl = await getSignedUrl(S3, command, { expiresIn: 300 });

    res.json({
      presignedUrl,
      fileUrl: `${PUBLIC_URL}/${BUCKET}/${encodeURIComponent(key)}`,
      key,
    });
  } catch (err) {
    console.error('Presign failed:', err);
    res.status(500).json({ error: err.message });
  }
});

// ─── Start ──────────────────────────────────────────────
app.listen(PORT, () => {
  console.log(`Mavica API running on port ${PORT}`);
  console.log(`Bucket: ${BUCKET}`);
  console.log(`Public URL: ${PUBLIC_URL}`);
});
```

### .env

Create this file in `/opt/mavica-api/.env`. Do not commit to version control.

```bash
# Internal MinIO address (what the server sees directly)
S3_ENDPOINT=http://127.0.0.1:9000

# Your MinIO credentials
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadminsecret

# Bucket name
S3_BUCKET=mavica-gallery

# Public URL browsers use to fetch images (your proxy URL)
PUBLIC_URL=https://minio.yourdomain.com

# API port
PORT=3090
```

### Install and Run

```bash
cd /opt/mavica-api
npm install
npm start
```

---

## 3. Reverse Proxy Configuration

Your proxy needs two distinct paths:

- **MinIO path** — serves image files directly (what `PUBLIC_URL` points to)
- **API path** — routes to the Node.js backend

### nginx Example

```nginx
# MinIO — serves the actual image files
# This is what PUBLIC_URL points to
location / {
    proxy_pass http://127.0.0.1:9000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Cache images aggressively
    proxy_cache_valid 200 30d;
    add_header Cache-Control "public, max-age=2592000, immutable";
}

# MinIO console (optional, for visual bucket browsing)
location /minio/ {
    proxy_pass http://127.0.0.1:9000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}

# Mavica API backend
location /api/ {
    proxy_pass http://127.0.0.1:3090/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Optional: rate limit
    # limit_req zone=api burst=10 nodelay;
}
```

### Caddy Example

```
minio.yourdomain.com {
    # API backend
    handle /api/* {
        reverse_proxy localhost:3090
    }

    # Everything else → MinIO
    handle {
        reverse_proxy localhost:9000
    }
}
```

---

## 4. Run as a Systemd Service

Create `/etc/systemd/system/mavica-api.service`:

```ini
[Unit]
Description=Mavica Gallery API
After=network.target

[Service]
Type=simple
User=nobody
WorkingDirectory=/opt/mavica-api
EnvironmentFile=/opt/mavica-api/.env
ExecStart=/usr/bin/node /opt/mavica-api/server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mavica-api
sudo systemctl start mavica-api
sudo systemctl status mavica-api
```

Useful commands:

```bash
# View logs
sudo journalctl -u mavica-api -f

# Restart after changes
sudo systemctl restart mavica-api
```

---

## 5. Connect the Gallery Frontend

1. Open the Mavica Gallery page in your browser
2. Click the gear icon (top-right corner)
3. Fill in the fields:

| Field | Value |
|---|---|
| **Upload API Endpoint** | `https://your-proxy-url/api/upload-url` |
| **Gallery List Endpoint** | `https://your-proxy-url/api/images` |

4. Click **SAVE CONFIG**
5. Click **TEST** — should show a success toast
6. The status indicator in the hero section should change from amber `LOCAL` to green `S3`

---

## 6. Verification

Test each endpoint from the command line:

```bash
# Health check
curl https://your-proxy-url/api/
# Expected: {"status":"ok","bucket":"mavica-gallery"}

# List images (empty bucket returns empty array)
curl https://your-proxy-url/api/images
# Expected: {"images":[]}

# Request a presigned upload URL
curl -X POST https://your-proxy-url/api/upload-url \
  -H "Content-Type: application/json" \
  -d '{"filename":"test.jpg","contentType":"image/jpeg"}'
# Expected: {"presignedUrl":"https://...","fileUrl":"https://...","key":"2025-01/...-test.jpg"}

# Actually upload a file using the presigned URL
PRESIGNED_URL=$(curl -s -X POST https://your-proxy-url/api/upload-url \
  -H "Content-Type: application/json" \
  -d '{"filename":"test.jpg","contentType":"image/jpeg"}' | jq -r '.presignedUrl')

curl -X PUT "$PRESIGNED_URL" \
  -H "Content-Type: image/jpeg" \
  --data-binary @/path/to/some/photo.jpg

# Verify it appears in the list
curl https://your-proxy-url/api/images
# Expected: {"images":[{"url":"https://...","filename":"2025-01/...-test.jpg",...}]}
```

Then test in the gallery: drop an image, watch the progress bar complete, confirm it appears in the grid. Refresh the page — images should reload from S3.

---

## 7. File Organization

Uploaded files are automatically organized into date-prefixed folders:

```
mavica-gallery/
├── 2025-01/
│   ├── 1706200000000-sunset.jpg
│   ├── 1706200100000-cathedral.png
│   └── 1706200200000-street-photo.jpg
├── 2025-02/
│   ├── 1706700000000-portrait.jpg
│   └── ...
└── ...
```

Each filename is prefixed with a Unix timestamp to prevent collisions.

---

## 8. Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Status stays `LOCAL` | API fields empty or not saved | Re-enter URLs, click Save, check localStorage |
| `Presign request failed: 500` | Backend can't reach MinIO | Check `S3_ENDPOINT` in `.env` — must be the internal address (`127.0.0.1:9000`), not the proxy URL |
| Images upload but don't display | `PUBLIC_URL` is wrong | Must be the proxy URL that browsers can reach, not `127.0.0.1` |
| `403 Forbidden` on image load | Bucket policy not set | Run `mc anonymous set download myminio/mavica-gallery` |
| CORS error in browser | Missing CORS headers | Backend includes `cors()` — if proxying differently, add CORS at the proxy level |
| Upload reaches 100% but no image | Presigned URL expired | Default is 5 minutes — increase `expiresIn` in `getSignedUrl()` if needed |
| `SignatureDoesNotMatch` | URL encoding mismatch | Ensure `forcePathStyle: true` is set in the S3 client config |
| `NetworkingError` on presign | DNS or firewall | Verify the server can reach MinIO: `curl http://127.0.0.1:9000/minio/health/live` |
| Gallery loads but images are broken | Mixed content (HTTP vs HTTPS) | Ensure `PUBLIC_URL` uses `https://` if the gallery is served over HTTPS |
| `RequestTimeout` on list | Too many objects | Increase `MaxKeys` or implement pagination |

---

## 9. Security Notes

- **Credentials never leave the server.** The browser only sees presigned URLs with a 5-minute expiry window.
- **Presigned URLs are scoped.** Each URL only permits a single `PUT` operation on one specific key — it cannot list, delete, or overwrite other files.
- **Public read is intentional.** The bucket policy allows `s3:GetObject` for anyone so images load in `<img>` tags without auth. Write operations remain restricted to the backend's credentials.
- **If you need fully private images**, you would need a second backend endpoint that generates presigned GET URLs and either redirects to them or proxies the image through the backend. This adds latency and complexity.
- **Lock down the API endpoint** in production:

```nginx
location /api/ {
    # Option A: restrict by IP
    # allow 192.168.0.0/16;
    # deny all;

    # Option B: basic auth
    # auth_basic "Mavica API";
    # auth_basic_user_file /etc/nginx/.mavica_htpasswd;

    # Option C: rate limit
    # limit_req zone=api burst=10 nodelay;

    proxy_pass http://127.0.0.1:3090/;
}
```

- **Rotate credentials** periodically via MinIO:
```bash
mc admin user svcacct add myminio your-user --access-key NEW_KEY --secret-key NEW_SECRET
```

---

## 10. Backup and Maintenance

### Backup the Bucket

```bash
# Full backup to local directory
mc mirror myminio/mavica-gallery /backup/mavica-gallery-$(date +%Y%m%d)/

# Sync to another MinIO instance
mc mirror myminio/mavica-gallery myminio-backup/mavica-gallery
```

### Backup the API

```bash
# The entire API is just two files
cp /opt/mavica-api/server.js /backup/mavica-api/
cp /opt/mavica-api/.env /backup/mavica-api/
cp /opt/mavica-api/package.json /backup/mavica-api/
```

### Cleanup Old Files

```bash
# List files older than 90 days
mc ls --recursive myminio/mavica-gallery | while read line; do
    # Parse and delete logic here, or use mc rm with --older-than
done

# MinIO ILM policy for automatic cleanup (if configured)
mc ilm add --expire-days 365 myminio/mavica-gallery
```

---

## Quick Reference Card

```
Bucket:          mavica-gallery
API Port:        3090
Upload Endpoint: POST /api/upload-url  →  { presignedUrl, fileUrl, key }
List Endpoint:   GET  /api/images      →  { images: [{ url, filename }] }
Image Base URL:  PUBLIC_URL/bucket/key
File Layout:     YYYY-MM/timestamp-filename.jpg
Presign TTL:     300 seconds (5 minutes)
```
```

Save this as `MAVICA-S3-SETUP.md` wherever you keep your project documentation.
