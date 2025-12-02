# Feature 03: Media Management

---

**Feature**: Media Management
**Priority**: P0 - Critical
**Test Coverage**: CRUD operations, filtering, authorization
**Dependencies**: Authentication (Feature 01), Media Processing (Feature 02)

---

## Table of Contents

1. [Overview](#overview)
2. [Test Scope](#test-scope)
3. [API Endpoints](#api-endpoints)
4. [Test Scenarios](#test-scenarios)
5. [Test Cases](#test-cases)
6. [Edge Cases](#edge-cases)
7. [Security Tests](#security-tests)
8. [Performance Benchmarks](#performance-benchmarks)
9. [Test Results](#test-results)
10. [Issues Found](#issues-found)
11. [Recommendations](#recommendations)

---

## Overview

### Purpose

Media Management provides comprehensive CRUD (Create, Read, Update, Delete) operations for managing uploaded media files. Users can list their media with filtering and pagination, retrieve individual files, update metadata and visibility settings, and delete files either individually or in bulk.

### Key Features

- **Media Listing** - Paginated retrieval with filtering by type, status, visibility
- **Media Details** - Retrieve complete information about a single media file
- **Media Updates** - Modify visibility settings and custom metadata
- **Media Deletion** - Single file or bulk deletion with quota updates
- **Authorization** - Users can only access/modify their own media
- **Soft Deletion** - Media is marked as deleted, allowing recovery if needed
- **Quota Management** - Storage quota automatically updated on deletion
- **Access Tracking** - Last accessed timestamp updated on retrieval

### Business Value

- Enables users to organize and manage their uploaded content
- Provides filtering and search capabilities for large media libraries
- Supports privacy controls through visibility settings
- Maintains data integrity through authorization checks
- Optimizes storage through bulk deletion capabilities

---

## Test Scope

### In Scope

**Core CRUD Operations:**
- ✅ List user media with pagination
- ✅ Retrieve single media by ID
- ✅ Update media visibility and metadata
- ✅ Delete single media file
- ✅ Bulk delete multiple media files

**Filtering & Sorting:**
- ✅ Filter by upload type (image, video, audio, document)
- ✅ Filter by status (pending, uploaded, processing, processed, failed, corrupted)
- ✅ Filter by visibility (public, private, unlisted)
- ✅ Sort by various fields (createdAt, fileSize, uploadedAt, etc.)
- ✅ Ascending/descending order
- ✅ Pagination with configurable page size

**Authorization:**
- ✅ Users can only access their own media
- ✅ Media ownership validation on all operations
- ✅ Proper error responses for unauthorized access

**Data Integrity:**
- ✅ Soft deletion with recovery capability
- ✅ S3 file deletion on media deletion
- ✅ Quota updates on deletion
- ✅ Profile picture cleanup if deleted
- ✅ Last accessed timestamp tracking

### Out of Scope

- ❌ Media upload operations (covered in Feature 02)
- ❌ Thumbnail generation (covered in Feature 02)
- ❌ Metadata extraction (covered in Feature 04)
- ❌ Media processing workflows (covered in Feature 02)
- ❌ Sharing and collaboration features (future enhancement)
- ❌ Advanced search with full-text search (future enhancement)
- ❌ Media collections/albums (future enhancement)

---

## API Endpoints

### 1. List User Media

```
GET /api/media/
```

**Description**: Retrieve paginated list of user's media with optional filtering and sorting.

**Authentication**: Required (JWT Bearer token)

**Query Parameters**:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | Number | No | `1` | Page number for pagination (min: 1) |
| `limit` | Number | No | `20` | Results per page (min: 1, max: 100) |
| `sort` | String | No | `createdAt` | Field to sort by (createdAt, fileSize, uploadedAt, originalFilename) |
| `order` | String | No | `desc` | Sort order: `asc` or `desc` |
| `uploadType` | String | No | - | Filter by upload type: `general_image`, `video`, `audio`, `document`, `profile_image` |
| `status` | String | No | - | Filter by status: `pending`, `uploaded`, `processing`, `processed`, `failed`, `corrupted` |
| `visibility` | String | No | - | Filter by visibility: `public`, `private`, `unlisted` |

**Success Response** (200 OK):

```json
{
  "success": true,
  "message": "Media retrieved successfully",
  "data": {
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 45,
      "pages": 3,
      "hasNext": true,
      "hasPrev": false
    },
    "media": [
      {
        "id": "507f1f77bcf86cd799439011",
        "filename": "landscape_photo.jpg",
        "fileSize": 2456789,
        "mimeType": "image/jpeg",
        "uploadType": "general_image",
        "status": "processed",
        "visibility": "private",
        "publicUrl": "https://cdn.example.com/users/user123/abc123.jpg",
        "thumbnailUrl": "https://cdn.example.com/users/user123/thumbnails/abc123.jpg",
        "uploadedAt": "2025-12-02T10:30:00.000Z",
        "createdAt": "2025-12-02T10:25:00.000Z",
        "metadata": {
          "uploadMethod": "direct",
          "width": 1920,
          "height": 1080,
          "camera": "Canon EOS 5D Mark IV"
        },
        "timeline": {
          "hasVerification": true,
          "status": "verified",
          "confidence": 0.92
        }
      }
    ]
  },
  "timestamp": "2025-12-02T14:20:35.123Z"
}
```

**Error Responses**:

| Status Code | Error | Description |
|-------------|-------|-------------|
| 401 | Unauthorized | Missing or invalid authentication token |
| 403 | Forbidden | Email not verified |
| 500 | Internal Server Error | Database or server error |

---

### 2. Get Media by ID

```
GET /api/media/:id
```

**Description**: Retrieve detailed information about a single media file. Updates the `lastAccessedAt` timestamp.

**Authentication**: Required (JWT Bearer token)

**URL Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | String | Yes | MongoDB ObjectId of the media (24 hex characters) |

**Success Response** (200 OK):

```json
{
  "success": true,
  "message": "Media retrieved successfully",
  "data": {
    "media": {
      "id": "507f1f77bcf86cd799439011",
      "filename": "landscape_photo.jpg",
      "sanitizedFilename": "landscape_photo_abc123.jpg",
      "fileSize": 2456789,
      "mimeType": "image/jpeg",
      "uploadType": "general_image",
      "status": "processed",
      "visibility": "private",
      "publicUrl": "https://cdn.example.com/users/user123/abc123.jpg",
      "thumbnailUrl": "https://cdn.example.com/users/user123/thumbnails/abc123.jpg",
      "uploadedAt": "2025-12-02T10:30:00.000Z",
      "processedAt": "2025-12-02T10:31:45.000Z",
      "lastAccessedAt": "2025-12-02T14:20:35.000Z",
      "createdAt": "2025-12-02T10:25:00.000Z",
      "updatedAt": "2025-12-02T14:20:35.000Z",
      "metadata": {
        "uploadMethod": "direct",
        "width": 1920,
        "height": 1080,
        "camera": "Canon EOS 5D Mark IV",
        "exif": {
          "DateTimeOriginal": "2024:11:15 14:32:18",
          "GPSLatitude": 40.7128,
          "GPSLongitude": -74.0060
        }
      },
      "timeline": {
        "hasVerification": true,
        "status": "verified",
        "confidence": 0.92,
        "timestamps": {
          "fileCreation": "2024-11-15T14:32:18.000Z",
          "exifDate": "2024-11-15T14:32:18.000Z",
          "uploadDate": "2025-12-02T10:30:00.000Z"
        }
      }
    }
  },
  "timestamp": "2025-12-02T14:20:35.123Z"
}
```

**Error Responses**:

| Status Code | Error | Description |
|-------------|-------|-------------|
| 400 | Bad Request | Invalid media ID format (must be 24 hex characters) |
| 401 | Unauthorized | Missing or invalid authentication token |
| 403 | Forbidden | User does not own this media file |
| 404 | Not Found | Media not found or has been deleted |
| 500 | Internal Server Error | Database or server error |

**Side Effects**:
- Updates `lastAccessedAt` timestamp to current time (async operation, doesn't block response)

---

### 3. Update Media

```
PATCH /api/media/:id
```

**Description**: Update media visibility settings and/or custom metadata. At least one field must be provided.

**Authentication**: Required (JWT Bearer token)

**URL Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | String | Yes | MongoDB ObjectId of the media (24 hex characters) |

**Request Body**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `visibility` | String | No* | Visibility setting: `public`, `private`, or `unlisted` |
| `metadata` | Object | No* | Custom metadata object (key-value pairs) |

*At least one field (visibility or metadata) must be provided

**Example Request**:

```json
{
  "visibility": "public",
  "metadata": {
    "description": "Beautiful sunset over the mountains",
    "tags": ["nature", "landscape", "sunset"],
    "location": "Rocky Mountains, Colorado",
    "customField": "any value"
  }
}
```

**Success Response** (200 OK):

```json
{
  "success": true,
  "message": "Media updated successfully",
  "data": {
    "media": {
      "id": "507f1f77bcf86cd799439011",
      "visibility": "public",
      "metadata": {
        "uploadMethod": "direct",
        "width": 1920,
        "height": 1080,
        "camera": "Canon EOS 5D Mark IV",
        "description": "Beautiful sunset over the mountains",
        "tags": ["nature", "landscape", "sunset"],
        "location": "Rocky Mountains, Colorado",
        "customField": "any value"
      },
      "updatedAt": "2025-12-02T14:25:00.000Z"
    }
  },
  "timestamp": "2025-12-02T14:25:00.123Z"
}
```

**Error Responses**:

| Status Code | Error | Description |
|-------------|-------|-------------|
| 400 | Bad Request | Invalid media ID format, invalid visibility value, or no fields provided |
| 401 | Unauthorized | Missing or invalid authentication token |
| 403 | Forbidden | User does not own this media file |
| 404 | Not Found | Media not found or has been deleted |
| 500 | Internal Server Error | Database or server error |

**Field Specifications**:

- **Visibility Values**: Must be one of `public`, `private`, or `unlisted`
  - `public`: Accessible via public URL without authentication
  - `private`: Only accessible by the owner
  - `unlisted`: Accessible via public URL but not listed in public galleries

- **Metadata Object**:
  - Merged with existing metadata (new fields added, existing fields updated)
  - Can contain any key-value pairs
  - No limit on number of fields (within MongoDB document size limits)
  - Values can be strings, numbers, booleans, arrays, or nested objects

---

### 4. Delete Media

```
DELETE /api/media/:id
```

**Description**: Delete a single media file. Performs soft deletion in database and schedules S3 file deletion. Updates user's storage quota.

**Authentication**: Required (JWT Bearer token)

**URL Parameters**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | String | Yes | MongoDB ObjectId of the media (24 hex characters) |

**Success Response** (200 OK):

```json
{
  "success": true,
  "message": "Media deleted successfully",
  "data": null,
  "timestamp": "2025-12-02T14:30:00.123Z"
}
```

**Error Responses**:

| Status Code | Error | Description |
|-------------|-------|-------------|
| 400 | Bad Request | Invalid media ID format (must be 24 hex characters) |
| 401 | Unauthorized | Missing or invalid authentication token |
| 403 | Forbidden | User does not own this media file |
| 404 | Not Found | Media not found or already deleted |
| 500 | Internal Server Error | Database or server error |

**Side Effects**:

1. **Soft Deletion**: Media record marked as deleted (`isDeleted: true`) with `deletedAt` timestamp
2. **S3 Cleanup**: File deletion from S3 bucket scheduled (async, non-blocking)
3. **Quota Update**: User's storage quota decreased by file size
4. **Profile Picture**: If media was user's profile picture, profile picture field cleared
5. **Cascading**: Related records (analysis results, verification data) may be cleaned up

**Recovery**: Soft-deleted media can potentially be recovered by admin within a retention period before permanent S3 deletion.

---

### 5. Bulk Delete Media

```
DELETE /api/media/bulk
```

**Description**: Delete multiple media files in a single request. Maximum 50 files per request. Returns detailed results for each file.

**Authentication**: Required (JWT Bearer token)

**Request Body**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mediaIds` | Array[String] | Yes | Array of media IDs to delete (1-50 items) |

**Example Request**:

```json
{
  "mediaIds": [
    "507f1f77bcf86cd799439011",
    "507f1f77bcf86cd799439012",
    "507f1f77bcf86cd799439013"
  ]
}
```

**Success Response - All Deleted** (200 OK):

```json
{
  "success": true,
  "message": "All files deleted successfully",
  "data": {
    "deleted": 3,
    "failed": 0,
    "errors": []
  },
  "timestamp": "2025-12-02T14:35:00.123Z"
}
```

**Partial Success Response** (206 Partial Content):

```json
{
  "success": false,
  "message": "Bulk delete completed with 1 failures",
  "data": {
    "deleted": 2,
    "failed": 1,
    "errors": [
      "507f1f77bcf86cd799439013: Media not found"
    ]
  },
  "timestamp": "2025-12-02T14:35:00.123Z"
}
```

**Error Responses**:

| Status Code | Error | Description |
|-------------|-------|-------------|
| 400 | Bad Request | Invalid request body, empty array, or more than 50 IDs |
| 401 | Unauthorized | Missing or invalid authentication token |
| 403 | Forbidden | Email not verified |
| 500 | Internal Server Error | Database or server error |

**Validation Rules**:
- Minimum 1 media ID required
- Maximum 50 media IDs per request
- Each ID must be valid MongoDB ObjectId format (24 hex characters)
- Invalid IDs will be reported in the errors array

**Behavior**:
- Processes each deletion independently
- Continues processing even if some deletions fail
- Returns 200 (OK) if all succeeded
- Returns 206 (Partial Content) if any failed
- Only deletes media owned by the authenticated user
- Same side effects as single deletion (quota updates, S3 cleanup, etc.)

---

## Test Scenarios

### Scenario 1: Media Listing & Filtering

**User Story**: As a user with multiple uploaded files, I want to view and filter my media library to find specific files.

**Test Coverage**:
- List all media with default pagination
- Paginate through large result sets
- Filter by upload type (images only, videos only, etc.)
- Filter by status (processed files only)
- Filter by visibility (private files only)
- Sort by different fields (date, size, filename)
- Combine multiple filters
- Handle empty result sets

---

### Scenario 2: Media Retrieval & Access Tracking

**User Story**: As a user, I want to view detailed information about a specific media file, including all metadata and processing results.

**Test Coverage**:
- Retrieve valid media by ID
- Access tracking (lastAccessedAt updated)
- View complete metadata and timeline data
- Handle non-existent media IDs
- Handle deleted media
- Prevent access to other users' media

---

### Scenario 3: Media Updates

**User Story**: As a user, I want to change the visibility of my media and add custom descriptions or tags.

**Test Coverage**:
- Update visibility setting only
- Update custom metadata only
- Update both visibility and metadata
- Merge new metadata with existing metadata
- Validate visibility values
- Prevent unauthorized updates
- Handle updates to non-existent media

---

### Scenario 4: Media Deletion

**User Story**: As a user, I want to delete unwanted media files to free up storage space.

**Test Coverage**:
- Delete single media file
- Verify quota updates after deletion
- Verify S3 cleanup initiated
- Handle profile picture deletion
- Prevent deletion of other users' media
- Handle deletion of already-deleted media
- Verify soft deletion behavior

---

### Scenario 5: Bulk Operations

**User Story**: As a user with many files, I want to delete multiple files at once rather than one by one.

**Test Coverage**:
- Delete multiple files successfully
- Handle partial failures gracefully
- Validate request limits (max 50 files)
- Handle mix of valid and invalid IDs
- Handle mix of owned and unowned media
- Verify quota updates for bulk deletions

---

## Test Cases

### List Media Tests

#### TC-301: List Media with Default Pagination

**Objective**: Verify that authenticated users can retrieve their media with default pagination settings.

**Prerequisites**:
- User account with verified email
- At least 5 uploaded media files

**Test Steps**:
1. Authenticate as a user with multiple media files
2. Send GET request to `/api/media/` with no query parameters
3. Verify response status and data structure
4. Check pagination metadata

**Expected Result**:
- Status: 200 OK
- Response includes pagination object with correct values
- Media array contains up to 20 items (default limit)
- Items sorted by createdAt descending (newest first)
- Each media object includes all required fields

**Status**: ⏳ Pending

---

#### TC-302: Paginate Through Media

**Objective**: Verify pagination works correctly across multiple pages.

**Prerequisites**:
- User account with at least 25 media files

**Test Steps**:
1. Authenticate as user
2. Request page 1 with limit=10: `GET /api/media/?page=1&limit=10`
3. Request page 2 with limit=10: `GET /api/media/?page=2&limit=10`
4. Request page 3 with limit=10: `GET /api/media/?page=3&limit=10`
5. Verify no duplicate items across pages
6. Check pagination metadata for each page

**Expected Result**:
- Page 1: Returns items 1-10, hasNext=true, hasPrev=false
- Page 2: Returns items 11-20, hasNext=true, hasPrev=true
- Page 3: Returns items 21-25, hasNext=false, hasPrev=true
- No duplicate media IDs across pages
- Total count matches across all requests

**Status**: ⏳ Pending

---

#### TC-303: Filter by Upload Type

**Objective**: Verify filtering by upload type returns only matching media.

**Prerequisites**:
- User account with mix of images, videos, and audio files

**Test Steps**:
1. Authenticate as user
2. Request images only: `GET /api/media/?uploadType=general_image`
3. Request videos only: `GET /api/media/?uploadType=video`
4. Request audio only: `GET /api/media/?uploadType=audio`
5. Verify each response contains only the specified type

**Expected Result**:
- Image filter returns only media with uploadType=general_image or profile_image
- Video filter returns only media with uploadType=video
- Audio filter returns only media with uploadType=audio
- All returned items match the filter criteria

**Status**: ⏳ Pending

---

#### TC-304: Filter by Status

**Objective**: Verify filtering by processing status works correctly.

**Prerequisites**:
- User account with media in various statuses (pending, processing, processed, failed)

**Test Steps**:
1. Authenticate as user
2. Request processed files: `GET /api/media/?status=processed`
3. Request pending files: `GET /api/media/?status=pending`
4. Request failed files: `GET /api/media/?status=failed`
5. Verify each response contains only files with matching status

**Expected Result**:
- Each filter returns only media with the specified status
- Status values match filter exactly
- Empty array returned if no files match the status

**Status**: ⏳ Pending

---

#### TC-305: Filter by Visibility

**Objective**: Verify filtering by visibility setting works correctly.

**Prerequisites**:
- User account with mix of public, private, and unlisted media

**Test Steps**:
1. Authenticate as user
2. Request private files: `GET /api/media/?visibility=private`
3. Request public files: `GET /api/media/?visibility=public`
4. Request unlisted files: `GET /api/media/?visibility=unlisted`
5. Verify each response contains only files with matching visibility

**Expected Result**:
- Private filter returns only media with visibility=private
- Public filter returns only media with visibility=public
- Unlisted filter returns only media with visibility=unlisted
- Visibility values match filter exactly

**Status**: ⏳ Pending

---

#### TC-306: Sort by Different Fields

**Objective**: Verify sorting works correctly for various fields and orders.

**Prerequisites**:
- User account with at least 10 media files with varying sizes and dates

**Test Steps**:
1. Authenticate as user
2. Sort by creation date ascending: `GET /api/media/?sort=createdAt&order=asc`
3. Sort by creation date descending: `GET /api/media/?sort=createdAt&order=desc`
4. Sort by file size ascending: `GET /api/media/?sort=fileSize&order=asc`
5. Sort by file size descending: `GET /api/media/?sort=fileSize&order=desc`
6. Verify sort order for each request

**Expected Result**:
- Ascending order: Items arranged from smallest to largest / oldest to newest
- Descending order: Items arranged from largest to smallest / newest to oldest
- Sort field values are correctly ordered in response
- Sort order is consistent across pagination

**Status**: ⏳ Pending

---

#### TC-307: Combine Multiple Filters

**Objective**: Verify that multiple filters can be combined in a single query.

**Prerequisites**:
- User account with diverse media library

**Test Steps**:
1. Authenticate as user
2. Request: `GET /api/media/?uploadType=video&status=processed&visibility=private&sort=fileSize&order=desc&limit=5`
3. Verify response matches all criteria
4. Check that only videos that are processed, private, and sorted by size descending are returned

**Expected Result**:
- Response contains only media matching ALL filter criteria
- Media items are videos (uploadType=video)
- All items have status=processed
- All items have visibility=private
- Items sorted by fileSize in descending order
- Maximum 5 items returned

**Status**: ⏳ Pending

---

#### TC-308: Empty Result Set

**Objective**: Verify handling of queries that return no results.

**Prerequisites**:
- User account with media (but no files matching the specific filter)

**Test Steps**:
1. Authenticate as user
2. Request media with filter that matches no files: `GET /api/media/?uploadType=document&status=failed`
3. Verify response structure

**Expected Result**:
- Status: 200 OK
- Response structure is valid
- Media array is empty: `[]`
- Pagination shows: total=0, pages=0
- No errors returned

**Status**: ⏳ Pending

---

#### TC-309: Page Limit Boundary (Max 100)

**Objective**: Verify that limit parameter is capped at 100 items per page.

**Prerequisites**:
- User account with at least 100 media files

**Test Steps**:
1. Authenticate as user
2. Request with limit=100: `GET /api/media/?limit=100`
3. Request with limit=150: `GET /api/media/?limit=150`
4. Request with limit=1000: `GET /api/media/?limit=1000`
5. Verify maximum items returned is 100

**Expected Result**:
- Limit=100: Returns exactly 100 items (if available)
- Limit=150: Returns maximum 100 items (capped)
- Limit=1000: Returns maximum 100 items (capped)
- Pagination metadata reflects actual limit applied
- No server errors from large limit values

**Status**: ⏳ Pending

---

#### TC-310: Minimum Page Number (Invalid Page 0)

**Objective**: Verify handling of invalid page numbers.

**Prerequisites**:
- User account with media files

**Test Steps**:
1. Authenticate as user
2. Request page 0: `GET /api/media/?page=0`
3. Request negative page: `GET /api/media/?page=-1`
4. Verify error handling or default behavior

**Expected Result**:
- Page 0 or negative pages: Should return validation error (400) or default to page 1
- Error message indicates page must be >= 1
- Response includes helpful error details

**Status**: ⏳ Pending

---

### Get Media by ID Tests

#### TC-311: Get Valid Media by ID

**Objective**: Verify retrieval of media details by valid ID.

**Prerequisites**:
- User account with at least one uploaded and processed media file
- Media ID known

**Test Steps**:
1. Authenticate as media owner
2. Send GET request to `/api/media/{mediaId}` with valid ID
3. Verify response contains complete media details
4. Check all fields are populated

**Expected Result**:
- Status: 200 OK
- Response includes all media fields: id, filename, sanitizedFilename, fileSize, mimeType, uploadType, status, visibility, publicUrl, thumbnailUrl, timestamps, metadata, timeline
- Timestamps include: uploadedAt, processedAt, lastAccessedAt, createdAt, updatedAt
- Metadata includes processing results
- Timeline data includes verification status if available

**Status**: ⏳ Pending

---

#### TC-312: Last Accessed Timestamp Updated

**Objective**: Verify that accessing media updates the lastAccessedAt timestamp.

**Prerequisites**:
- User account with media file

**Test Steps**:
1. Authenticate as user
2. Get media details and note lastAccessedAt timestamp
3. Wait 2 seconds
4. Get media details again
5. Compare lastAccessedAt timestamps

**Expected Result**:
- Second request shows lastAccessedAt updated to more recent time
- Timestamp difference matches wait time (approximately)
- Update is async, doesn't block response
- Other fields remain unchanged

**Status**: ⏳ Pending

---

#### TC-313: Get Non-Existent Media ID

**Objective**: Verify proper error handling for non-existent media IDs.

**Prerequisites**:
- User account authenticated

**Test Steps**:
1. Authenticate as user
2. Send GET request to `/api/media/507f1f77bcf86cd799439099` (valid format but doesn't exist)
3. Verify error response

**Expected Result**:
- Status: 404 Not Found
- Error message: "Media not found"
- Response structure includes success=false
- No media data in response

**Status**: ⏳ Pending

---

#### TC-314: Get Media with Invalid ID Format

**Objective**: Verify validation of media ID format.

**Prerequisites**:
- User account authenticated

**Test Steps**:
1. Authenticate as user
2. Send GET request with invalid ID formats:
   - `/api/media/invalid` (not hex)
   - `/api/media/123` (too short)
   - `/api/media/507f1f77bcf86cd799439011abc` (too long)
3. Verify validation errors

**Expected Result**:
- Status: 400 Bad Request
- Error message indicates invalid media ID format
- Message specifies ID must be 24 hex characters
- No database query attempted with invalid ID

**Status**: ⏳ Pending

---

#### TC-315: Get Other User's Media

**Objective**: Verify authorization prevents accessing other users' media.

**Prerequisites**:
- Two user accounts: User A and User B
- User A has uploaded media

**Test Steps**:
1. Authenticate as User A and upload media
2. Note the media ID
3. Authenticate as User B
4. Attempt to GET User A's media: `/api/media/{userAMediaId}`
5. Verify access denied

**Expected Result**:
- Status: 403 Forbidden
- Error message: "Access denied"
- No media details leaked in error response
- User B cannot see User A's media metadata

**Status**: ⏳ Pending

---

#### TC-316: Get Deleted Media

**Objective**: Verify that soft-deleted media cannot be retrieved.

**Prerequisites**:
- User account with media file

**Test Steps**:
1. Authenticate as user
2. Upload and confirm media, note ID
3. Delete the media: `DELETE /api/media/{mediaId}`
4. Attempt to get deleted media: `GET /api/media/{mediaId}`
5. Verify not found response

**Expected Result**:
- Status: 404 Not Found
- Error message: "Media not found"
- Deleted media not accessible via API
- isDeleted flag prevents retrieval

**Status**: ⏳ Pending

---

### Update Media Tests

#### TC-317: Update Media Visibility Only

**Objective**: Verify updating only the visibility setting.

**Prerequisites**:
- User account with private media file

**Test Steps**:
1. Authenticate as user
2. Get current media details, verify visibility=private
3. Send PATCH request to `/api/media/{mediaId}` with:
   ```json
   { "visibility": "public" }
   ```
4. Verify update response
5. Get media details again to confirm change

**Expected Result**:
- Status: 200 OK
- Response shows visibility updated to "public"
- Metadata unchanged
- updatedAt timestamp updated
- GET request confirms visibility is now "public"
- Other fields remain unchanged

**Status**: ⏳ Pending

---

#### TC-318: Update Media Metadata Only

**Objective**: Verify updating custom metadata without changing visibility.

**Prerequisites**:
- User account with media file

**Test Steps**:
1. Authenticate as user
2. Send PATCH request to `/api/media/{mediaId}` with:
   ```json
   {
     "metadata": {
       "description": "My custom description",
       "tags": ["test", "example"],
       "customField": "value123"
     }
   }
   ```
3. Verify response includes updated metadata
4. Get media details to confirm merge

**Expected Result**:
- Status: 200 OK
- New metadata fields added to existing metadata
- Original metadata (width, height, exif, etc.) preserved
- Custom fields accessible in response
- Metadata structure correctly merged

**Status**: ⏳ Pending

---

#### TC-319: Update Both Visibility and Metadata

**Objective**: Verify updating both visibility and metadata in single request.

**Prerequisites**:
- User account with media file

**Test Steps**:
1. Authenticate as user
2. Send PATCH request with both fields:
   ```json
   {
     "visibility": "unlisted",
     "metadata": {
       "description": "Unlisted test media",
       "category": "testing"
     }
   }
   ```
3. Verify both updates applied
4. Confirm with GET request

**Expected Result**:
- Status: 200 OK
- Visibility changed to "unlisted"
- Metadata merged with custom fields
- Both changes reflected in response
- Single database update performed

**Status**: ⏳ Pending

---

#### TC-320: Update with Invalid Visibility

**Objective**: Verify validation of visibility values.

**Prerequisites**:
- User account with media file

**Test Steps**:
1. Authenticate as user
2. Send PATCH request with invalid visibility:
   ```json
   { "visibility": "restricted" }
   ```
3. Send PATCH request with typo:
   ```json
   { "visibility": "privat" }
   ```
4. Verify validation errors

**Expected Result**:
- Status: 400 Bad Request
- Error message indicates invalid visibility setting
- Allowed values listed in response: "public", "private", "unlisted"
- No database update performed
- Original visibility unchanged

**Status**: ⏳ Pending

---

#### TC-321: Update with No Fields

**Objective**: Verify that at least one field is required for update.

**Prerequisites**:
- User account with media file

**Test Steps**:
1. Authenticate as user
2. Send PATCH request with empty body: `{}`
3. Send PATCH request with null values: `{ "visibility": null, "metadata": null }`
4. Verify validation error

**Expected Result**:
- Status: 400 Bad Request
- Error message: "At least one field (visibility or metadata) must be provided"
- No database update attempted
- Media remains unchanged

**Status**: ⏳ Pending

---

#### TC-322: Update Other User's Media

**Objective**: Verify authorization prevents updating other users' media.

**Prerequisites**:
- Two user accounts with media

**Test Steps**:
1. Authenticate as User A, note media ID
2. Authenticate as User B
3. Attempt to update User A's media:
   ```
   PATCH /api/media/{userAMediaId}
   { "visibility": "public" }
   ```
4. Verify access denied

**Expected Result**:
- Status: 403 Forbidden or 404 Not Found
- Error message: "Media not found" or "Access denied"
- No update performed
- User A's media unchanged

**Status**: ⏳ Pending

---

#### TC-323: Update Non-Existent Media

**Objective**: Verify error handling for updates to non-existent media.

**Prerequisites**:
- User account authenticated

**Test Steps**:
1. Authenticate as user
2. Send PATCH request to non-existent ID: `/api/media/507f1f77bcf86cd799439099`
   ```json
   { "visibility": "public" }
   ```
3. Verify not found response

**Expected Result**:
- Status: 404 Not Found
- Error message: "Media not found"
- No database modification
- Clear error message returned

**Status**: ⏳ Pending

---

#### TC-324: Metadata Merge Behavior

**Objective**: Verify that metadata updates merge with existing metadata rather than replace.

**Prerequisites**:
- User account with processed media containing extracted metadata

**Test Steps**:
1. Authenticate as user
2. Get media with existing metadata:
   ```json
   {
     "metadata": {
       "width": 1920,
       "height": 1080,
       "camera": "Canon EOS"
     }
   }
   ```
3. Update with new fields:
   ```json
   {
     "metadata": {
       "description": "New description"
     }
   }
   ```
4. Get media again and verify merge

**Expected Result**:
- Original metadata preserved: width, height, camera still present
- New field added: description now included
- Metadata object contains both original and new fields
- No data loss from update

**Status**: ⏳ Pending

---

### Delete Media Tests

#### TC-325: Delete Single Media

**Objective**: Verify successful deletion of a media file.

**Prerequisites**:
- User account with media file
- Note current storage quota before deletion

**Test Steps**:
1. Authenticate as user
2. Get current user stats to check quota
3. Delete media: `DELETE /api/media/{mediaId}`
4. Verify deletion response
5. Attempt to get deleted media (should fail)
6. Check user stats again to verify quota updated

**Expected Result**:
- Status: 200 OK
- Success message: "Media deleted successfully"
- GET request returns 404 Not Found
- Storage quota decreased by file size
- S3 deletion scheduled (check logs)
- Soft delete: isDeleted=true in database

**Status**: ⏳ Pending

---

#### TC-326: Delete Already Deleted Media

**Objective**: Verify handling of attempts to delete already-deleted media.

**Prerequisites**:
- User account with media file

**Test Steps**:
1. Authenticate as user
2. Delete media: `DELETE /api/media/{mediaId}`
3. Attempt to delete same media again
4. Verify error response

**Expected Result**:
- First delete: 200 OK
- Second delete: 404 Not Found
- Error message: "Media not found"
- Quota only decremented once
- Idempotent behavior

**Status**: ⏳ Pending

---

#### TC-327: Delete Other User's Media

**Objective**: Verify authorization prevents deleting other users' media.

**Prerequisites**:
- Two user accounts with media

**Test Steps**:
1. Authenticate as User A, upload media
2. Note media ID
3. Authenticate as User B
4. Attempt to delete User A's media: `DELETE /api/media/{userAMediaId}`
5. Verify access denied
6. Authenticate as User A and verify media still exists

**Expected Result**:
- Status: 404 Not Found (media not found in User B's library)
- User A's media unchanged
- User A's quota unchanged
- No S3 deletion initiated

**Status**: ⏳ Pending

---

#### TC-328: Delete Profile Picture

**Objective**: Verify that deleting a media file that is set as profile picture clears the profile picture reference.

**Prerequisites**:
- User account
- Media file set as profile picture

**Test Steps**:
1. Authenticate as user
2. Upload image and set as profile picture
3. Get user profile, verify profilePicture URL is set
4. Delete the profile picture media: `DELETE /api/media/{mediaId}`
5. Get user profile again
6. Verify profilePicture field is cleared

**Expected Result**:
- Media deletion successful
- Profile picture field cleared from user profile
- User profile still accessible
- No broken references remain

**Status**: ⏳ Pending

---

#### TC-329: Delete Non-Existent Media

**Objective**: Verify error handling for deletion of non-existent media.

**Prerequisites**:
- User account authenticated

**Test Steps**:
1. Authenticate as user
2. Attempt to delete with non-existent ID: `DELETE /api/media/507f1f77bcf86cd799439099`
3. Verify error response

**Expected Result**:
- Status: 404 Not Found
- Error message: "Media not found"
- No quota changes
- No S3 operations attempted

**Status**: ⏳ Pending

---

#### TC-330: Delete with Invalid ID Format

**Objective**: Verify validation of media ID format on deletion.

**Prerequisites**:
- User account authenticated

**Test Steps**:
1. Authenticate as user
2. Attempt deletion with invalid IDs:
   - `DELETE /api/media/invalid`
   - `DELETE /api/media/12345`
3. Verify validation errors

**Expected Result**:
- Status: 400 Bad Request
- Error message indicates invalid ID format
- No database queries with invalid format
- Clear error message returned

**Status**: ⏳ Pending

---

### Bulk Delete Tests

#### TC-331: Bulk Delete Multiple Media Successfully

**Objective**: Verify successful bulk deletion of multiple media files.

**Prerequisites**:
- User account with at least 5 media files
- Note current storage quota

**Test Steps**:
1. Authenticate as user
2. Get list of media IDs (at least 3)
3. Send DELETE request to `/api/media/bulk` with:
   ```json
   {
     "mediaIds": ["id1", "id2", "id3"]
   }
   ```
4. Verify response shows all deleted
5. Attempt to get each deleted media (should fail)
6. Check quota updated by sum of file sizes

**Expected Result**:
- Status: 200 OK
- Response: `{ deleted: 3, failed: 0, errors: [] }`
- All media return 404 on GET requests
- Storage quota decreased by total size of all deleted files
- Success message: "All files deleted successfully"

**Status**: ⏳ Pending

---

#### TC-332: Bulk Delete with Partial Failures

**Objective**: Verify graceful handling of partial failures in bulk delete.

**Prerequisites**:
- User account with 2 media files
- User B with 1 media file

**Test Steps**:
1. Authenticate as User A
2. Get User A's 2 media IDs
3. Get User B's 1 media ID (User A doesn't own this)
4. Send bulk delete with mix:
   ```json
   {
     "mediaIds": ["userAMedia1", "userAMedia2", "userBMedia1"]
   }
   ```
5. Verify partial success response

**Expected Result**:
- Status: 206 Partial Content
- Response: `{ deleted: 2, failed: 1, errors: ["userBMedia1: Media not found"] }`
- User A's media deleted successfully
- User B's media unchanged
- Quota updated only for successfully deleted files
- Detailed error information for failed deletions

**Status**: ⏳ Pending

---

#### TC-333: Bulk Delete - Empty Array

**Objective**: Verify validation of empty mediaIds array.

**Prerequisites**:
- User account authenticated

**Test Steps**:
1. Authenticate as user
2. Send bulk delete with empty array:
   ```json
   { "mediaIds": [] }
   ```
3. Verify validation error

**Expected Result**:
- Status: 400 Bad Request
- Error message: "mediaIds must be a non-empty array"
- No deletions performed
- Clear validation message

**Status**: ⏳ Pending

---

#### TC-334: Bulk Delete - Exceed Maximum (51 IDs)

**Objective**: Verify enforcement of 50-item maximum for bulk delete.

**Prerequisites**:
- User account authenticated
- 51 valid media IDs (can be duplicates for testing)

**Test Steps**:
1. Authenticate as user
2. Send bulk delete with 51 IDs:
   ```json
   {
     "mediaIds": ["id1", "id2", ..., "id51"]
   }
   ```
3. Verify validation error

**Expected Result**:
- Status: 400 Bad Request
- Error message: "Cannot delete more than 50 files at once"
- No deletions performed
- Limit clearly communicated

**Status**: ⏳ Pending

---

#### TC-335: Bulk Delete - Invalid ID Format

**Objective**: Verify validation of ID formats in bulk delete array.

**Prerequisites**:
- User account authenticated

**Test Steps**:
1. Authenticate as user
2. Send bulk delete with invalid IDs:
   ```json
   {
     "mediaIds": ["valid24hexstring123456", "invalid", "123"]
   }
   ```
3. Verify validation error

**Expected Result**:
- Status: 400 Bad Request
- Error indicates invalid media ID format
- Specifies which IDs are invalid
- No deletions performed (all-or-nothing validation)

**Status**: ⏳ Pending

---

#### TC-336: Bulk Delete - Duplicate IDs

**Objective**: Verify handling of duplicate IDs in bulk delete request.

**Prerequisites**:
- User account with media file

**Test Steps**:
1. Authenticate as user
2. Upload single media file, note ID
3. Send bulk delete with duplicate IDs:
   ```json
   {
     "mediaIds": ["mediaId1", "mediaId1", "mediaId1"]
   }
   ```
4. Verify response

**Expected Result**:
- Status: 206 Partial Content (or 200 OK)
- First deletion succeeds: deleted: 1
- Subsequent attempts fail: failed: 2, errors: ["mediaId1: Media not found", ...]
- Quota decremented only once
- Duplicate handling clearly reported

**Status**: ⏳ Pending

---

#### TC-337: Bulk Delete - All Non-Existent IDs

**Objective**: Verify handling when all IDs in bulk delete don't exist.

**Prerequisites**:
- User account authenticated

**Test Steps**:
1. Authenticate as user
2. Send bulk delete with all non-existent IDs:
   ```json
   {
     "mediaIds": ["507f1f77bcf86cd799439099", "507f1f77bcf86cd799439098"]
   }
   ```
3. Verify failure response

**Expected Result**:
- Status: 206 Partial Content
- Response: `{ deleted: 0, failed: 2, errors: ["id1: Media not found", "id2: Media not found"] }`
- No quota changes
- Clear error details for each ID

**Status**: ⏳ Pending

---

#### TC-338: Bulk Delete - Maximum 50 Valid IDs

**Objective**: Verify successful bulk deletion at maximum allowed limit.

**Prerequisites**:
- User account with 50+ media files

**Test Steps**:
1. Authenticate as user
2. Get 50 media IDs
3. Send bulk delete with exactly 50 IDs
4. Verify all 50 deleted successfully
5. Check quota updated by sum of all file sizes

**Expected Result**:
- Status: 200 OK
- Response: `{ deleted: 50, failed: 0, errors: [] }`
- All 50 media deleted
- Quota updated correctly
- Process completes in reasonable time (<10 seconds)

**Status**: ⏳ Pending

---

## Edge Cases

### EC-01: List Media with No Files

**Objective**: Verify behavior when user has no media files.

**Test Steps**:
1. Create new user account with verified email
2. Request media list: `GET /api/media/`
3. Verify empty result handling

**Expected Result**:
- Status: 200 OK
- Empty media array: `[]`
- Pagination: total=0, pages=0
- No errors

**Status**: ⏳ Pending

---

### EC-02: List Media with Special Characters in Sort Field

**Objective**: Verify handling of invalid or unsupported sort fields.

**Test Steps**:
1. Authenticate as user
2. Request with invalid sort field: `GET /api/media/?sort=invalidField`
3. Request with SQL injection attempt: `GET /api/media/?sort=createdAt;DROP TABLE media;`
4. Verify safe handling

**Expected Result**:
- Either defaults to safe field (createdAt) or returns validation error
- No database errors
- No SQL injection possible
- Safe fallback behavior

**Status**: ⏳ Pending

---

### EC-03: Get Media Immediately After Upload

**Objective**: Verify media is accessible immediately after confirmation, even if processing hasn't completed.

**Test Steps**:
1. Authenticate as user
2. Upload media and confirm
3. Immediately get media by ID (before processing completes)
4. Verify accessible with status=uploaded or processing

**Expected Result**:
- Status: 200 OK
- Media accessible before processing completes
- Status field indicates processing state
- Basic fields populated (no metadata yet)
- No timing-related errors

**Status**: ⏳ Pending

---

### EC-04: Update Media During Processing

**Objective**: Verify that visibility/metadata can be updated while processing is in progress.

**Test Steps**:
1. Authenticate as user
2. Upload large video file (processing takes time)
3. Immediately update visibility before processing completes
4. Verify update succeeds despite processing status

**Expected Result**:
- Update succeeds regardless of processing status
- Status remains "processing"
- Visibility change applied
- No conflicts between processing and update operations

**Status**: ⏳ Pending

---

### EC-05: Delete Media During Processing

**Objective**: Verify media can be deleted while processing is in progress.

**Test Steps**:
1. Authenticate as user
2. Upload media that triggers slow processing
3. Delete media before processing completes
4. Verify deletion succeeds and processing job handles deletion gracefully

**Expected Result**:
- Deletion succeeds immediately
- Processing job detects deletion and terminates gracefully
- Quota updated correctly
- No orphaned processing jobs
- Logs show graceful handling

**Status**: ⏳ Pending

---

### EC-06: Concurrent Updates to Same Media

**Objective**: Verify handling of concurrent update requests to same media file.

**Test Steps**:
1. Authenticate as user
2. Send two PATCH requests simultaneously:
   - Request A: `{ "visibility": "public" }`
   - Request B: `{ "visibility": "private" }`
3. Verify both succeed but last write wins
4. Check final state

**Expected Result**:
- Both requests return 200 OK
- Final state reflects last write (race condition acceptable)
- No database errors
- updatedAt reflects most recent update
- Data consistency maintained

**Status**: ⏳ Pending

---

### EC-07: Bulk Delete with One ID

**Objective**: Verify bulk delete works with minimum allowed input (1 ID).

**Test Steps**:
1. Authenticate as user
2. Send bulk delete with single ID:
   ```json
   { "mediaIds": ["507f1f77bcf86cd799439011"] }
   ```
3. Verify deletion succeeds

**Expected Result**:
- Status: 200 OK
- Response: `{ deleted: 1, failed: 0, errors: [] }`
- Functions same as single delete
- No errors with minimum input

**Status**: ⏳ Pending

---

### EC-08: Very Large Metadata Object

**Objective**: Verify handling of large custom metadata objects.

**Test Steps**:
1. Authenticate as user
2. Update media with very large metadata object (close to MongoDB document size limit):
   ```json
   {
     "metadata": {
       "field1": "very long string...",
       "field2": { "nested": "large object..." },
       // ... many fields
     }
   }
   ```
3. Verify handling (success or clear size limit error)

**Expected Result**:
- Either succeeds if within MongoDB limits
- Or returns clear error about document size
- No server crashes
- Error message helpful

**Status**: ⏳ Pending

---

### EC-09: List Media with Limit=1 (Extreme Pagination)

**Objective**: Verify pagination works with very small page size.

**Test Steps**:
1. Authenticate as user with 10 media files
2. Request: `GET /api/media/?limit=1`
3. Paginate through all 10 pages
4. Verify no performance issues or errors

**Expected Result**:
- Each page returns exactly 1 item
- Pagination: pages=10 (total items / 1)
- No duplicate items across pages
- hasNext and hasPrev correct for each page
- Performance acceptable

**Status**: ⏳ Pending

---

### EC-10: Update Metadata to Empty Object

**Objective**: Verify behavior when updating metadata to empty object.

**Test Steps**:
1. Authenticate as user with media containing metadata
2. Update with empty metadata:
   ```json
   { "metadata": {} }
   ```
3. Verify merge behavior

**Expected Result**:
- Empty object merged with existing metadata
- Existing metadata preserved (merge doesn't clear)
- OR metadata cleared if business logic requires
- Behavior is consistent and documented

**Status**: ⏳ Pending

---

## Security Tests

### SEC-01: List Media Without Authentication

**Objective**: Verify media listing requires authentication.

**Test Steps**:
1. Send GET request to `/api/media/` without Authorization header
2. Verify authentication required error

**Expected Result**:
- Status: 401 Unauthorized
- Error message: "Authentication required" or "No token provided"
- No media data returned
- No database query executed

**Status**: ⏳ Pending

---

### SEC-02: Access Media with Expired Token

**Objective**: Verify expired tokens are rejected.

**Test Steps**:
1. Create account and login to get token
2. Wait for token to expire (or use test token with past expiry)
3. Attempt to list media with expired token
4. Verify rejection

**Expected Result**:
- Status: 401 Unauthorized
- Error message: "Token expired" or similar
- No access granted with expired token
- User must re-authenticate

**Status**: ⏳ Pending

---

### SEC-03: Access Media with Unverified Email

**Objective**: Verify email verification requirement for media operations.

**Test Steps**:
1. Create account but don't verify email
2. Login and get token
3. Attempt to list media
4. Verify access denied

**Expected Result**:
- Status: 403 Forbidden
- Error message indicates email verification required
- No media operations allowed until verified
- Security requirement enforced

**Status**: ⏳ Pending

---

### SEC-04: MongoDB Injection in Filters

**Objective**: Verify protection against NoSQL injection in filter parameters.

**Test Steps**:
1. Authenticate as user
2. Attempt injection in uploadType: `GET /api/media/?uploadType[$ne]=video`
3. Attempt injection in status: `GET /api/media/?status[$regex]=.*`
4. Verify safe handling

**Expected Result**:
- Injection attempts treated as invalid values
- Returns validation error or empty results
- No unintended database behavior
- No access to other users' data
- Zod validation prevents injection

**Status**: ⏳ Pending

---

### SEC-05: Mass Assignment in Update Endpoint

**Objective**: Verify that protected fields cannot be updated through PATCH endpoint.

**Test Steps**:
1. Authenticate as user
2. Attempt to update protected fields:
   ```json
   {
     "userId": "differentUserId",
     "s3Key": "malicious-key",
     "fileSize": 0,
     "uploadedAt": "2020-01-01T00:00:00Z"
   }
   ```
3. Verify only allowed fields updated

**Expected Result**:
- Only visibility and metadata can be updated
- Protected fields ignored or error returned
- Original values unchanged
- No privilege escalation
- Schema validation prevents mass assignment

**Status**: ⏳ Pending

---

### SEC-06: Rate Limiting on List Endpoint

**Objective**: Verify rate limiting prevents excessive listing requests.

**Test Steps**:
1. Authenticate as user
2. Send rapid succession of list requests (>100 in 1 minute)
3. Verify rate limit enforcement

**Expected Result**:
- After threshold, returns 429 Too Many Requests
- Error message indicates rate limit exceeded
- Retry-After header present
- Limits abuse while allowing normal usage

**Status**: ⏳ Pending

---

### SEC-07: Ownership Verification in All Operations

**Objective**: Verify consistent authorization checks across all operations.

**Test Steps**:
1. User A uploads media, note ID
2. User B attempts:
   - Get media by ID
   - Update media
   - Delete media
   - Include in bulk delete
3. Verify all operations denied

**Expected Result**:
- All operations return 403/404
- No information leakage
- Consistent authorization enforcement
- User B cannot access/modify User A's media

**Status**: ⏳ Pending

---

### SEC-08: XSS in Custom Metadata

**Objective**: Verify XSS payloads in metadata are safely handled.

**Test Steps**:
1. Authenticate as user
2. Update media with XSS payload in metadata:
   ```json
   {
     "metadata": {
       "description": "<script>alert('XSS')</script>",
       "comment": "<img src=x onerror=alert('XSS')>"
     }
   }
   ```
3. Retrieve media and check response
4. Verify frontend sanitizes or escapes output

**Expected Result**:
- Metadata stored as-is (string)
- No script execution on retrieval
- Frontend responsible for sanitization on display
- API doesn't execute or process scripts
- Content-Type: application/json prevents browser execution

**Status**: ⏳ Pending

---

### SEC-09: Path Traversal in Sort Parameter

**Objective**: Verify protection against path traversal attempts in sort field.

**Test Steps**:
1. Authenticate as user
2. Attempt path traversal in sort: `GET /api/media/?sort=../../etc/passwd`
3. Attempt field access: `GET /api/media/?sort=user.password`
4. Verify safe handling

**Expected Result**:
- Invalid sort fields rejected or ignored
- Falls back to default sort
- No unauthorized field access
- No server errors
- Validation prevents path traversal

**Status**: ⏳ Pending

---

### SEC-10: Bulk Delete Authorization on Each Item

**Objective**: Verify bulk delete checks authorization for each individual media ID.

**Test Steps**:
1. User A uploads 2 media files
2. User B uploads 2 media files
3. User A attempts bulk delete with mix of both users' IDs
4. Verify only User A's media deleted

**Expected Result**:
- User A's media deleted: deleted=2
- User B's media rejected: failed=2
- Clear error messages for unauthorized IDs
- No cross-user deletion
- Authorization checked per item, not just bulk operation

**Status**: ⏳ Pending

---

## Performance Benchmarks

### Expected Response Times

| Operation | Expected Time | Maximum Acceptable |
|-----------|--------------|-------------------|
| List media (page 1, 20 items) | < 200ms | 500ms |
| List media with filters | < 300ms | 1s |
| Get media by ID | < 100ms | 300ms |
| Update media | < 200ms | 500ms |
| Delete media | < 300ms | 1s |
| Bulk delete (10 items) | < 1s | 3s |
| Bulk delete (50 items) | < 3s | 10s |

### Scalability Considerations

- **Pagination**: Supports efficient pagination for libraries with 10,000+ files
- **Indexing**: Database indexes on userId, status, uploadType, createdAt ensure fast queries
- **Soft Deletion**: Deleted media filtered via isDeleted index
- **Bulk Operations**: Process up to 50 items sequentially without timeout

---

## Test Results

### Summary

| Test Category | Total Tests | Passed | Failed | Blocked | Coverage |
|--------------|-------------|---------|---------|---------|----------|
| List Media Tests | 10 | 0 | 0 | 0 | 0% |
| Get Media Tests | 6 | 0 | 0 | 0 | 0% |
| Update Media Tests | 8 | 0 | 0 | 0 | 0% |
| Delete Media Tests | 6 | 0 | 0 | 0 | 0% |
| Bulk Delete Tests | 8 | 0 | 0 | 0 | 0% |
| Edge Cases | 10 | 0 | 0 | 0 | 0% |
| Security Tests | 10 | 0 | 0 | 0 | 0% |
| **TOTAL** | **58** | **0** | **0** | **0** | **0%** |

*This section will be updated as testing progresses.*

---

## Issues Found

No issues found yet. This section will be updated as testing progresses.

### Issue Template

```markdown
**Issue ID**: MED-XXX
**Title**: Brief description
**Severity**: P0 Critical / P1 High / P2 Medium / P3 Low
**Category**: Functional / Security / Performance / UX

**Description**: Detailed description of the issue

**Steps to Reproduce**:
1. Step one
2. Step two

**Expected Behavior**: What should happen

**Actual Behavior**: What actually happens

**Impact**: Effect on users or system

**Screenshots/Logs**: Evidence

**Suggested Fix**: Potential solution

**Status**: Open / In Progress / Fixed / Closed
```

---

## Recommendations

### High Priority

1. **Add Batch Get Endpoint**: Consider adding endpoint to retrieve multiple media by IDs in single request to reduce API calls from frontend

2. **Add Search Endpoint**: Implement full-text search across filename, metadata tags, descriptions for better media discovery

3. **Soft Delete Cleanup Job**: Implement scheduled job to permanently delete soft-deleted media after retention period (e.g., 30 days)

4. **Media Collections**: Consider allowing users to organize media into collections/albums for better organization

5. **Advanced Filtering**: Support range queries for fileSize (min/max), date ranges for uploadedAt, and combined text search

### Medium Priority

6. **Export Media List**: Add ability to export media list as CSV/JSON for backup or analysis purposes

7. **Media Restore**: Provide endpoint to restore soft-deleted media within retention period

8. **Thumbnail Batch Update**: Allow bulk thumbnail regeneration with custom settings for multiple videos

9. **Audit Log**: Track all media modifications (updates, deletes) for security and compliance

10. **Last Accessed Stats**: Provide analytics on media access patterns to identify unused files

### Low Priority

11. **Favorites/Starring**: Allow users to mark media as favorites for quick access

12. **Media Duplicates**: Detect and report duplicate media based on content hash

13. **Bulk Visibility Update**: Add endpoint to update visibility for multiple media files at once

14. **Download Statistics**: Track download counts and provide analytics per media file

15. **Custom Sort Orders**: Allow sorting by custom metadata fields

---

## Test Execution Notes

- All tests should be executed against a production-like staging environment
- Use fresh test accounts for authorization tests
- Document actual API responses for successful test cases
- Screenshot critical workflows and error states
- Note any deviations from expected behavior
- Report performance metrics for all benchmarked operations

---

**[← Previous: Media Processing](./02-media-processing.md)** | **[Next: Metadata Analysis & Tamper Detection →](./04-metadata-analysis.md)**

---

*Last Updated: December 2, 2025*
*Test Status: Not Started*
*Total Test Cases: 58*
