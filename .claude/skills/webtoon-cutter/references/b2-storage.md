# Backblaze B2 Storage Setup

## Configuration

Backblaze B2 is used for webtoon image storage to reduce Supabase egress costs.

### Required Environment Variables

```bash
B2_APPLICATION_KEY_ID=your-key-id
B2_APPLICATION_KEY=your-application-key
B2_BUCKET_NAME=pseed-dev
B2_ENDPOINT=s3.us-east-005.backblazeb2.com
B2_BUCKET_ID=your-bucket-id
```

### S3-Compatible API

B2 uses AWS S3-compatible API. Configure AWS SDK or CLI:

```bash
# AWS CLI config
aws configure set aws_access_key_id $B2_APPLICATION_KEY_ID
aws configure set aws_secret_access_key $B2_APPLICATION_KEY

# Use with endpoint
aws s3 ls s3://pseed-dev --endpoint-url https://s3.us-east-005.backblazeb2.com
```

## Upload Methods

### Method 1: AWS SDK (Node.js)

```typescript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

const s3 = new S3Client({
  endpoint: 'https://s3.us-east-005.backblazeb2.com',
  region: 'us-east-005',
  credentials: {
    accessKeyId: process.env.B2_APPLICATION_KEY_ID!,
    secretAccessKey: process.env.B2_APPLICATION_KEY!,
  },
});

await s3.send(new PutObjectCommand({
  Bucket: 'pseed-dev',
  Key: 'webtoons/phase1-act5/phase1-act5-01.png',
  Body: fileBuffer,
  ContentType: 'image/png',
}));
```

### Method 2: AWS CLI

```bash
aws s3 cp ./phase1-act5-01.png \
  s3://pseed-dev/webtoons/phase1-act5/ \
  --endpoint-url https://s3.us-east-005.backblazeb2.com
```

### Method 3: curl (S3 API)

```bash
curl -X PUT \
  "https://s3.us-east-005.backblazeb2.com/pseed-dev/webtoons/phase1-act5/phase1-act5-01.png" \
  -H "Content-Type: image/png" \
  -H "Authorization: AWS ${B2_APPLICATION_KEY_ID}:${SIGNATURE}" \
  --data-binary "@./phase1-act5-01.png"
```

## Public URL Format

```
https://s3.us-east-005.backblazeb2.com/pseed-dev/webtoons/{story-id}/{filename}
```

Example:
```
https://s3.us-east-005.backblazeb2.com/pseed-dev/webtoons/phase1-act1/phase1-act1-00.png
```

## Folder Structure

```
pseed-dev/
└── webtoons/
    ├── phase1-act1/
    │   ├── phase1-act1-00.png
    │   ├── phase1-act1-01.png
    │   └── ...
    ├── phase1-act2/
    │   └── ...
    └── phase1-act3/
        └── ...
```

## Cost Comparison

| Service | Storage | Egress | Notes |
|---------|---------|--------|-------|
| Supabase | $0.021/GB | $0.09/GB | 250GB included |
| Backblaze B2 | $0.005/GB | $0.01/GB | 1GB/day free egress |

**Savings: ~90% on egress costs**

## Migration Script

To migrate existing webtoons from Supabase to B2:

```bash
# Export B2 credentials
export B2_APPLICATION_KEY_ID=...
export B2_APPLICATION_KEY=...

# Run migration
npx tsx scripts/migrate-webtoons-to-b2.ts
```

## Troubleshooting

- **403 Forbidden**: Check credentials and bucket permissions
- **Slow uploads**: B2 is optimized for cost, not speed. Use multipart for large files.
- **CORS issues**: Configure CORS on B2 bucket if loading from web
