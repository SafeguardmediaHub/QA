# Feature 13: Analytics & Reporting

## Overview

The **Analytics & Reporting** feature provides comprehensive data export and analysis capabilities with support for multiple report types, formats, and priority levels. This enterprise-grade reporting system enables users and administrators to generate detailed insights from media analytics, user activity, security audits, and system performance data through asynchronous background processing.

### What is Analytics & Reporting?

Analytics & Reporting encompasses all data export and analysis operations:

1. **Report Generation**: Create custom reports with advanced filtering and date range selection
2. **Multiple Report Types**: 6 specialized report types for different use cases
3. **Format Flexibility**: Export in JSON, CSV, PDF, or Excel formats
4. **Priority Processing**: URGENT, HIGH, NORMAL, or LOW priority queue management
5. **Asynchronous Processing**: Background job processing with progress tracking
6. **Secure Downloads**: Time-limited S3 presigned URLs for report retrieval
7. **Lifecycle Management**: Automatic expiration and retention policy enforcement
8. **Audit Trail**: Complete access logging for compliance and security

This feature empowers users to:

- **Export Analysis Data**: Generate reports from deepfake detection results
- **Monitor User Activity**: Track user behavior, uploads, and platform usage
- **Audit Security Events**: Review authentication, access attempts, and security incidents
- **Analyze System Performance**: Monitor system health, resource usage, and capacity
- **Ensure Compliance**: Generate reports meeting regulatory requirements (GDPR, SOC 2)
- **Custom Filtering**: Apply date ranges, user filters, media type filters, and custom criteria

### Why is Analytics & Reporting Important?

- **Business Intelligence**: Data-driven decision making through comprehensive analytics
- **Compliance Requirements**: Meet regulatory reporting obligations (GDPR, HIPAA, SOC 2)
- **Operational Insights**: Identify trends, anomalies, and optimization opportunities
- **Audit Capability**: Track user activity and security events for forensic analysis
- **Resource Planning**: Understand storage usage and capacity requirements
- **Performance Monitoring**: Identify bottlenecks and performance degradation
- **Data Portability**: Export data for external analysis, backup, or migration

---

## Architecture

The Analytics & Reporting system uses asynchronous background processing with Bull/BullMQ queues:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   ANALYTICS & REPORTING ARCHITECTURE                     │
└─────────────────────────────────────────────────────────────────────────┘

Authentication & Authorization Layer
┌──────────────────────────────────────────────────────────────────────┐
│  All /api/reports/* routes require:                                  │
│  1. Valid JWT authentication                                         │
│  2. Email verification (requireEmailVerification middleware)         │
│                                                                       │
│  Role-Based Access (Report Types):                                   │
│    USER: MEDIA_ANALYTICS only                                        │
│    MODERATOR: MEDIA_ANALYTICS, USER_ACTIVITY, SYSTEM_OVERVIEW        │
│    ADMIN/SUPER_ADMIN: All report types                               │
│                                                                       │
│  Special Endpoints:                                                  │
│    GET /reports/queue/stats: Admin only                              │
└──────────────────────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      REPORT CREATION FLOW                            │
│                                                                      │
│  POST /api/reports                                                  │
│  ┌────────────────────────────────────────────────┐                │
│  │ Request:                                       │                │
│  │ {                                              │                │
│  │   "type": "media_analytics",                   │                │
│  │   "format": "pdf",                             │                │
│  │   "priority": "normal",                        │                │
│  │   "title": "Q4 2025 Analysis Report",          │                │
│  │   "description": "Quarterly review",           │                │
│  │   "analysisId": "analysis-123", // optional    │                │
│  │   "filters": {                                 │                │
│  │     "dateRange": {                             │                │
│  │       "startDate": "2025-10-01",               │                │
│  │       "endDate": "2025-12-31"                  │                │
│  │     },                                         │                │
│  │     "mediaTypes": ["image", "video"],          │                │
│  │     "predictedClasses": ["real", "fake"],      │                │
│  │     "confidenceMin": 80,                       │                │
│  │     "riskScoreMin": 70                         │                │
│  │   }                                            │                │
│  │ }                                              │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────┐                │
│  │ Validation Phase                               │                │
│  │                                                 │                │
│  │ 1. Check Report Type Permission                │                │
│  │    - USER can only create MEDIA_ANALYTICS      │                │
│  │    - MODERATOR can create 3 types              │                │
│  │    - ADMIN can create all 6 types              │                │
│  │                                                 │                │
│  │ 2. Validate Format                             │                │
│  │    - JSON, CSV, PDF, EXCEL                     │                │
│  │                                                 │                │
│  │ 3. Validate Priority                           │                │
│  │    - URGENT, HIGH, NORMAL, LOW                 │                │
│  │                                                 │                │
│  │ 4. Validate Filters (if provided)              │                │
│  │    - Date range: max 1 year                    │                │
│  │    - startDate < endDate                       │                │
│  │    - Valid enum values for filters             │                │
│  │    - Confidence/risk scores: 0-100             │                │
│  │                                                 │                │
│  │ 5. Check Pending Report Limit                  │                │
│  │    - Max 10 pending/processing reports         │                │
│  │    - Prevents queue flooding                   │                │
│  │                                                 │                │
│  │ 6. Validate Single Analysis Access (if ID)     │                │
│  │    - Analysis exists                           │                │
│  │    - User has access (owner or admin)          │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────┐                │
│  │ Report Record Creation                         │                │
│  │                                                 │                │
│  │ Create Report document:                        │                │
│  │   - requesterId: user ID                       │                │
│  │   - type, format, priority                     │                │
│  │   - filters                                    │                │
│  │   - title, description                         │                │
│  │   - status: PENDING                            │                │
│  │   - progress: 0                                │                │
│  │   - dataClassification: determined by type     │                │
│  │   - containsPII: boolean                       │                │
│  │   - retentionPeriod: 30-365 days               │                │
│  │   - estimatedCompletion: calculated            │                │
│  │   - expiresAt: completedAt + retention         │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────┐                │
│  │ Background Job Creation                        │                │
│  │                                                 │                │
│  │ Add job to reportQueue (Bull/BullMQ):          │                │
│  │   - jobId: unique identifier                   │                │
│  │   - reportId: MongoDB report ID                │                │
│  │   - type, format, filters, priority            │                │
│  │   - requesterId                                │                │
│  │                                                 │                │
│  │ Priority Queue Management:                     │                │
│  │   - URGENT: priority 1 (highest)               │                │
│  │   - HIGH: priority 3                           │                │
│  │   - NORMAL: priority 5                         │                │
│  │   - LOW: priority 10 (lowest)                  │                │
│  │                                                 │                │
│  │ Estimated Completion:                          │                │
│  │   - Base time by type (2-15 min)               │                │
│  │   - Priority multiplier (0.5x-1.5x)            │                │
│  │   - Filter complexity multiplier               │                │
│  │   - Date range multiplier                      │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────┐                │
│  │ Response (201 Created)                         │                │
│  │ {                                              │                │
│  │   "reportId": "report-abc123",                 │                │
│  │   "status": "pending",                         │                │
│  │   "estimatedCompletion": "2025-12-03T11:45",   │                │
│  │   "jobId": "job-xyz789"                        │                │
│  │ }                                              │                │
│  └────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   BACKGROUND PROCESSING (Worker)                     │
│                                                                      │
│  Worker Process (separate from API server)                          │
│  ┌────────────────────────────────────────────────┐                │
│  │ Job Pickup from Queue                          │                │
│  │   - Worker polls reportQueue                   │                │
│  │   - Picks jobs by priority order               │                │
│  │   - URGENT jobs processed first                │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────┐                │
│  │ Update Report Status: PROCESSING               │                │
│  │   - status = PROCESSING                        │                │
│  │   - startedAt = now()                          │                │
│  │   - progress = 10                              │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────┐                │
│  │ Data Retrieval & Aggregation                   │                │
│  │                                                 │                │
│  │ Based on Report Type:                          │                │
│  │                                                 │                │
│  │ MEDIA_ANALYTICS:                               │                │
│  │   - Query Analysis collection                  │                │
│  │   - Apply filters (date, mediaType, etc.)      │                │
│  │   - Aggregate: total analyses, by result,      │                │
│  │     confidence distribution, risk scores       │                │
│  │   - Include: fileName, uploadDate, userId,     │                │
│  │     predictedClass, confidence, riskScore      │                │
│  │                                                 │                │
│  │ USER_ACTIVITY:                                 │                │
│  │   - Query User, Media, Analysis collections    │                │
│  │   - Apply user filters, date ranges            │                │
│  │   - Aggregate: logins, uploads, analyses       │                │
│  │   - Include: userId, email, activity counts    │                │
│  │                                                 │                │
│  │ SECURITY_AUDIT:                                │                │
│  │   - Query audit logs, authentication logs      │                │
│  │   - Apply IP filters, user filters, date       │                │
│  │   - Aggregate: login attempts, failures,       │                │
│  │     suspicious activity                        │                │
│  │                                                 │                │
│  │ SYSTEM_OVERVIEW:                               │                │
│  │   - System metrics, queue stats, DB stats      │                │
│  │   - Performance metrics                        │                │
│  │                                                 │                │
│  │ STORAGE_USAGE:                                 │                │
│  │   - Query Media, User quota data               │                │
│  │   - Aggregate: total storage, per-user usage   │                │
│  │                                                 │                │
│  │ COMPLIANCE_REPORT:                             │                │
│  │   - GDPR, data retention, access logs          │                │
│  │   - User consent records                       │                │
│  │                                                 │                │
│  │ Progress updates: 30%, 50%, 70%                │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────┐                │
│  │ Format Generation                              │                │
│  │                                                 │                │
│  │ JSON:                                          │                │
│  │   - Native JavaScript object                   │                │
│  │   - JSON.stringify() with formatting           │                │
│  │                                                 │                │
│  │ CSV:                                           │                │
│  │   - Row-based flattening                       │                │
│  │   - Header row with column names               │                │
│  │   - Proper escaping (quotes, commas)           │                │
│  │                                                 │                │
│  │ PDF:                                           │                │
│  │   - PDFKit or similar library                  │                │
│  │   - Formatted tables, charts                   │                │
│  │   - Branding, headers, footers                 │                │
│  │                                                 │                │
│  │ EXCEL:                                         │                │
│  │   - xlsx library                               │                │
│  │   - Multiple sheets (summary + details)        │                │
│  │   - Formatting, formulas                       │                │
│  │                                                 │                │
│  │ Progress update: 90%                           │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────┐                │
│  │ S3 Upload                                      │                │
│  │   - Generate unique S3 key                     │                │
│  │   - Upload to S3 bucket                        │                │
│  │   - Set metadata (content-type, etc.)          │                │
│  │   - Store: s3Key, s3Bucket, fileSize           │                │
│  │   - Progress: 100%                             │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────┐                │
│  │ Update Report Status: COMPLETED                │                │
│  │   - status = COMPLETED                         │                │
│  │   - completedAt = now()                        │                │
│  │   - progress = 100                             │                │
│  │   - fileSize = actual bytes                    │                │
│  │   - metadata: totalRecords, processing time    │                │
│  │   - Calculate expiresAt (retention period)     │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────┐                │
│  │ Job Complete                                   │                │
│  │   - Worker marks job as complete               │                │
│  │   - Job result includes reportId, status       │                │
│  └────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      REPORT RETRIEVAL & DOWNLOAD                     │
│                                                                      │
│  GET /api/reports/:reportId                                         │
│  ┌────────────────────────────────────────────────┐                │
│  │ Retrieve Report Metadata                       │                │
│  │   - Check user access (owner or admin)         │                │
│  │   - Check expiration (expiresAt > now)         │                │
│  │   - Return: status, progress, metadata         │                │
│  │   - If completed: include downloadUrl          │                │
│  │   - Increment accessCount                      │                │
│  │   - Update lastAccessedAt                      │                │
│  │   - Log access in ReportAccess                 │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  GET /api/reports/:reportId/download                                │
│  ┌────────────────────────────────────────────────┐                │
│  │ Generate Download URL & Stream File            │                │
│  │                                                 │                │
│  │ 1. Validate Access                             │                │
│  │    - User is owner or admin                    │                │
│  │    - Report status = COMPLETED                 │                │
│  │    - Report not expired                        │                │
│  │                                                 │                │
│  │ 2. Generate S3 Presigned URL                   │                │
│  │    - Expiration: 15 minutes                    │                │
│  │    - Read-only access                          │                │
│  │                                                 │                │
│  │ 3. Set Response Headers                        │                │
│  │    - Content-Type: based on format             │                │
│  │    - Content-Disposition: attachment           │                │
│  │    - Filename: sanitized title + format        │                │
│  │    - Cache-Control: no-cache                   │                │
│  │                                                 │                │
│  │ 4. Stream from S3                              │                │
│  │    - Axios GET presigned URL (stream)          │                │
│  │    - Pipe to response                          │                │
│  │                                                 │                │
│  │ 5. Log Download                                │                │
│  │    - ReportAccess: download action             │                │
│  │    - Increment accessCount                     │                │
│  │    - Update lastAccessedAt                     │                │
│  └────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      LIFECYCLE MANAGEMENT                            │
│                                                                      │
│  Report Expiration                                                  │
│  ┌────────────────────────────────────────────────┐                │
│  │ expiresAt Calculation:                         │                │
│  │   completedAt + retentionPeriod                │                │
│  │                                                 │                │
│  │ Retention Periods (by classification):         │                │
│  │   - public: 30 days                            │                │
│  │   - internal: 90 days                          │                │
│  │   - confidential: 180 days                     │                │
│  │   - restricted: 365 days                       │                │
│  │                                                 │                │
│  │ Scheduled Cleanup Job:                         │                │
│  │   - Runs daily at midnight                     │                │
│  │   - Finds reports where expiresAt < now()      │                │
│  │   - Updates status = EXPIRED                   │                │
│  │   - Deletes S3 files                           │                │
│  │   - Soft-deletes report record (isDeleted)     │                │
│  └────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘

Data Classification & Retention
┌──────────────────────────────────────────────────────────────────────┐
│                                                                       │
│  Report Type          Classification    PII    Retention             │
│  ─────────────────────────────────────────────────────────           │
│  MEDIA_ANALYTICS      internal          No     90 days               │
│  USER_ACTIVITY        confidential      Yes    180 days              │
│  SECURITY_AUDIT       restricted        Yes    365 days              │
│  SYSTEM_OVERVIEW      internal          No     90 days               │
│  STORAGE_USAGE        internal          Yes    90 days               │
│  COMPLIANCE_REPORT    restricted        Yes    365 days              │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Report Types

### 1. MEDIA_ANALYTICS

**Description**: Deepfake detection analysis results and media processing statistics.

**Access**: All users (own media), Moderators, Admins

**Contains**:

- Total analyses performed
- Results by classification (real/fake)
- Confidence score distribution
- Risk score distribution
- Media type breakdown (image/video/audio)
- Upload trends over time
- User-specific analysis history

**Filters**:

- Date range
- Media types (image, video, audio, document)
- Predicted classes (real, fake, uncertain)
- Confidence range (min/max 0-100)
- Risk score range (min/max 0-100)
- Is deepfake (boolean)

**Data Classification**: Internal (90-day retention)
**Contains PII**: No

---

### 2. USER_ACTIVITY

**Description**: User behavior, uploads, login activity, and engagement metrics.

**Access**: Moderators, Admins only

**Contains**:

- User registration trends
- Login frequency and patterns
- Media upload statistics
- Feature usage metrics
- Geographic distribution
- Active vs inactive users
- Session duration averages

**Filters**:

- Date range
- User IDs (specific users)
- User roles
- Activity types

**Data Classification**: Confidential (180-day retention)
**Contains PII**: Yes (email, usernames, IP addresses)

---

### 3. SECURITY_AUDIT

**Description**: Security events, authentication attempts, access logs, and threat detection.

**Access**: Admins only

**Contains**:

- Failed login attempts
- Account lockouts
- Suspicious activity patterns
- IP address tracking
- Access control violations
- Token issuance/revocation
- Administrative actions

**Filters**:

- Date range
- IP addresses
- User IDs
- Event types

**Data Classification**: Restricted (365-day retention)
**Contains PII**: Yes (email, IP addresses, user agents)

---

### 4. SYSTEM_OVERVIEW

**Description**: System health, performance metrics, and operational statistics.

**Access**: Moderators, Admins

**Contains**:

- Queue statistics (pending jobs, processing times)
- Database metrics (collection sizes, query performance)
- API response times
- Error rates
- Resource utilization (CPU, memory, storage)
- Service uptime

**Filters**:

- Date range

**Data Classification**: Internal (90-day retention)
**Contains PII**: No

---

### 5. STORAGE_USAGE

**Description**: Storage consumption, quota tracking, and capacity planning.

**Access**: Admins only

**Contains**:

- Total storage used
- Per-user storage breakdown
- Storage trends over time
- Quota utilization percentages
- File count statistics
- Largest users by storage

**Filters**:

- Date range
- User IDs

**Data Classification**: Internal (90-day retention)
**Contains PII**: Yes (user IDs, email)

---

### 6. COMPLIANCE_REPORT

**Description**: GDPR/CCPA compliance, data retention, consent records, and access rights.

**Access**: Admins only

**Contains**:

- User consent records
- Data retention compliance
- Right to erasure requests
- Data access requests
- Data portability exports
- Privacy policy acceptances
- Cookie consent tracking

**Filters**:

- Date range
- Compliance type (GDPR, CCPA)

**Data Classification**: Restricted (365-day retention)
**Contains PII**: Yes (comprehensive user data)

---

## Real-World Test Scenarios

### Scenario 1: Generate Quarterly Media Analytics Report ✅

**Context**: User wants PDF report of Q4 2025 deepfake detections for stakeholder presentation.

**Test Flow**:

1. User: POST /reports with:
   - type: "media_analytics"
   - format: "pdf"
   - filters.dateRange: Oct 1 - Dec 31, 2025
   - filters.predictedClasses: ["fake"]
   - filters.confidenceMin: 80
   - title: "Q4 2025 Deepfake Detection Report"
2. System validates filters, creates report (status: PENDING)
3. Returns 201 with reportId, estimatedCompletion (5 min)
4. Background worker picks up job
5. Worker updates status: PROCESSING, progress: 10%
6. Worker queries Analysis collection:
   - dateRange filter applied
   - predictedClass = "fake"
   - confidence >= 80
   - Returns 1247 matching analyses
7. Worker aggregates data:
   - Total deepfakes: 1247
   - By media type: images (892), videos (355)
   - Average confidence: 94.3%
   - Risk score distribution
8. Worker generates PDF:
   - Cover page with title, date range
   - Executive summary
   - Detailed tables and charts
   - Progress: 90%
9. Worker uploads to S3
10. Status: COMPLETED, fileSize: 2.3 MB
11. User polls GET /reports/:reportId → status complete
12. User: GET /reports/:reportId/download
13. PDF downloaded successfully

**Expected Result**:

- Report generated in ~5 minutes
- PDF contains accurate filtered data (1247 records)
- Download works seamlessly
- Report expires after 90 days

---

### Scenario 2: Admin Generates Security Audit for Compliance ✅

**Context**: Admin needs security audit for SOC 2 compliance review.

**Test Flow**:

1. Admin: POST /reports:
   - type: "security_audit"
   - format: "excel"
   - priority: "high"
   - filters.dateRange: Last 365 days
2. System validates admin role, creates report
3. Estimated completion: 7 min (base 10 min × 0.7 high priority)
4. Worker aggregates:
   - Failed login attempts: 234
   - Account lockouts: 12
   - Suspicious IP addresses: 8
   - Admin actions: 567
5. Generates Excel with multiple sheets:
   - Sheet 1: Summary dashboard
   - Sheet 2: Failed logins (details)
   - Sheet 3: Lockouts
   - Sheet 4: Admin activity log
6. Upload to S3 (3.7 MB)
7. Status: COMPLETED
8. Admin downloads Excel
9. Report retained for 365 days (restricted classification)

**Expected Result**:

- Comprehensive security data exported
- Multiple sheets for organization
- Contains PII (flagged containsPII: true)
- Long retention for compliance

---

### Scenario 3: User Exceeds Pending Report Limit ✅

**Context**: User creates many reports rapidly and hits limit.

**Test Flow**:

1. User creates 10 reports (all pending/processing)
2. User attempts 11th report: POST /reports
3. System checks: countDocuments where requesterId = user AND status IN [PENDING, PROCESSING]
4. Count: 10 (limit reached)
5. Returns 429 Too Many Requests:
   - "Too many pending reports. Please wait for existing reports to complete."

**Expected Result**:

- Request rejected
- User must wait for reports to complete/cancel
- Prevents queue flooding

---

### Scenario 4: Report Expires After Retention Period ✅

**Context**: 90-day retention period elapses for media analytics report.

**Test Flow**:

1. Report completed on Dec 3, 2025 (MEDIA_ANALYTICS)
2. expiresAt = completedAt + 90 days = March 3, 2026
3. User can download Dec 3, 2025 - March 2, 2026
4. March 4, 2026: Scheduled cleanup job runs
5. Job finds report where expiresAt < now()
6. Updates status = EXPIRED
7. Deletes S3 file (reports/report-xyz789.pdf)
8. Soft-deletes report record (isDeleted = true)
9. User attempts download March 5, 2026
10. GET /reports/:reportId/download returns 410 Gone

**Expected Result**:

- Report automatically expires
- S3 storage freed
- User notified report expired
- Audit log preserved

---

### Scenario 5: Generate Report for Single Analysis ✅

**Context**: User wants detailed PDF report for specific suspicious video analysis.

**Test Flow**:

1. User analyzed "suspicious_video.mp4" → analysisId: "analysis-abc123"
2. User: POST /reports:
   - type: "media_analytics"
   - analysisId: "analysis-abc123"
   - format: "pdf"
   - title: "Analysis Report - suspicious_video.mp4"
3. System validates access to analysis (user is owner)
4. Creates report for single analysis (no filters)
5. Worker retrieves analysis record:
   - fileName: "suspicious_video.mp4"
   - predictedClass: "fake"
   - confidence: 97.3%
   - riskScore: 94
   - detectionMethods: {facial_manipulation: true, audio_mismatch: true}
6. Generates detailed single-analysis PDF:
   - Video thumbnail
   - Detection results
   - Confidence breakdown by method
   - Timeline if available
   - Metadata
7. Upload to S3
8. User downloads comprehensive analysis report

**Expected Result**:

- Single analysis report generated
- More detailed than filtered report
- Quick generation (small dataset)

---

### Scenario 6: Cancel Long-Running Report ✅

**Context**: User realizes report has wrong filters and cancels.

**Test Flow**:

1. User: POST /reports (large date range, 1-year)
2. Report starts processing (estimated 20 min)
3. User realizes mistake after 5 minutes
4. User: DELETE /reports/:reportId
5. System validates user is owner
6. Checks status: PROCESSING (can cancel)
7. Cancels Bull job via jobId
8. Updates report:
   - status = CANCELLED
   - completedAt = now()
9. Worker stops processing (graceful shutdown)
10. Returns 200 OK
11. User creates corrected report

**Expected Result**:

- Report cancelled mid-processing
- Worker stops gracefully
- Partial work discarded
- User can create new report immediately

---

## Test Cases

### Report Creation Tests (POST /reports)

#### TC-REPORT-001: Create Media Analytics Report with Filters ✅

**Preconditions**: User authenticated, emailVerified: true
**Steps**:

1. POST /api/reports:
   ```json
   {
     "type": "media_analytics",
     "format": "pdf",
     "filters": {
       "dateRange": {
         "startDate": "2025-11-01",
         "endDate": "2025-11-30"
       },
       "predictedClasses": ["fake"],
       "confidenceMin": 80
     }
   }
   ```

**Expected Result**:

- 201 Created
- Response includes: reportId, status: "pending", estimatedCompletion, jobId
- Report record created in database
- Bull job added to queue
- estimatedCompletion calculated based on filters

---

#### TC-REPORT-002: Create Single Analysis Report ✅

**Preconditions**: User has analysis "analysis-abc123"
**Steps**:

1. POST /reports:
   ```json
   {
     "type": "media_analytics",
     "analysisId": "analysis-abc123",
     "format": "json",
     "priority": "high"
   }
   ```

**Expected Result**:

- 201 Created
- Report linked to specific analysis
- Faster processing (small dataset)
- Title auto-generated if not provided

---

#### TC-REPORT-003: Invalid Date Range (Start > End) ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports with:
   ```json
   {
     "filters": {
       "dateRange": {
         "startDate": "2025-12-31",
         "endDate": "2025-01-01"
       }
     }
   }
   ```

**Expected Result**:

- 400 Bad Request
- Error: "Start date must be before end date"
- No report created

---

#### TC-REPORT-004: Date Range Exceeds 1 Year ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports with dateRange spanning 400 days

**Expected Result**:

- 400 Bad Request
- Error: "Date range cannot exceed 1 year"

---

#### TC-REPORT-005: Invalid Report Type ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports with type: "invalid_type"

**Expected Result**:

- 400 Bad Request
- Error: "Invalid report type"
- data.allowedTypes: array of valid types

---

#### TC-REPORT-006: Invalid Format ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports with format: "xml"

**Expected Result**:

- 400 Bad Request
- Error: "Invalid report format"
- data.allowedFormats: ["json", "csv", "pdf", "excel"]

---

#### TC-REPORT-007: User Attempts Admin-Only Report Type ✅

**Preconditions**: Regular user (role: "user")
**Steps**:

1. POST /reports with type: "security_audit"

**Expected Result**:

- 403 Forbidden
- Error: "Insufficient permissions for this report type"

---

#### TC-REPORT-008: Moderator Creates Allowed Report Type ✅

**Preconditions**: Moderator role
**Steps**:

1. POST /reports with type: "user_activity"

**Expected Result**:

- 201 Created
- Moderators can create USER_ACTIVITY, SYSTEM_OVERVIEW, MEDIA_ANALYTICS

---

#### TC-REPORT-009: Exceed Pending Report Limit ✅

**Preconditions**: User has 10 pending/processing reports
**Steps**:

1. POST /reports (11th report)

**Expected Result**:

- 429 Too Many Requests
- Error: "Too many pending reports. Please wait for existing reports to complete."

---

#### TC-REPORT-010: Rate Limit Exceeded ✅

**Preconditions**: User created 20 reports in last hour
**Steps**:

1. POST /reports (21st request within hour)

**Expected Result**:

- 429 Too Many Requests
- Error: "Too many report requests. Please try again later."
- Retry-After: "3600" (seconds)

---

#### TC-REPORT-011: Analysis Access Validation ✅

**Preconditions**: Analysis belongs to different user
**Steps**:

1. User A: POST /reports with analysisId owned by User B

**Expected Result**:

- 403 Forbidden OR 404 Not Found
- Error: "Access denied to this analysis"

---

#### TC-REPORT-012: Non-Existent Analysis ID ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports with analysisId: "nonexistent-id"

**Expected Result**:

- 404 Not Found
- Error: "Analysis not found"

---

#### TC-REPORT-013: Create with URGENT Priority ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports with priority: "urgent"

**Expected Result**:

- 201 Created
- estimatedCompletion reduced by 50% (0.5× multiplier)
- Job added to queue with high priority (priority: 1)

---

#### TC-REPORT-014: Invalid Confidence Range ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports with filters:
   ```json
   {
     "confidenceMin": 90,
     "confidenceMax": 70
   }
   ```

**Expected Result**:

- 400 Bad Request
- Error: "Confidence minimum cannot be greater than maximum"

---

#### TC-REPORT-015: Unverified Email Access ✅

**Preconditions**: User authenticated but emailVerified: false
**Steps**:

1. POST /reports

**Expected Result**:

- 403 Forbidden
- Error: "Email verification required"
- requireEmailVerification middleware blocks

---

### Report Retrieval Tests (GET /reports/:reportId)

#### TC-REPORT-016: Retrieve Pending Report ✅

**Preconditions**: Report created, status: PENDING
**Steps**:

1. GET /reports/:reportId

**Expected Result**:

- 200 OK
- report.status: "pending"
- report.progress: 0
- report.downloadUrl: null
- jobStatus.state: "waiting"

---

#### TC-REPORT-017: Retrieve Processing Report ✅

**Preconditions**: Report being processed
**Steps**:

1. GET /reports/:reportId

**Expected Result**:

- 200 OK
- report.status: "processing"
- report.progress: 0-100 (reflects actual progress)
- report.metadata.processedRecords present
- jobStatus.state: "active"

---

#### TC-REPORT-018: Retrieve Completed Report ✅

**Preconditions**: Report completed
**Steps**:

1. GET /reports/:reportId

**Expected Result**:

- 200 OK
- report.status: "completed"
- report.progress: 100
- report.downloadUrl: presigned S3 URL
- report.fileSize: actual bytes
- report.completedAt: timestamp
- report.expiresAt: completedAt + retentionPeriod

---

#### TC-REPORT-019: Retrieve Failed Report ✅

**Preconditions**: Report failed during processing
**Steps**:

1. GET /reports/:reportId

**Expected Result**:

- 200 OK
- report.status: "failed"
- report.errorMessage: detailed error description
- report.downloadUrl: null

---

#### TC-REPORT-020: Access Other User's Report ✅

**Preconditions**: Report belongs to User B
**Steps**:

1. User A: GET /reports/user_b_report_id

**Expected Result**:

- 403 Forbidden
- Error: "Access denied"

---

#### TC-REPORT-021: Admin Access Any Report ✅

**Preconditions**: Admin authenticated
**Steps**:

1. Admin: GET /reports/any_user_report_id

**Expected Result**:

- 200 OK
- Admins can access all reports

---

#### TC-REPORT-022: Non-Existent Report ID ✅

**Preconditions**: User authenticated
**Steps**:

1. GET /reports/nonexistent-report-id

**Expected Result**:

- 404 Not Found
- Error: "Report not found"

---

#### TC-REPORT-023: Retrieve Expired Report ✅

**Preconditions**: Report expired (expiresAt < now)
**Steps**:

1. GET /reports/:expiredReportId

**Expected Result**:

- 410 Gone
- Error: "Report has expired"

---

#### TC-REPORT-024: Access Count Increment ✅

**Preconditions**: Report accessed once (accessCount: 1)
**Steps**:

1. GET /reports/:reportId
2. Check report.accessCount

**Expected Result**:

- accessCount: 2
- lastAccessedAt: updated to current time

---

### Report Download Tests (GET /reports/:reportId/download)

#### TC-REPORT-025: Download Completed PDF Report ✅

**Preconditions**: Report completed, format: PDF
**Steps**:

1. GET /reports/:reportId/download

**Expected Result**:

- 200 OK
- Content-Type: application/pdf
- Content-Disposition: attachment; filename="Report_Title_report-id.pdf"
- Body: PDF binary data
- accessCount incremented
- Download logged in ReportAccess

---

#### TC-REPORT-026: Download JSON Report ✅

**Preconditions**: Report completed, format: JSON
**Steps**:

1. GET /reports/:reportId/download

**Expected Result**:

- 200 OK
- Content-Type: application/json
- JSON data in response body

---

#### TC-REPORT-027: Download CSV Report ✅

**Preconditions**: Report completed, format: CSV
**Steps**:

1. GET /reports/:reportId/download

**Expected Result**:

- 200 OK
- Content-Type: text/csv
- CSV data with proper escaping

---

#### TC-REPORT-028: Download Excel Report ✅

**Preconditions**: Report completed, format: Excel
**Steps**:

1. GET /reports/:reportId/download

**Expected Result**:

- 200 OK
- Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
- Excel file (.xlsx) downloaded

---

#### TC-REPORT-029: Attempt Download Before Completion ✅

**Preconditions**: Report status: PROCESSING
**Steps**:

1. GET /reports/:reportId/download

**Expected Result**:

- 400 Bad Request
- Error: "Report is not ready for download"
- data.status: "processing", data.progress: current progress

---

#### TC-REPORT-030: Download Expired Report ✅

**Preconditions**: Report expired
**Steps**:

1. GET /reports/:reportId/download

**Expected Result**:

- 410 Gone
- Error: "Report has expired"

---

#### TC-REPORT-031: S3 Presigned URL Generation ✅

**Preconditions**: Report completed
**Steps**:

1. GET /reports/:reportId/download
2. Inspect download URL generation

**Expected Result**:

- Presigned URL valid for 15 minutes
- URL includes signature parameter
- Read-only access
- After 15 minutes, URL expires

---

#### TC-REPORT-032: Download with Special Characters in Title ✅

**Preconditions**: Report title: "Q4 2025 Report: Análysis & Review"
**Steps**:

1. GET /reports/:reportId/download

**Expected Result**:

- Filename sanitized: "Q4_2025_Report_Analisis_Review_report-id.pdf"
- Special characters removed/replaced
- Safe filename for all operating systems

---

### Report Listing Tests (GET /reports)

#### TC-REPORT-033: List All User Reports ✅

**Preconditions**: User has 15 reports
**Steps**:

1. GET /reports

**Expected Result**:

- 200 OK
- Returns first 10 reports (default limit)
- Sorted by createdAt descending (newest first)
- pagination.total: 15
- pagination.hasMore: true

---

#### TC-REPORT-034: Filter by Status ✅

**Preconditions**: User has completed and pending reports
**Steps**:

1. GET /reports?status=completed

**Expected Result**:

- Returns only completed reports
- Pending, processing reports excluded

---

#### TC-REPORT-035: Filter by Report Type ✅

**Preconditions**: User has various report types
**Steps**:

1. GET /reports?type=media_analytics

**Expected Result**:

- Returns only media_analytics reports
- Other types excluded

---

#### TC-REPORT-036: Filter by Date Range ✅

**Preconditions**: Reports created over 6 months
**Steps**:

1. GET /reports?dateRange=2025-11-01,2025-11-30

**Expected Result**:

- Returns only reports created in November 2025
- Reports outside range excluded

---

#### TC-REPORT-037: Pagination ✅

**Preconditions**: User has 50 reports
**Steps**:

1. GET /reports?limit=20&offset=0 (page 1)
2. GET /reports?limit=20&offset=20 (page 2)
3. GET /reports?limit=20&offset=40 (page 3)

**Expected Result**:

- Page 1: Reports 1-20
- Page 2: Reports 21-40
- Page 3: Reports 41-50
- No duplicates

---

#### TC-REPORT-038: Sort by Completion Date ✅

**Preconditions**: Multiple completed reports
**Steps**:

1. GET /reports?sort=completedAt&order=desc

**Expected Result**:

- Reports sorted by completedAt (most recent first)
- Pending reports (no completedAt) appear last

---

#### TC-REPORT-039: Admin Lists All Reports ✅

**Preconditions**: Admin authenticated
**Steps**:

1. Admin: GET /reports

**Expected Result**:

- Returns reports from all users
- requester field populated with user info
- Allows admin to monitor system-wide reports

---

#### TC-REPORT-040: Empty Results ✅

**Preconditions**: User has no reports
**Steps**:

1. GET /reports

**Expected Result**:

- 200 OK
- reports: [] (empty array)
- pagination.total: 0

---

### Report Cancellation Tests (DELETE /reports/:reportId)

#### TC-REPORT-041: Cancel Pending Report ✅

**Preconditions**: Report status: PENDING
**Steps**:

1. DELETE /reports/:reportId

**Expected Result**:

- 200 OK
- report.status: "cancelled"
- report.completedAt: current timestamp
- Bull job cancelled
- ReportAccess logs cancellation

---

#### TC-REPORT-042: Cancel Processing Report ✅

**Preconditions**: Report status: PROCESSING
**Steps**:

1. DELETE /reports/:reportId

**Expected Result**:

- 200 OK
- Worker gracefully stops processing
- Partial work discarded
- Status: CANCELLED

---

#### TC-REPORT-043: Attempt Cancel Completed Report ✅

**Preconditions**: Report status: COMPLETED
**Steps**:

1. DELETE /reports/:reportId

**Expected Result**:

- 400 Bad Request
- Error: "Cannot cancel report in current status"
- data.currentStatus: "completed"

---

#### TC-REPORT-044: Attempt Cancel Failed Report ✅

**Preconditions**: Report status: FAILED
**Steps**:

1. DELETE /reports/:reportId

**Expected Result**:

- 400 Bad Request
- Cannot cancel already-failed report

---

#### TC-REPORT-045: Cancel Own Report ✅

**Preconditions**: User owns report
**Steps**:

1. DELETE /reports/:reportId

**Expected Result**:

- 200 OK
- User can cancel own reports

---

#### TC-REPORT-046: Attempt Cancel Other User's Report ✅

**Preconditions**: Report belongs to User B
**Steps**:

1. User A: DELETE /reports/user_b_report_id

**Expected Result**:

- 403 Forbidden
- Error: "Access denied"

---

#### TC-REPORT-047: Admin Cancels Any Report ✅

**Preconditions**: Admin authenticated
**Steps**:

1. Admin: DELETE /reports/any_report_id

**Expected Result**:

- 200 OK
- Admins can cancel any report

---

### Queue Statistics Tests (GET /reports/queue/stats)

#### TC-REPORT-048: Retrieve Queue Stats as Admin ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /reports/queue/stats

**Expected Result**:

- 200 OK
- queue object: waiting, active, completed, failed, delayed, paused counts
- database object: totalReports, recentReports, byStatus

---

#### TC-REPORT-049: Non-Admin Access to Queue Stats ✅

**Preconditions**: Regular user authenticated
**Steps**:

1. GET /reports/queue/stats

**Expected Result**:

- 403 Forbidden
- Error: "Access denied"

---

#### TC-REPORT-050: Queue Stats Accuracy ✅

**Preconditions**: Known queue state (e.g., 5 waiting, 2 active)
**Steps**:

1. GET /reports/queue/stats
2. Verify counts match actual queue state

**Expected Result**:

- Counts accurate
- Reflects real-time queue status

---

### Edge Cases and Error Handling

#### TC-REPORT-051: Very Large Date Range (1 Year Max) ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports with dateRange: Jan 1, 2025 - Dec 31, 2025 (exactly 1 year)

**Expected Result**:

- 201 Created
- Accepted (max limit)
- estimatedCompletion increased (dateRange multiplier: 2.0×)

---

#### TC-REPORT-052: Empty Filters Object ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports with filters: {}

**Expected Result**:

- 201 Created
- No filters applied (generates report for all data)

---

#### TC-REPORT-053: Null vs Omitted Filters ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports without filters field (omitted)
2. POST /reports with filters: null

**Expected Result**:

- Both accepted
- Treated as no filters

---

#### TC-REPORT-054: Concurrent Report Requests ✅

**Preconditions**: User authenticated
**Steps**:

1. Send 5 simultaneous POST /reports requests

**Expected Result**:

- All 5 created successfully (if under pending limit)
- Each gets unique reportId and jobId
- All processed in queue

---

#### TC-REPORT-055: Database Connection Failure During Processing ✅

**Preconditions**: Report processing, DB connection lost
**Steps**:

1. Report in PROCESSING state
2. Simulate DB connection failure

**Expected Result**:

- Worker fails gracefully
- report.status: "failed"
- report.errorMessage: "Database error during data retrieval"
- Job marked as failed in queue

---

#### TC-REPORT-056: S3 Upload Failure ✅

**Preconditions**: Report generated, S3 upload fails
**Steps**:

1. Report data generated successfully
2. S3 upload encounters error (network, credentials, etc.)

**Expected Result**:

- report.status: "failed"
- report.errorMessage: "Failed to upload report to storage"
- Job retried (exponential backoff)
- After max retries: marked as failed

---

#### TC-REPORT-057: Invalid MongoDB ObjectId Format ✅

**Preconditions**: User authenticated
**Steps**:

1. GET /reports/invalid-id-format

**Expected Result**:

- 400 Bad Request
- Validation error: "Invalid report ID format"

---

#### TC-REPORT-058: Report Title with Special Characters ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports with title: "Q4 2025: Análysis & Review <script>alert(1)</script>"

**Expected Result**:

- 201 Created
- Title stored as-is in database
- When used in filename: sanitized/escaped
- XSS prevented in frontend rendering

---

#### TC-REPORT-059: Extremely Long Description ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /reports with description: 10,000 characters

**Expected Result**:

- 201 Created OR
- 400 Bad Request if validation limits description length
- Database field truncated if applicable

---

#### TC-REPORT-060: Retry Failed Report ✅

**Preconditions**: Report failed due to temporary error
**Steps**:

1. User creates identical report again

**Expected Result**:

- New report created
- New jobId assigned
- Previous failed report remains in history

---

### Security Tests

#### TC-REPORT-SEC-001: Unauthenticated Access ✅

**Steps**:

1. POST /reports without JWT token

**Expected Result**:

- 401 Unauthorized
- Error: "Authentication required"

---

#### TC-REPORT-SEC-002: Unverified Email Access ✅

**Steps**:

1. User with emailVerified: false attempts POST /reports

**Expected Result**:

- 403 Forbidden
- Error: "Email verification required"

---

#### TC-REPORT-SEC-003: SQL Injection in Filters ✅

**Steps**:

1. POST /reports with filters containing SQL injection:
   ```json
   { "search": "'; DROP TABLE reports; --" }
   ```

**Expected Result**:

- Treated as literal string
- No SQL execution (MongoDB uses BSON)
- No security impact

---

#### TC-REPORT-SEC-004: NoSQL Injection in Filters ✅

**Steps**:

1. POST /reports with filters: {"mediaTypes": {"$ne": null}}

**Expected Result**:

- 400 Bad Request
- Validation rejects non-array values
- NoSQL injection prevented

---

#### TC-REPORT-SEC-005: Path Traversal in Report ID ✅

**Steps**:

1. GET /reports/../../etc/passwd

**Expected Result**:

- 400 Bad Request OR 404 Not Found
- Route validation prevents path traversal

---

#### TC-REPORT-SEC-006: Excessive File Size (DoS) ✅

**Preconditions**: Malicious user creates report with huge dataset
**Steps**:

1. Create report for 1-year dateRange, all media types (millions of records)

**Expected Result**:

- Report processes but may take longer
- Worker timeout prevents indefinite processing
- File size limits enforced (if implemented)
- System remains stable

---

#### TC-REPORT-SEC-007: Rate Limiting Bypass Attempt ✅

**Steps**:

1. Attempt 21 reports using multiple IP addresses/tokens

**Expected Result**:

- Rate limiting per user ID (not IP)
- All attempts from same user blocked after limit
- IP rotation doesn't bypass

---

#### TC-REPORT-SEC-008: Access Control Bypass Attempt ✅

**Steps**:

1. User modifies reportId in request to access another user's report

**Expected Result**:

- Server-side validation checks req.user.id against report.requesterId
- 403 Forbidden if mismatch
- Cannot bypass by tampering request

---

#### TC-REPORT-SEC-009: Audit Log Completeness ✅

**Steps**:

1. Perform various report operations (create, view, download, cancel)
2. Check ReportAccess audit logs

**Expected Result**:

- All actions logged with:
  - reportId
  - userId
  - action (generate, view, download, delete)
  - success/failure
  - ipAddress
  - userAgent
  - timestamp

---

#### TC-REPORT-SEC-010: PII Exposure in Reports ✅

**Steps**:

1. Create USER_ACTIVITY or SECURITY_AUDIT report (containsPII: true)
2. Share report link with unauthorized user

**Expected Result**:

- Access control prevents viewing
- PII not exposed to unauthorized users
- Reports marked with containsPII flag

---

### Performance Tests

#### TC-REPORT-PERF-001: Report Creation Response Time ✅

**Steps**:

1. POST /reports
2. Measure response time

**Expected Result**:

- Response time: < 500ms
- Report created, job added to queue
- Quick synchronous operation

---

#### TC-REPORT-PERF-002: Report Retrieval Response Time ✅

**Steps**:

1. GET /reports/:reportId
2. Measure response time

**Expected Result**:

- Response time: < 300ms
- Single database query
- Presigned URL generation (if completed)

---

#### TC-REPORT-PERF-003: Large Report Generation Time ✅

**Steps**:

1. Create report for 1-year dateRange with 100,000+ records
2. Measure processing time

**Expected Result**:

- Processing time: 5-20 minutes (depending on data size)
- Progress updates every 10-20%
- Estimated completion reasonably accurate

---

#### TC-REPORT-PERF-004: Download Streaming Performance ✅

**Steps**:

1. Download 50 MB PDF report
2. Measure throughput

**Expected Result**:

- Streaming starts immediately (no buffering)
- Throughput: depends on network, but S3 supports high bandwidth
- No memory issues (streaming, not loading entire file)

---

#### TC-REPORT-PERF-005: Concurrent Report Downloads ✅

**Steps**:

1. 20 users simultaneously download reports
2. Measure server load and response times

**Expected Result**:

- All downloads succeed
- Server remains stable
- S3 handles concurrent streaming efficiently

---

#### TC-REPORT-PERF-006: Queue Processing Throughput ✅

**Steps**:

1. Add 100 reports to queue
2. Measure processing rate (reports/minute)

**Expected Result**:

- Throughput: 5-15 reports/minute (depends on worker count and report complexity)
- Priority queue maintains URGENT > HIGH > NORMAL > LOW order

---

#### TC-REPORT-PERF-007: Database Aggregation Performance ✅

**Steps**:

1. Create MEDIA_ANALYTICS report with complex filters
2. Measure database aggregation time

**Expected Result**:

- Aggregation time: < 60 seconds for 100,000+ records
- Indexes utilized (dateRange, mediaType, predictedClass)
- Efficient MongoDB aggregation pipeline

---

## Conclusion

This test plan covers **70 comprehensive test cases** for the Analytics & Reporting feature:

- **Report Creation**: 15 tests
- **Report Retrieval**: 9 tests
- **Report Download**: 8 tests
- **Report Listing**: 8 tests
- **Report Cancellation**: 7 tests
- **Queue Statistics**: 3 tests
- **Edge Cases**: 10 tests
- **Security**: 10 tests
- **Performance**: 7 tests
