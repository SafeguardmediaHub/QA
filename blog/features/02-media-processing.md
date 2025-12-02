# Feature 02: Media Upload & Processing Pipeline
## Comprehensive Testing Documentation

---

**[← Back to Authentication](./01-authentication.md)** | **[Home](../README.md)** | **[Next: Media Management →](./03-media-management.md)**

---

## Table of Contents

1. [Feature Overview](#feature-overview)
2. [Test Scope](#test-scope)
3. [Upload Flow Architecture](#upload-flow-architecture)
4. [API Endpoints](#api-endpoints)
5. [Test Scenarios](#test-scenarios)
6. [Detailed Test Cases](#detailed-test-cases)
7. [Edge Cases & Boundary Testing](#edge-cases--boundary-testing)
8. [Security Testing](#security-testing)
9. [Performance Considerations](#performance-considerations)
10. [Test Results](#test-results)
11. [Issues Found](#issues-found)
12. [Recommendations](#recommendations)

---

## Feature Overview

### Description

The Media Upload & Processing Pipeline is the core feature that handles secure file uploads, automatic metadata extraction, thumbnail generation, content hashing, and file validation. It implements a sophisticated three-stage upload flow using AWS S3 presigned URLs for direct client-to-S3 uploads, followed by server-side confirmation and processing.

### Importance

This is **the foundation of the entire platform**. All verification, analysis, and detection features depend on successfully uploaded and processed media. Issues here cascade to:
- Timeline verification (requires metadata)
- Geolocation verification (requires GPS data)
- Deepfake detection (requires processed media)
- C2PA verification (requires media files)
- All analysis features

### Key Features

#### Upload Methods
- ✅ **Direct S3 Upload** - Presigned URL for client-side upload
- ✅ **URL Upload** - Server downloads from external URL then uploads to S3

#### Automatic Processing
- ✅ **Metadata Extraction** - EXIF, IPTC, XMP, GPS (images), codec info (video/audio)
- ✅ **Thumbnail Generation** - Automatic for videos
- ✅ **Content Hashing** - MD5, SHA256, perceptual hashing
- ✅ **Tamper Detection** - Integrity scoring, authenticity analysis

#### File Management
- ✅ **File Validation** - Type, size, signature verification
- ✅ **Quota Management** - Per-user storage limits
- ✅ **Visibility Control** - Public, private, unlisted
- ✅ **Batch Processing** - Video batch processing for admins

#### Supported Media Types
- **Images**: JPG, JPEG, PNG, WEBP, GIF, BMP, TIFF, AVIF, HEIC
- **Videos**: MP4, MOV, AVI, MKV, WEBM, M4V
- **Audio**: MP3, WAV, AAC, FLAC, OGG, M4A
- **Documents**: PDF, DOCX, DOC (if configured)

---

## Test Scope

### In Scope ✅

**Upload Flow:**
- Presigned URL generation
- Direct S3 upload (simulated/verified via confirmation)
- Upload confirmation
- URL-based upload

**Processing Pipeline:**
- Automatic metadata extraction on upload confirmation
- Thumbnail generation for videos
- Content hash calculation (MD5, SHA256)
- Tamper detection analysis
- Queue-based background processing

**Validation:**
- File type validation (MIME type + file signature)
- File size limits (per type)
- Storage quota checking
- Upload type inference
- URL validation (for URL uploads)

**Error Handling:**
- Quota exceeded scenarios
- Invalid file types
- Oversized files
- Missing/corrupted files
- S3 upload failures
- Processing failures

### Out of Scope ❌

- Actual S3 upload (client-side, tested separately)
- Image editing/manipulation
- Video transcoding (only thumbnail extraction)
- Client-side upload libraries
- Network bandwidth testing
- CDN performance

### Dependencies

- ✅ Authentication - User must be logged in
- ✅ Email Verification - Required for uploads
- ✅ AWS S3 - File storage
- ✅ MongoDB - Media records
- ✅ Redis - Queue management
- ✅ FFmpeg - Video processing
- ✅ ExifTool/Sharp - Metadata extraction

---

## Upload Flow Architecture

### Standard Upload Flow (3-Step Process)

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ 1. Request Presigned URL
       │    POST /api/media/presigned-url
       │    { filename, contentType, fileSize, uploadType }
       ▼
┌──────────────────────┐
│   SafeguardMedia API │
│                      │
│  ✓ Validate user     │
│  ✓ Check quota       │
│  ✓ Validate file     │
│  ✓ Generate S3 key   │
└──────┬───────────────┘
       │ 2. Returns Presigned URL
       │    { uploadUrl, s3Key, correlationId, expiresAt }
       ▼
┌─────────────┐
│   Client    │
│             │
│  Uploads    │──────────────────────┐
│  directly   │                      │ 3. Direct Upload
│  to S3      │                      ▼
└──────┬──────┘              ┌──────────────┐
       │                     │   AWS S3     │
       │                     └──────────────┘
       │ 4. Confirm Upload
       │    POST /api/media/confirm-upload
       │    { s3Key, correlationId }
       ▼
┌──────────────────────────────────┐
│   SafeguardMedia API             │
│                                  │
│  ✓ Verify file exists in S3     │
│  ✓ Validate file signature       │
│  ✓ Create media record           │
│  ✓ Queue metadata extraction     │
│  ✓ Queue thumbnail generation    │
│  ✓ Update user quota             │
└──────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────┐
│   Background Processing Queue    │
│                                  │
│  → Extract metadata (EXIF, etc.) │
│  → Generate content hashes       │
│  → Detect tampering              │
│  → Generate video thumbnail      │
│  → Update media status           │
└──────────────────────────────────┘
```

### URL Upload Flow (Simplified)

```
Client → POST /api/media/upload-url { url }
         ↓
    Server downloads from URL
         ↓
    Server uploads to S3
         ↓
    Automatic confirmation & processing
         ↓
    Returns media record
```

---

## API Endpoints

### Base URL

```
http://localhost:3000/api/media
```

### Endpoint Summary

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/presigned-url` | POST | Yes | Generate presigned URL for direct S3 upload |
| `/confirm-upload` | POST | Yes | Confirm successful S3 upload and trigger processing |
| `/upload-url` | POST | Yes | Upload media from external URL |
| `/config` | GET | Yes | Get upload configuration and limits |
| `/stats` | GET | Yes | Get user's media statistics |
| `/:id/thumbnail` | POST | Yes | Manually trigger video thumbnail generation |
| `/video/batch-process` | POST | Admin | Batch process multiple videos |

---

## Detailed Endpoint Documentation

### 1. Generate Presigned URL

#### Endpoint

```
POST /api/media/presigned-url
```

#### Description

Generates a presigned URL for direct client-to-S3 upload. This is step 1 of the standard upload flow. The presigned URL expires in 15 minutes.

#### Request Headers

```
Content-Type: application/json
Authorization: Bearer <access_token>
```

#### Request Body

```json
{
  "filename": "vacation-photo.jpg",
  "contentType": "image/jpeg",
  "fileSize": 2048576,
  "uploadType": "general_image",
  "visibility": "private",
  "metadata": {
    "description": "Beach vacation 2025",
    "tags": ["vacation", "beach"]
  }
}
```

**Field Specifications**:

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `filename` | string | Yes | Max 255 chars | Original filename |
| `contentType` | string | Yes | Valid MIME type | File MIME type |
| `fileSize` | number | Yes | > 0, within limits | File size in bytes |
| `uploadType` | string | Yes | Valid UploadType enum | Type of upload |
| `visibility` | string | No | public/private/unlisted | Default: private |
| `metadata` | object | No | Valid JSON | Additional metadata |

**Upload Types**:
- `general_image` - General images
- `video` - Video files
- `audio` - Audio files
- `document` - Documents (if enabled)
- `profile_image` - Profile pictures

**File Size Limits** (Default):
- Images: 10 MB
- Videos: 500 MB
- Audio: 50 MB
- Documents: 25 MB

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "Presigned URL generated successfully",
  "data": {
    "uploadUrl": "https://bucket.s3.amazonaws.com/media/user123/uuid.jpg?X-Amz-Algorithm=...",
    "s3Key": "media/user123/abc-def-ghi-jkl.jpg",
    "correlationId": "550e8400-e29b-41d4-a716-446655440000",
    "expiresAt": "2025-12-02T11:00:00.000Z",
    "maxFileSize": 10485760,
    "allowedContentTypes": ["image/jpeg", "image/png", "image/webp"]
  },
  "timestamp": "2025-12-02T10:45:00.000Z"
}
```

**Response Fields**:
- `uploadUrl`: Presigned S3 URL (valid for 15 minutes)
- `s3Key`: S3 object key for confirmation
- `correlationId`: Unique ID to correlate confirmation request
- `expiresAt`: URL expiration timestamp
- `maxFileSize`: Maximum allowed file size
- `allowedContentTypes`: List of accepted MIME types

#### Error Responses

**403 Forbidden - Quota Exceeded**

```json
{
  "success": false,
  "message": "Media quota exceeded. Cannot upload more files.",
  "timestamp": "2025-12-02T10:45:00.000Z"
}
```

**400 Bad Request - File Too Large**

```json
{
  "success": false,
  "message": "File validation failed: File size exceeds maximum allowed size of 10485760 bytes",
  "timestamp": "2025-12-02T10:45:00.000Z"
}
```

**400 Bad Request - Invalid File Type**

```json
{
  "success": false,
  "message": "File validation failed: Content type application/x-executable is not allowed for upload type general_image",
  "data": {
    "allowedTypes": ["image/jpeg", "image/png", "image/gif", "image/webp"]
  },
  "timestamp": "2025-12-02T10:45:00.000Z"
}
```

**403 Forbidden - Storage Quota Exceeded**

```json
{
  "success": false,
  "message": "Upload would exceed storage quota: used=9500000000, limit=10737418240, requested=2048576",
  "timestamp": "2025-12-02T10:45:00.000Z"
}
```

#### Side Effects

1. Correlation ID stored temporarily (Redis/memory)
2. User's pending upload count incremented
3. Presigned URL generated with 15-minute expiration
4. S3 key reserved (not yet used)

---

### 2. Confirm Upload

#### Endpoint

```
POST /api/media/confirm-upload
```

#### Description

Confirms that the client successfully uploaded the file to S3. This triggers server-side validation, media record creation, and queues background processing (metadata extraction, thumbnail generation, hashing).

#### Request Headers

```
Content-Type: application/json
Authorization: Bearer <access_token>
```

#### Request Body

```json
{
  "s3Key": "media/user123/abc-def-ghi-jkl.jpg",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Field Specifications**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `s3Key` | string | Yes | S3 key from presigned URL response |
| `correlationId` | string | Yes | Correlation ID from presigned URL response |

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "Upload confirmed successfully",
  "data": {
    "media": {
      "id": "507f1f77bcf86cd799439011",
      "filename": "vacation-photo.jpg",
      "fileSize": 2048576,
      "mimeType": "image/jpeg",
      "uploadType": "general_image",
      "status": "processing",
      "visibility": "private",
      "publicUrl": null,
      "uploadedAt": "2025-12-02T10:45:30.000Z",
      "metadata": {
        "description": "Beach vacation 2025",
        "tags": ["vacation", "beach"],
        "uploadMethod": "direct",
        "processingQueued": true
      }
    }
  },
  "timestamp": "2025-12-02T10:45:30.000Z"
}
```

**Status Values**:
- `pending` - Upload initiated but not confirmed
- `uploaded` - Upload confirmed, not yet processed
- `processing` - Background processing in progress
- `processed` - All processing complete
- `failed` - Processing failed

#### Error Responses

**400 Bad Request - Missing Fields**

```json
{
  "success": false,
  "message": "Missing required fields: s3Key, correlationId",
  "timestamp": "2025-12-02T10:45:30.000Z"
}
```

**404 Not Found - File Not in S3**

```json
{
  "success": false,
  "message": "File not found in S3. Upload may have failed.",
  "timestamp": "2025-12-02T10:45:30.000Z"
}
```

**400 Bad Request - Invalid File Signature**

```json
{
  "success": false,
  "message": "File signature validation failed. File may be corrupted or type mismatch.",
  "data": {
    "expectedType": "image/jpeg",
    "detectedType": "application/octet-stream"
  },
  "timestamp": "2025-12-02T10:45:30.000Z"
}
```

**400 Bad Request - Invalid Correlation ID**

```json
{
  "success": false,
  "message": "Invalid correlation ID or expired upload session",
  "timestamp": "2025-12-02T10:45:30.000Z"
}
```

#### Side Effects

1. **Media Record Created** in MongoDB
2. **S3 Object Verified** (HEAD request to confirm existence)
3. **File Signature Validated** (first bytes checked against MIME type)
4. **User Quota Updated** (storage used incremented)
5. **Background Jobs Queued**:
   - Metadata extraction job
   - Content hashing job
   - Thumbnail generation job (videos)
   - Tamper detection analysis
6. **Pending Upload Cleared** from temporary storage

---

### 3. Upload from URL

#### Endpoint

```
POST /api/media/upload-url
```

#### Description

Downloads media from an external URL, uploads to S3, and automatically confirms/processes. Combines all three steps of the standard flow into one request.

#### Request Headers

```
Content-Type: application/json
Authorization: Bearer <access_token>
```

#### Request Body

```json
{
  "url": "https://example.com/images/photo.jpg",
  "visibility": "private",
  "metadata": {
    "source": "example.com",
    "description": "Downloaded image"
  }
}
```

**Field Specifications**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | Yes | Valid HTTP/HTTPS URL |
| `visibility` | string | No | public/private/unlisted (default: private) |
| `metadata` | object | No | Additional metadata |

**URL Validation**:
- Must be HTTP or HTTPS
- Must be publicly accessible
- Must return valid content type
- File size must be within limits
- Response must complete within 60 seconds

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "URL upload completed successfully",
  "data": {
    "media": {
      "id": "507f1f77bcf86cd799439012",
      "filename": "photo.jpg",
      "fileSize": 1536789,
      "mimeType": "image/jpeg",
      "uploadType": "general_image",
      "status": "processing",
      "visibility": "private",
      "uploadedAt": "2025-12-02T10:50:00.000Z",
      "metadata": {
        "uploadMethod": "url",
        "sourceUrl": "https://example.com/images/photo.jpg",
        "downloadMetadata": {
          "contentLength": 1536789,
          "lastModified": "2025-12-01T10:00:00.000Z",
          "contentType": "image/jpeg"
        }
      }
    }
  },
  "timestamp": "2025-12-02T10:50:00.000Z"
}
```

#### Error Responses

**400 Bad Request - Invalid URL**

```json
{
  "success": false,
  "message": "Invalid URL",
  "data": {
    "errors": ["URL must use HTTP or HTTPS protocol"]
  },
  "timestamp": "2025-12-02T10:50:00.000Z"
}
```

**400 Bad Request - Download Failed**

```json
{
  "success": false,
  "message": "Failed to download file from URL",
  "data": {
    "reason": "Connection timeout after 60 seconds"
  },
  "timestamp": "2025-12-02T10:50:00.000Z"
}
```

**400 Bad Request - File Too Large**

```json
{
  "success": false,
  "message": "Downloaded file size (550000000 bytes) exceeds maximum allowed (524288000 bytes)",
  "timestamp": "2025-12-02T10:50:00.000Z"
}
```

**429 Too Many Requests - Rate Limit**

```json
{
  "success": false,
  "message": "Too many URL upload requests. Please try again later.",
  "timestamp": "2025-12-02T10:50:00.000Z"
}
```

#### Side Effects

1. Server downloads file from URL (temporary storage)
2. File uploaded to S3
3. Media record created
4. Background processing queued
5. Temporary downloaded file cleaned up

---

### 4. Get Upload Configuration

#### Endpoint

```
GET /api/media/config
```

#### Description

Returns upload configuration including file size limits, allowed types, and quota information.

#### Request Headers

```
Authorization: Bearer <access_token>
```

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "Upload configuration retrieved",
  "data": {
    "limits": {
      "general_image": {
        "maxSize": 10485760,
        "allowedTypes": ["image/jpeg", "image/png", "image/gif", "image/webp", "image/bmp", "image/tiff"]
      },
      "video": {
        "maxSize": 524288000,
        "allowedTypes": ["video/mp4", "video/quicktime", "video/x-msvideo", "video/webm"]
      },
      "audio": {
        "maxSize": 52428800,
        "allowedTypes": ["audio/mpeg", "audio/wav", "audio/aac", "audio/flac"]
      }
    },
    "quota": {
      "used": 1536000000,
      "limit": 10737418240,
      "remaining": 9201418240,
      "percentage": 14.3
    },
    "features": {
      "autoMetadataExtraction": true,
      "autoThumbnailGeneration": true,
      "contentHashingEnabled": true,
      "tamperDetectionEnabled": true
    }
  },
  "timestamp": "2025-12-02T10:55:00.000Z"
}
```

---

### 5. Get Media Statistics

#### Endpoint

```
GET /api/media/stats
```

#### Description

Returns user's media statistics including counts by type, storage usage, and processing status.

#### Request Headers

```
Authorization: Bearer <access_token>
```

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "Media statistics retrieved",
  "data": {
    "total": 45,
    "byType": {
      "general_image": 30,
      "video": 10,
      "audio": 5
    },
    "byStatus": {
      "processed": 40,
      "processing": 3,
      "failed": 2
    },
    "byVisibility": {
      "private": 35,
      "public": 8,
      "unlisted": 2
    },
    "storage": {
      "used": 1536000000,
      "limit": 10737418240,
      "percentage": 14.3
    },
    "recentUploads": [
      {
        "id": "507f1f77bcf86cd799439011",
        "filename": "recent-photo.jpg",
        "uploadedAt": "2025-12-02T10:30:00.000Z"
      }
    ]
  },
  "timestamp": "2025-12-02T10:55:00.000Z"
}
```

---

### 6. Generate Video Thumbnail

#### Endpoint

```
POST /api/media/:id/thumbnail
```

#### Description

Manually triggers video thumbnail generation for a specific media item. Normally thumbnails are generated automatically, but this allows regeneration.

#### Request Headers

```
Content-Type: application/json
Authorization: Bearer <access_token>
```

#### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string | Yes | Media document ID |

#### Request Body (Optional)

```json
{
  "config": {
    "timePosition": 10,
    "width": 1280,
    "height": 720,
    "quality": 85
  }
}
```

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "Thumbnail generation queued",
  "data": {
    "jobId": "thumbnail-job-123",
    "status": "queued",
    "estimatedTime": 30
  },
  "timestamp": "2025-12-02T11:00:00.000Z"
}
```

---

## Test Scenarios

### TS-MP-01: Complete Direct Upload Flow

**Objective**: Verify end-to-end direct S3 upload process

**Steps**:
1. Request presigned URL with valid image data
2. Verify presigned URL response format
3. [Simulate] Upload file to S3 using presigned URL
4. Confirm upload with s3Key and correlationId
5. Verify media record created
6. Wait for background processing
7. Verify metadata extracted
8. Verify thumbnail generated (if video)
9. Verify content hashes calculated

**Expected Outcome**: File successfully uploaded, processed, and metadata available

---

### TS-MP-02: URL Upload Flow

**Objective**: Verify upload from external URL

**Steps**:
1. Provide valid public image URL
2. Monitor upload progress
3. Verify file downloaded and re-uploaded to S3
4. Verify media record includes source URL
5. Verify automatic processing triggered

**Expected Outcome**: External file successfully imported and processed

---

### TS-MP-03: Quota Management

**Objective**: Verify storage quota enforcement

**Steps**:
1. Check current quota usage
2. Calculate remaining space
3. Attempt upload that would exceed quota
4. Verify rejection with quota error
5. Delete some media
6. Retry upload
7. Verify success after quota freed

**Expected Outcome**: Quota properly enforced and updated

---

### TS-MP-04: File Type Validation

**Objective**: Verify file type validation at multiple levels

**Steps**:
1. Attempt upload with mismatched MIME type
2. Attempt upload with invalid file signature
3. Attempt upload with unsupported file type
4. Attempt upload with correct type and signature
5. Verify only valid uploads succeed

**Expected Outcome**: Invalid files rejected, valid files accepted

---

### TS-MP-05: Large File Handling

**Objective**: Verify handling of large files

**Steps**:
1. Attempt upload at size limit (just under)
2. Attempt upload exceeding size limit
3. Attempt upload of large video file
4. Monitor processing time
5. Verify thumbnail generation for large video

**Expected Outcome**: Size limits enforced, large files processed correctly

---

## Detailed Test Cases

### Presigned URL Generation Tests

#### TC-MP-001: Generate Presigned URL for Image

**Objective**: Verify presigned URL generation for valid image upload

**Priority**: P0 - Critical

**Prerequisites**:
- Authenticated user with available quota
- Email verified

**Test Data**:
```json
{
  "filename": "test-image.jpg",
  "contentType": "image/jpeg",
  "fileSize": 2048576,
  "uploadType": "general_image",
  "visibility": "private"
}
```

**Test Steps**:
1. Send POST request to `/api/media/presigned-url`
2. Verify response status is `200 OK`
3. Verify response contains `uploadUrl`
4. Verify `uploadUrl` starts with S3 bucket URL
5. Verify `s3Key` has proper format: `media/{userId}/{uuid}.jpg`
6. Verify `correlationId` is valid UUID
7. Verify `expiresAt` is 15 minutes from now
8. Verify `maxFileSize` matches image limit (10MB)
9. Verify `allowedContentTypes` includes `image/jpeg`

**Expected Result**:
- Status: `200 OK`
- Valid presigned URL returned
- Expiration set correctly
- All required fields present

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

**Screenshot**: *[Add screenshot here]*

---

#### TC-MP-002: Presigned URL with Quota Exceeded

**Objective**: Verify quota check prevents presigned URL generation

**Priority**: P0 - Critical

**Prerequisites**:
- User with quota at or near limit

**Test Data**:
```json
{
  "filename": "large-image.jpg",
  "contentType": "image/jpeg",
  "fileSize": 10485760,
  "uploadType": "general_image"
}
```

**Test Steps**:
1. Check user's current quota (GET `/api/user/me/media-quota`)
2. Calculate if upload would exceed quota
3. Send POST request to `/api/media/presigned-url`
4. Verify response status is `403 Forbidden`
5. Verify error message mentions quota exceeded
6. Verify error includes used, limit, and requested values

**Expected Result**:
- Status: `403 Forbidden`
- Clear quota error message
- No presigned URL generated

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-003: Presigned URL for Oversized File

**Objective**: Verify file size validation

**Priority**: P0 - Critical

**Test Data**:
```json
{
  "filename": "huge-image.jpg",
  "contentType": "image/jpeg",
  "fileSize": 20971520,
  "uploadType": "general_image"
}
```

**Test Steps**:
1. Request presigned URL with file size exceeding limit (20MB > 10MB)
2. Verify response status is `400 Bad Request`
3. Verify error message specifies size limit
4. Verify error includes maximum allowed size

**Expected Result**:
- Status: `400 Bad Request`
- Error: "File size exceeds maximum allowed size of 10485760 bytes"

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-004: Presigned URL for Invalid File Type

**Objective**: Verify content type validation

**Priority**: P0 - Critical

**Test Data**:
```json
{
  "filename": "virus.exe",
  "contentType": "application/x-executable",
  "fileSize": 1024000,
  "uploadType": "general_image"
}
```

**Test Steps**:
1. Request presigned URL with disallowed content type
2. Verify response status is `400 Bad Request`
3. Verify error message indicates invalid content type
4. Verify response includes list of allowed types

**Expected Result**:
- Status: `400 Bad Request`
- Error mentions invalid content type
- `data.allowedTypes` array present

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-005: Presigned URL for Video

**Objective**: Verify presigned URL generation for video upload

**Priority**: P1 - High

**Test Data**:
```json
{
  "filename": "vacation.mp4",
  "contentType": "video/mp4",
  "fileSize": 52428800,
  "uploadType": "video",
  "visibility": "public"
}
```

**Test Steps**:
1. Request presigned URL for video
2. Verify success response
3. Verify `maxFileSize` matches video limit (500MB)
4. Verify `allowedContentTypes` includes video MIME types
5. Verify S3 key has video-appropriate path

**Expected Result**:
- Status: `200 OK`
- Video-specific limits applied
- Valid presigned URL

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-006: Presigned URL for Audio

**Objective**: Verify presigned URL generation for audio upload

**Priority**: P1 - High

**Test Data**:
```json
{
  "filename": "podcast.mp3",
  "contentType": "audio/mpeg",
  "fileSize": 10485760,
  "uploadType": "audio"
}
```

**Test Steps**:
1. Request presigned URL for audio
2. Verify success response
3. Verify `maxFileSize` matches audio limit (50MB)
4. Verify audio MIME types allowed

**Expected Result**:
- Status: `200 OK`
- Audio-specific limits applied

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-007: Presigned URL with Custom Metadata

**Objective**: Verify custom metadata is accepted

**Priority**: P2 - Medium

**Test Data**:
```json
{
  "filename": "test.jpg",
  "contentType": "image/jpeg",
  "fileSize": 2048576,
  "uploadType": "general_image",
  "metadata": {
    "description": "Test image for QA",
    "tags": ["test", "qa", "validation"],
    "location": "Test Lab",
    "customField": "custom value"
  }
}
```

**Test Steps**:
1. Request presigned URL with custom metadata
2. Verify success response
3. After upload confirmation, retrieve media record
4. Verify custom metadata stored correctly

**Expected Result**:
- Custom metadata accepted and stored

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

### Upload Confirmation Tests

#### TC-MP-008: Successful Upload Confirmation

**Objective**: Verify upload confirmation creates media record

**Priority**: P0 - Critical

**Prerequisites**:
- Valid presigned URL generated
- File uploaded to S3 (simulated or actual)

**Test Data**:
```json
{
  "s3Key": "media/user123/abc-def-ghi.jpg",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Test Steps**:
1. Generate presigned URL (TC-MP-001)
2. [Simulate] Upload file to S3
3. Send POST to `/api/media/confirm-upload` with s3Key and correlationId
4. Verify response status is `200 OK`
5. Verify media object returned with valid ID
6. Verify `status` is `processing` or `uploaded`
7. Verify `uploadedAt` timestamp present
8. Verify `filename`, `fileSize`, `mimeType` correct
9. Query media by ID to confirm record exists
10. Check background processing queue for jobs

**Expected Result**:
- Status: `200 OK`
- Media record created
- Background jobs queued
- User quota updated

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

**Screenshot**: *[Add screenshot here]*

---

#### TC-MP-009: Confirm with Missing Fields

**Objective**: Verify validation of required fields

**Priority**: P1 - High

**Test Data (Missing s3Key)**:
```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Test Steps**:
1. Send confirmation request with missing `s3Key`
2. Verify `400 Bad Request` response
3. Verify error message specifies missing fields
4. Repeat with missing `correlationId`

**Expected Result**:
- Status: `400 Bad Request`
- Message: "Missing required fields: s3Key, correlationId"

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-010: Confirm with Invalid Correlation ID

**Objective**: Verify correlation ID validation

**Priority**: P0 - Critical

**Test Data**:
```json
{
  "s3Key": "media/user123/abc-def-ghi.jpg",
  "correlationId": "invalid-correlation-id-12345"
}
```

**Test Steps**:
1. Send confirmation with non-existent correlation ID
2. Verify `400 Bad Request` response
3. Verify error message indicates invalid/expired session

**Expected Result**:
- Status: `400 Bad Request`
- Error: "Invalid correlation ID or expired upload session"

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-011: Confirm Non-Existent S3 File

**Objective**: Verify S3 file existence check

**Priority**: P0 - Critical

**Test Data**:
```json
{
  "s3Key": "media/user123/non-existent-file.jpg",
  "correlationId": "valid-correlation-id"
}
```

**Test Steps**:
1. Generate valid presigned URL
2. Do NOT upload file to S3
3. Attempt confirmation with s3Key
4. Verify `404 Not Found` or `400 Bad Request` response
5. Verify error indicates file not found in S3

**Expected Result**:
- Status: `404 Not Found`
- Error: "File not found in S3. Upload may have failed."

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-012: Confirm with File Signature Mismatch

**Objective**: Verify file signature validation

**Priority**: P0 - Critical (Security)

**Prerequisites**:
- Upload file with wrong extension/content type

**Test Steps**:
1. Request presigned URL for `image/jpeg`
2. Upload `.exe` file renamed as `.jpg` to S3
3. Attempt confirmation
4. Verify confirmation fails
5. Verify error indicates signature mismatch

**Expected Result**:
- Status: `400 Bad Request`
- Error: "File signature validation failed"
- Expected vs detected type shown

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-013: Confirm Duplicate Upload

**Objective**: Verify handling of duplicate confirmation attempts

**Priority**: P2 - Medium

**Test Steps**:
1. Perform successful upload and confirmation (TC-MP-008)
2. Attempt to confirm same s3Key and correlationId again
3. Verify appropriate response (either success with existing media or error)

**Expected Result**:
- Either returns existing media record or indicates already confirmed
- No duplicate media records created

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

### URL Upload Tests

#### TC-MP-014: Successful URL Upload

**Objective**: Verify upload from valid public URL

**Priority**: P1 - High

**Prerequisites**:
- Public URL with accessible image file

**Test Data**:
```json
{
  "url": "https://httpbin.org/image/jpeg",
  "visibility": "private"
}
```

**Test Steps**:
1. Send POST to `/api/media/upload-url` with valid URL
2. Verify response status is `200 OK`
3. Verify media object returned
4. Verify `metadata.uploadMethod` is `"url"`
5. Verify `metadata.sourceUrl` contains original URL
6. Verify `metadata.downloadMetadata` present
7. Verify file exists in user's media list
8. Verify background processing triggered

**Expected Result**:
- Status: `200 OK`
- Media successfully imported
- Source URL tracked

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

**Screenshot**: *[Add screenshot here]*

---

#### TC-MP-015: URL Upload with Invalid URL

**Objective**: Verify URL validation

**Priority**: P1 - High

**Test Data - Invalid URLs**:
- `ftp://example.com/file.jpg` (wrong protocol)
- `not-a-url` (malformed)
- `javascript:alert(1)` (XSS attempt)
- `file:///etc/passwd` (local file)

**Test Steps**:
1. For each invalid URL, send upload request
2. Verify `400 Bad Request` response
3. Verify error indicates invalid URL
4. Verify specific validation failure mentioned

**Expected Result**:
- Status: `400 Bad Request`
- Error: "Invalid URL"
- `data.errors` array with specific issues

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-016: URL Upload from Inaccessible URL

**Objective**: Verify handling of download failures

**Priority**: P1 - High

**Test Data**:
```json
{
  "url": "https://example.com/404-not-found.jpg"
}
```

**Test Steps**:
1. Attempt upload from non-existent URL (404)
2. Verify error response
3. Verify error indicates download failure
4. Test with timeout scenario (very slow server)
5. Test with network error

**Expected Result**:
- Status: `400 Bad Request` or `502 Bad Gateway`
- Error describes download failure reason

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-017: URL Upload Exceeding Size Limit

**Objective**: Verify size validation for URL downloads

**Priority**: P1 - High

**Test Data**:
```json
{
  "url": "https://example.com/huge-video.mp4"
}
```
(URL points to file >500MB)

**Test Steps**:
1. Attempt URL upload of oversized file
2. Verify download aborted when size detected
3. Verify error response
4. Verify no partial file left in S3

**Expected Result**:
- Status: `400 Bad Request`
- Error: "Downloaded file size exceeds maximum allowed"
- No storage quota consumed

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-018: URL Upload Rate Limiting

**Objective**: Verify rate limiting on URL uploads

**Priority**: P1 - High

**Prerequisites**:
- Rate limit configured (e.g., 5 requests per 15 minutes)

**Test Steps**:
1. Send 5 URL upload requests rapidly
2. Send 6th request
3. Verify 6th request returns `429 Too Many Requests`
4. Wait for rate limit window to expire
5. Verify requests work again

**Expected Result**:
- First N requests succeed
- (N+1)th request rate limited
- After timeout, works again

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

### Background Processing Tests

#### TC-MP-019: Metadata Extraction for Image

**Objective**: Verify automatic metadata extraction after upload

**Priority**: P0 - Critical

**Prerequisites**:
- Upload image with rich EXIF data (GPS, camera info, etc.)

**Test Steps**:
1. Upload image with EXIF data
2. Confirm upload
3. Wait for background processing (poll status or wait 30s)
4. Retrieve media by ID
5. Verify `status` changed to `processed`
6. Verify `metadata.extracted` present
7. Verify EXIF data extracted (camera, date, GPS)
8. Verify content hashes calculated (MD5, SHA256)
9. Verify tamper detection scores present

**Expected Result**:
- Status becomes `processed`
- Rich metadata extracted
- All analysis scores present

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

**Screenshot**: *[Add screenshot showing extracted metadata]*

---

#### TC-MP-020: Thumbnail Generation for Video

**Objective**: Verify automatic thumbnail generation

**Priority**: P0 - Critical

**Prerequisites**:
- Upload valid video file

**Test Steps**:
1. Upload video (MP4)
2. Confirm upload
3. Wait for background processing
4. Retrieve media by ID
5. Verify `thumbnailUrl` present
6. Verify thumbnail accessible via URL
7. Verify thumbnail dimensions (1280x720 default)
8. Verify `metadata.thumbnail` contains generation details

**Expected Result**:
- Thumbnail generated successfully
- URL accessible
- Proper dimensions

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

**Screenshot**: *[Add screenshot of generated thumbnail]*

---

#### TC-MP-021: Content Hashing

**Objective**: Verify content hash generation

**Priority**: P1 - High

**Test Steps**:
1. Upload file
2. Confirm upload
3. Wait for processing
4. Retrieve media metadata
5. Verify `metadata.hashes.md5` present (32 char hex)
6. Verify `metadata.hashes.sha256` present (64 char hex)
7. Upload same file again
8. Verify hashes match (duplicate detection)

**Expected Result**:
- MD5 and SHA256 hashes generated
- Hashes consistent for identical files

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-022: Tamper Detection Analysis

**Objective**: Verify tamper detection runs automatically

**Priority**: P1 - High

**Test Steps**:
1. Upload pristine image (from camera, unedited)
2. Confirm and wait for processing
3. Check metadata for tamper analysis:
   - `integrityScore`
   - `authenticityScore`
   - `confidence`
   - `possibleTampering` flag
4. Upload edited image (known Photoshop edit)
5. Compare scores

**Expected Result**:
- Pristine image: High integrity/authenticity scores
- Edited image: Lower scores, detected editing software

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-023: Processing Failure Handling

**Objective**: Verify handling of processing failures

**Priority**: P1 - High

**Test Steps**:
1. Upload corrupted media file
2. Confirm upload
3. Wait for processing
4. Check media status
5. Verify status is `failed`
6. Verify error message present in metadata
7. Verify user can retry processing or delete

**Expected Result**:
- Status: `failed`
- Error details available
- File not stuck in processing state

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

### Configuration & Statistics Tests

#### TC-MP-024: Get Upload Configuration

**Objective**: Verify upload config endpoint

**Priority**: P2 - Medium

**Test Steps**:
1. Send GET to `/api/media/config`
2. Verify `200 OK` response
3. Verify `limits` object present with all upload types
4. Verify `quota` object shows current usage
5. Verify `features` object lists enabled features
6. Verify all max sizes and allowed types correct

**Expected Result**:
- Complete configuration returned
- Accurate quota information

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-025: Get Media Statistics

**Objective**: Verify statistics endpoint

**Priority**: P2 - Medium

**Test Steps**:
1. Upload several files of different types
2. Send GET to `/api/media/stats`
3. Verify counts by type accurate
4. Verify counts by status accurate
5. Verify storage usage matches quota
6. Verify recent uploads list present

**Expected Result**:
- Accurate statistics
- All categories covered

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

### Manual Thumbnail Generation Tests

#### TC-MP-026: Manual Thumbnail Generation

**Objective**: Verify manual thumbnail trigger

**Priority**: P2 - Medium

**Prerequisites**:
- Existing video media without thumbnail

**Test Steps**:
1. Upload video
2. Send POST to `/api/media/:id/thumbnail`
3. Verify job queued response
4. Wait for completion
5. Verify thumbnail generated
6. Verify old thumbnail replaced (if regenerating)

**Expected Result**:
- Job queued successfully
- Thumbnail generated

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-MP-027: Thumbnail with Custom Config

**Objective**: Verify custom thumbnail configuration

**Priority**: P3 - Low

**Test Data**:
```json
{
  "config": {
    "timePosition": 50,
    "width": 1920,
    "height": 1080,
    "quality": 90
  }
}
```

**Test Steps**:
1. Request thumbnail with custom config
2. Verify thumbnail matches specifications
3. Verify dimensions correct
4. Verify quality level applied

**Expected Result**:
- Custom config respected
- High-quality thumbnail at specified position

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

## Edge Cases & Boundary Testing

### Edge Case Tests

#### TC-MP-028: Zero-Byte File

**Objective**: Verify handling of empty files

**Test Data**:
```json
{
  "filename": "empty.jpg",
  "contentType": "image/jpeg",
  "fileSize": 0,
  "uploadType": "general_image"
}
```

**Expected Result**: Rejected with appropriate error

**Status**: ⏳ Pending

---

#### TC-MP-029: File at Exact Size Limit

**Objective**: Verify boundary at size limit

**Test Data**: File exactly 10485760 bytes (10MB)

**Expected Result**: Accepted (should be <=, not <)

**Status**: ⏳ Pending

---

#### TC-MP-030: File One Byte Over Limit

**Objective**: Verify rejection just over limit

**Test Data**: File of 10485761 bytes

**Expected Result**: Rejected with size error

**Status**: ⏳ Pending

---

#### TC-MP-031: Filename with Special Characters

**Objective**: Verify filename sanitization

**Test Data**:
- `file with spaces.jpg`
- `file\twith\ttabs.jpg`
- `file/with/slashes.jpg`
- `file:with:colons.jpg`
- `файл-кириллица.jpg` (Cyrillic)
- `文件.jpg` (Chinese)
- `file\x00null.jpg` (null byte)

**Expected Result**: Filenames sanitized or accepted safely

**Status**: ⏳ Pending

---

#### TC-MP-032: Very Long Filename

**Objective**: Verify filename length limits

**Test Data**: Filename with 500 characters

**Expected Result**: Either truncated or rejected with clear error

**Status**: ⏳ Pending

---

#### TC-MP-033: Concurrent Upload Requests

**Objective**: Verify handling of simultaneous uploads

**Test Steps**:
1. Send 10 presigned URL requests simultaneously
2. Verify all succeed or fail gracefully
3. Verify no race conditions in quota checking
4. Confirm all uploads
5. Verify quota correctly updated

**Expected Result**: Concurrent requests handled safely

**Status**: ⏳ Pending

---

#### TC-MP-034: Upload During Quota Update

**Objective**: Verify race condition handling

**Test Steps**:
1. Start upload process
2. Simultaneously delete another media (freeing quota)
3. Verify consistent quota calculation
4. No double-counting or quota corruption

**Expected Result**: Quota remains consistent

**Status**: ⏳ Pending

---

#### TC-MP-035: Presigned URL Expiration

**Objective**: Verify expired URL handling

**Test Steps**:
1. Generate presigned URL
2. Wait 16 minutes (URL expires after 15)
3. Attempt to use expired URL for S3 upload
4. Verify S3 rejects upload
5. Attempt confirmation
6. Verify graceful error

**Expected Result**: Expired URLs don't work, clear error message

**Status**: ⏳ Pending

---

#### TC-MP-036: Upload Metadata Exceeding Limits

**Objective**: Verify metadata size limits

**Test Data**: Metadata object >16KB

**Expected Result**: Either truncated, rejected, or stored correctly

**Status**: ⏳ Pending

---

#### TC-MP-037: Malformed Image with Valid Signature

**Objective**: Verify handling of corrupted files

**Test Steps**:
1. Upload image with valid JPEG signature but corrupted data
2. Confirm upload (should succeed - signature valid)
3. Wait for metadata extraction
4. Verify processing marked as failed or partial
5. Verify error logged

**Expected Result**: Upload succeeds, processing fails gracefully

**Status**: ⏳ Pending

---

#### TC-MP-038: Upload While At Quota Limit

**Objective**: Verify edge case at exact quota

**Test Steps**:
1. Fill quota to exact limit
2. Attempt 1-byte upload
3. Verify rejection
4. Delete 1-byte file
5. Attempt 1-byte upload
6. Verify success

**Expected Result**: Quota enforcement exact, no off-by-one errors

**Status**: ⏳ Pending

---

#### TC-MP-039: Multiple File Extensions

**Objective**: Verify handling of unusual filenames

**Test Data**:
- `file.tar.gz.jpg`
- `file.jpg.exe` (suspicious)
- `file.JPG` (uppercase extension)
- `file.` (no extension)
- `file.jpeg` vs `file.jpg`

**Expected Result**: Extensions handled correctly, suspicious files rejected

**Status**: ⏳ Pending

---

#### TC-MP-040: Upload Type Inference Edge Cases

**Objective**: Verify upload type auto-detection

**Test Steps**:
1. Upload image without specifying uploadType
2. Verify system infers `general_image`
3. Upload video without specifying
4. Verify system infers `video`
5. Upload ambiguous file
6. Verify appropriate fallback or error

**Expected Result**: Smart inference works correctly

**Status**: ⏳ Pending

---

## Security Testing

### Security Test Cases

#### TC-MP-SEC-001: File Signature Validation

**Objective**: Prevent executable upload masquerading as image

**Priority**: P0 - Critical

**Test Steps**:
1. Create `.exe` file
2. Rename to `.jpg`
3. Request presigned URL as `image/jpeg`
4. Upload to S3
5. Attempt confirmation
6. Verify rejection due to signature mismatch

**Expected Result**: File signature check prevents malicious upload

**Status**: ⏳ Pending

---

#### TC-MP-SEC-002: Path Traversal in Filename

**Objective**: Prevent directory traversal attacks

**Priority**: P0 - Critical

**Test Data**:
- `../../etc/passwd.jpg`
- `..\\..\\windows\\system32\\config.jpg`
- `%2e%2e%2f%2e%2e%2fpasswd.jpg` (URL encoded)

**Expected Result**: Path traversal attempts sanitized or rejected

**Status**: ⏳ Pending

---

#### TC-MP-SEC-003: XXE in Metadata

**Objective**: Prevent XML External Entity attacks

**Priority**: P0 - Critical (if XML metadata supported)

**Test Steps**:
1. Upload image with malicious XML in EXIF
2. Verify XML parsing doesn't resolve external entities
3. Verify no file system access
4. Verify no SSRF attacks possible

**Expected Result**: XXE attacks prevented

**Status**: ⏳ Pending

---

#### TC-MP-SEC-004: SSRF via URL Upload

**Objective**: Prevent Server-Side Request Forgery

**Priority**: P0 - Critical

**Test Data**:
- `http://localhost/admin`
- `http://169.254.169.254/latest/meta-data/` (AWS metadata)
- `http://internal-service:8080/secret`

**Expected Result**: Internal URLs rejected, SSRF prevented

**Status**: ⏳ Pending

---

#### TC-MP-SEC-005: Zip Bomb Upload

**Objective**: Prevent resource exhaustion

**Priority**: P1 - High

**Test Steps**:
1. Create highly compressed zip file (expands to GBs)
2. Upload as image
3. Verify processing doesn't exhaust resources
4. Verify size limits enforced before decompression

**Expected Result**: Zip bombs detected or size limits prevent exhaustion

**Status**: ⏳ Pending

---

#### TC-MP-SEC-006: SQL/NoSQL Injection in Metadata

**Objective**: Prevent injection attacks

**Priority**: P0 - Critical

**Test Data**:
```json
{
  "metadata": {
    "description": "'; DROP TABLE media; --",
    "tags": ["$ne", {"$gt": ""}],
    "custom": { "$where": "malicious code" }
  }
}
```

**Expected Result**: Metadata sanitized, no injection possible

**Status**: ⏳ Pending

---

#### TC-MP-SEC-007: XSS in Filename or Metadata

**Objective**: Prevent stored XSS

**Priority**: P1 - High

**Test Data**:
```json
{
  "filename": "<script>alert('XSS')</script>.jpg",
  "metadata": {
    "description": "<img src=x onerror=alert(1)>"
  }
}
```

**Expected Result**: HTML entities escaped in storage/display

**Status**: ⏳ Pending

---

#### TC-MP-SEC-008: Token Correlation Hijacking

**Objective**: Prevent correlation ID hijacking

**Priority**: P0 - Critical

**Test Steps**:
1. User A generates presigned URL (gets correlationId)
2. User B attempts to use User A's correlationId and s3Key
3. Verify confirmation fails (user mismatch)
4. Verify User A's file not stolen

**Expected Result**: Correlation IDs tied to specific user

**Status**: ⏳ Pending

---

#### TC-MP-SEC-009: S3 Bucket Permissions

**Objective**: Verify S3 bucket security

**Priority**: P0 - Critical

**Test Steps**:
1. Verify bucket not publicly listable
2. Verify presigned URLs expire correctly
3. Verify signed URLs can't be modified
4. Verify direct S3 access denied without signed URL

**Expected Result**: Proper S3 security configuration

**Status**: ⏳ Pending

---

#### TC-MP-SEC-010: Rate Limiting Bypass

**Objective**: Prevent rate limit evasion

**Priority**: P1 - High

**Test Steps**:
1. Hit rate limit
2. Attempt bypass with different User-Agent
3. Attempt bypass with different IP (if using proxy)
4. Attempt bypass with new session
5. Verify all blocked

**Expected Result**: Rate limits enforced per user, not bypassable

**Status**: ⏳ Pending

---

## Performance Considerations

### Performance Benchmarks

| Operation | Target Response Time | Notes |
|-----------|---------------------|-------|
| Generate Presigned URL | < 500ms | Database + S3 API call |
| Confirm Upload | < 1000ms | File existence check + DB write |
| URL Upload | < 30s | Depends on file size and source speed |
| Metadata Extraction | < 10s (background) | Varies by file complexity |
| Thumbnail Generation | < 30s (background) | Depends on video length |

### Load Testing Scenarios

**Concurrent Uploads**:
- 50 simultaneous presigned URL requests
- 20 simultaneous confirmations
- 10 simultaneous URL uploads

**Large File Handling**:
- 500MB video upload
- Batch of 100 small images

---

## Test Results

### Summary

| Category | Total Tests | Passed | Failed | Blocked | Pass Rate |
|----------|-------------|--------|--------|---------|-----------|
| Presigned URL Generation | 7 | 0 | 0 | 0 | 0% |
| Upload Confirmation | 6 | 0 | 0 | 0 | 0% |
| URL Upload | 5 | 0 | 0 | 0 | 0% |
| Background Processing | 5 | 0 | 0 | 0 | 0% |
| Configuration & Stats | 2 | 0 | 0 | 0 | 0% |
| Manual Thumbnail | 2 | 0 | 0 | 0 | 0% |
| Edge Cases | 13 | 0 | 0 | 0 | 0% |
| Security | 10 | 0 | 0 | 0 | 0% |
| **TOTAL** | **50** | **0** | **0** | **0** | **0%** |

---

## Issues Found

### Critical Issues (P0)

No issues found yet.

### High Priority Issues (P1)

No issues found yet.

---

## Recommendations

### Feature Enhancements

1. **Upload Progress Tracking** - WebSocket or polling endpoint for real-time progress
2. **Resumable Uploads** - Support for pausing/resuming large uploads
3. **Batch Upload UI** - Better support for multiple file uploads
4. **Thumbnail Customization** - Allow users to select thumbnail frame

### Performance Optimizations

1. **Parallel Processing** - Process metadata, hashing, thumbnails concurrently
2. **CDN Integration** - Serve thumbnails via CloudFront
3. **Lazy Metadata Extraction** - Extract metadata on-demand for rarely accessed files

### Security Improvements

1. **File Scanning** - Integrate virus/malware scanning
2. **Content Moderation** - Automated NSFW/harmful content detection
3. **Enhanced SSRF Protection** - IP blacklisting for URL uploads

---

**[← Back to Authentication](./01-authentication.md)** | **[Home](../README.md)** | **[Next: Media Management →](./03-media-management.md)**

---

*Last Updated: December 2, 2025*
