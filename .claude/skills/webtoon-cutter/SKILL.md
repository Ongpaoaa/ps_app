---
name: webtoon-cutter
description: Cut long webtoon images into fixed-height chunks, upload to Backblaze B2 storage, and generate DB-ready metadata for hackathon activities. Use when you have a tall webtoon image that needs to be sliced into mobile-friendly vertical chunks.
---

# Webtoon Cutter

## Overview

Cut long webtoon images into fixed-height chunks, upload to Backblaze B2 (S3-compatible) storage, and generate DB-ready metadata for hackathon activities.

## Prerequisites

- ImageMagick installed (`brew install imagemagick` on macOS)
- Backblaze B2 credentials (B2_APPLICATION_KEY_ID, B2_APPLICATION_KEY)
- B2 bucket configured (pseed-dev)
- Source image file (PNG recommended)

## Workflow

### 1. Gather inputs

Ask user for:
- Source image path (e.g., `/path/to/phase1-act1.png`)
- Story ID (e.g., `phase1-act1`)
- Chunk height (default: 1280px)

### 2. Analyze source image

```bash
identify -format "%w %h" {source-path}
```

Parse output to get width and height.

### 3. Calculate chunks

- Number of chunks = ceil(height / chunk_height)
- Last chunk height = height - (chunks - 1) * chunk_height

### 4. Cut into chunks

For evenly divisible height:
```bash
convert {source} -crop {width}x{chunk_height} +repage +adjoin {story-id}-%02d.png
```

For uneven height, cut each chunk with offset:
```bash
convert {source} -crop {width}x{chunk_height}+0+0 {story-id}-01.png
convert {source} -crop {width}x{chunk_height}+0+{chunk_height} {story-id}-02.png
# ... continue for each chunk
convert {source} -crop {width}x{last_chunk_height}+0+{offset} {story-id}-{last}.png
```

### 5. Upload to Backblaze B2

Use the AWS S3 CLI or SDK with B2 credentials:

```bash
# Using AWS CLI with B2 endpoint
aws s3 cp {local-path} \
  s3://{bucket-name}/webtoons/{story-id}/{filename} \
  --endpoint-url https://{b2-endpoint} \
  --access-key {b2-key-id} \
  --secret-key {b2-application-key}
```

Or use the Node.js script:

```bash
npx tsx scripts/upload-webtoon-to-b2.ts \
  --story-id={story-id} \
  --chunks={chunk-files}
```

Public URL format:
```
https://s3.us-east-005.backblazeb2.com/pseed-dev/webtoons/{story-id}/{filename}
```

### 6. Generate DB-ready payload

Construct JSON payload matching webtoon schema:

```json
{
  "content_type": "webtoon",
  "metadata": {
    "variant": "webtoon",
    "originalWidth": {width},
    "originalHeight": {height},
    "panelWidth": {width},
    "panelHeight": {chunk_height},
    "chunks": [
      { "id": "c1", "order": 1, "imageKey": "{story-id}-01", "imageUrl": "https://s3.us-east-005.backblazeb2.com/pseed-dev/webtoons/{story-id}/{story-id}-01.png" },
      { "id": "c2", "order": 2, "imageKey": "{story-id}-02", "imageUrl": "https://s3.us-east-005.backblazeb2.com/pseed-dev/webtoons/{story-id}/{story-id}-02.png" }
    ]
  }
}
```

### 7. Output result

Return:
- List of uploaded URLs
- Full DB-ready payload
- Optional: Patch live DB if user requests

## Commands Reference

### ImageMagick

```bash
# Get dimensions
identify -format "%w %h" image.png

# Crop evenly
convert image.png -crop 720x1280 +repage +adjoin output_%02d.png

# Crop with offset
convert image.png -crop 720x1280+0+2560 output_03.png
```

### Backblaze B2 Upload

```bash
# Using curl with S3-compatible API
curl -X PUT "https://s3.us-east-005.backblazeb2.com/pseed-dev/webtoons/{story-id}/{filename}" \
  -H "Authorization: Bearer {b2-token}" \
  -H "Content-Type: image/png" \
  --data-binary "@{local-path}"

# Using AWS CLI
aws s3 cp {local-path} s3://pseed-dev/webtoons/{story-id}/{filename} \
  --endpoint-url https://s3.us-east-005.backblazeb2.com
```

### Node.js Upload Script

```typescript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { readFileSync } from 'fs';

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
  Key: `webtoons/${storyId}/${filename}`,
  Body: readFileSync(localPath),
  ContentType: 'image/png',
}));
```

## Environment Variables

Required:
- `B2_APPLICATION_KEY_ID` - B2 application key ID
- `B2_APPLICATION_KEY` - B2 application key (secret)
- `B2_BUCKET_NAME` - Bucket name (default: pseed-dev)
- `B2_ENDPOINT` - S3 endpoint (default: s3.us-east-005.backblazeb2.com)

Optional:
- `SUPABASE_SERVICE_ROLE_KEY` - For writing back to database
- `EXPO_PUBLIC_SUPABASE_URL` - Supabase project URL

## Failure Modes

- **ImageMagick not found**: Guide user to install with `brew install imagemagick`
- **Upload fails**: Check B2 credentials, verify bucket exists
- **Uneven chunks**: Handle last chunk with remaining height
- **DB write fails**: Verify activity ID, check Supabase permissions

## Migration Note

All existing webtoons have been migrated from Supabase Storage to Backblaze B2. Use this skill for NEW webtoons only. To migrate old webtoons, use `scripts/migrate-webtoons-to-b2.ts`.

## References

- [references/b2-storage.md](references/b2-storage.md) - B2 setup and upload patterns
- [references/db-payload.md](references/db-payload.md) - Payload structure and writeback workflow
