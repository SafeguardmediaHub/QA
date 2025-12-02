# Feature 04: Metadata Analysis & Tamper Detection

---

**Feature**: Metadata Analysis & Tamper Detection
**Priority**: P0 - Critical
**Test Coverage**: Extraction accuracy, tamper detection, scoring algorithms
**Dependencies**: Authentication (Feature 01), Media Processing (Feature 02)

---

## Table of Contents

1. [Overview](#overview)
2. [Test Scope](#test-scope)
3. [How It Works](#how-it-works)
4. [Metadata Extraction Details](#metadata-extraction-details)
5. [Tamper Detection Analysis](#tamper-detection-analysis)
6. [Scoring System](#scoring-system)
7. [Test Scenarios](#test-scenarios)
8. [Test Cases](#test-cases)
9. [Edge Cases](#edge-cases)
10. [Validation Tests](#validation-tests)
11. [Test Results](#test-results)
12. [Issues Found](#issues-found)
13. [Recommendations](#recommendations)

---

## Overview

### Purpose

Metadata Analysis & Tamper Detection is a core forensic feature that automatically extracts comprehensive metadata from uploaded media files and analyzes it for signs of manipulation, editing, or suspicious characteristics. This feature provides journalists, fact-checkers, and researchers with critical information to assess media authenticity.

### Key Features

- **Automatic Extraction** - Metadata extracted during upload processing (no manual trigger needed)
- **Comprehensive Coverage** - EXIF, IPTC, XMP, GPS, filesystem, and technical metadata
- **Tamper Detection** - Multi-factor analysis detecting editing software, suspicious values, temporal anomalies
- **Confidence Scoring** - Five separate scores: Integrity, Authenticity, Completeness, Temporal, Geolocation
- **Anomaly Detection** - Flags specific inconsistencies and suspicious patterns
- **Timezone Validation** - Cross-references GPS location with EXIF timezone data
- **Software Detection** - Identifies editing tools (Photoshop, GIMP, FFmpeg, etc.)

### Business Value

- Enables verification of media authenticity for investigative journalism
- Provides forensic evidence for identifying manipulated content
- Supports fact-checking workflows with objective metadata analysis
- Helps detect deepfakes and AI-generated content through metadata patterns
- Reduces manual forensic analysis time through automated detection

---

## Test Scope

### In Scope

**Metadata Extraction:**
- ✅ Image metadata (EXIF, IPTC, XMP)
- ✅ Video metadata (container, codec, tags)
- ✅ Audio metadata (codec, bitrate, tags)
- ✅ GPS data (coordinates, altitude, accuracy, timestamps)
- ✅ Temporal data (multiple datetime fields, timezone, subsecond precision)
- ✅ Filesystem metadata (creation, modification, access times)
- ✅ Technical specifications (camera settings, dimensions, codecs)

**Tamper Detection:**
- ✅ Editing software detection (20+ signatures)
- ✅ Suspicious timestamp detection (future dates, pre-digital era, defaults)
- ✅ GPS anomaly detection (Null Island, impossible coordinates, timezone mismatches)
- ✅ Temporal inconsistency detection (dates out of order, illogical sequences)
- ✅ Missing field detection (stripped metadata)
- ✅ Unrealistic value detection (ISO, focal length, technical parameters)

**Scoring & Analysis:**
- ✅ Integrity Score calculation
- ✅ Authenticity Score calculation
- ✅ Completeness Score calculation
- ✅ Temporal Consistency Score
- ✅ Geolocation Quality Score
- ✅ Overall Confidence calculation
- ✅ Anomaly reporting

### Out of Scope

- ❌ Manual metadata extraction triggering (fully automated)
- ❌ Metadata editing/modification (read-only)
- ❌ C2PA content authenticity (separate feature - Feature 05)
- ❌ Deepfake detection AI analysis (separate feature - Feature 08)
- ❌ Visual tampering detection (clone stamp, splicing)
- ❌ Audio deepfake detection (separate feature - Feature 08)
- ❌ Advanced forensic analysis (ELA, noise analysis, etc.)

---

## How It Works

### Processing Pipeline

Metadata extraction occurs automatically during media upload processing. The workflow is:

```
┌──────────────────────────────────────────────────────────────┐
│                    MEDIA UPLOAD PIPELINE                      │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
                   ┌──────────────────┐
                   │ Upload Confirmed │
                   └──────────────────┘
                              │
                              ▼
                   ┌──────────────────┐
                   │ Background Queue │
                   │  Job Triggered   │
                   └──────────────────┘
                              │
                              ▼
        ┌──────────────────────────────────────┐
        │   METADATA EXTRACTION (Step 1 of 8)  │
        ├──────────────────────────────────────┤
        │ • Download file from S3              │
        │ • Extract EXIF/IPTC/XMP (images)     │
        │ • Extract GPS data (20+ fields)      │
        │ • Extract temporal data              │
        │ • Extract video/audio metadata       │
        │ • Filesystem metadata                │
        │ • Timeout: 60 seconds                │
        └──────────────────────────────────────┘
                              │
                              ▼
        ┌──────────────────────────────────────┐
        │      TAMPER DETECTION ANALYSIS       │
        ├──────────────────────────────────────┤
        │ • Detect missing fields              │
        │ • Detect stripped metadata           │
        │ • Detect editing software            │
        │ • Detect suspicious timestamps       │
        │ • Detect GPS anomalies               │
        │ • Detect temporal inconsistencies    │
        │ • Validate timezone-GPS match        │
        │ • Calculate confidence scores        │
        └──────────────────────────────────────┘
                              │
                              ▼
        ┌──────────────────────────────────────┐
        │   STORE METADATA IN DATABASE         │
        ├──────────────────────────────────────┤
        │ • Save extracted metadata            │
        │ • Save analysis results              │
        │ • Save all scores                    │
        │ • Save anomaly list                  │
        │ • Update media status                │
        └──────────────────────────────────────┘
                              │
                              ▼
                   ┌──────────────────┐
                   │   Continue to    │
                   │ Thumbnail, Hash  │
                   │   Processing     │
                   └──────────────────┘
```

### No Dedicated Endpoints

**Important**: Metadata extraction does NOT have dedicated API endpoints for triggering. It happens automatically. To access metadata:

```http
GET /api/media/:id
```

The response includes the `metadata` object with full extraction results and `analysis` scores.

---

## Metadata Extraction Details

### Image Metadata (EXIF, IPTC, XMP)

**Libraries Used**: `exifr`, `sharp`

**Extracted Fields**:

| Category | Fields | Description |
|----------|--------|-------------|
| **Camera Info** | cameraMake, cameraModel, lens, lensModel, serialNumber | Device identification |
| **Camera Settings** | iso, aperture, exposureTime, focalLength, flash, whiteBalance | Capture parameters |
| **Timestamps** | dateTimeOriginal, dateTimeDigitized, createDate, modifyDate | Multiple datetime fields |
| **Timezone** | offsetTimeOriginal, offsetTimeDigitized, hasTimezone | Timezone information |
| **Precision** | subSecTimeOriginal, subSecTimeDigitized, hasSubseconds | Fractional seconds |
| **GPS** | 20+ fields (see GPS section) | Location data |
| **Image Properties** | width, height, orientation, colorSpace, compression | Technical specs |
| **Software** | software, imageUniqueID | Processing information |

### GPS Metadata (Comprehensive)

**Extracted Fields** (20+ fields):

| Field | Description | Forensic Value |
|-------|-------------|----------------|
| `latitude` | GPS latitude in decimal degrees | Location verification |
| `longitude` | GPS longitude in decimal degrees | Location verification |
| `altitude` | Altitude in meters | 3D location data |
| `altitudeRef` | Above/below sea level (0/1) | Altitude validation |
| `timestamp` | Combined GPS datetime | Independent timestamp source |
| `dateStamp` | GPS date (YYYY:MM:DD) | Date validation |
| `timeStamp` | GPS time (HH:MM:SS) | Time validation |
| `dop` | Dilution of Precision | Accuracy indicator (<5 good, <2 excellent) |
| `satellites` | Number of satellites used | Signal quality |
| `mapDatum` | Geodetic survey datum (e.g., WGS-84) | Coordinate system |
| `processingMethod` | GPS, NETWORK, etc. | Location method |
| `speed` | Speed of GPS receiver | Movement data |
| `speedRef` | Speed unit (K/M/N) | Unit reference |
| `track` | Direction of movement (0-359.99°) | Movement direction |
| `trackRef` | True/Magnetic (T/M) | Direction reference |
| `imgDirection` | Direction camera was pointing | Camera orientation |
| `imgDirectionRef` | True/Magnetic (T/M) | Orientation reference |
| `areaInformation` | Name of GPS area | Location description |
| `versionID` | GPS version | GPS standard version |

**Forensic Validation**:
- Cross-reference GPS timezone with EXIF timezone offset
- Detect "Null Island" (0,0) and impossible coordinates
- Validate coordinates are within Earth bounds (lat: ±90, lon: ±180)
- Check if GPS timestamp matches EXIF timestamp

### Video Metadata

**Library Used**: `ffprobe` (FFmpeg)

**Extracted Fields**:

| Field | Description |
|-------|-------------|
| `codec` | Video codec (H.264, VP9, etc.) |
| `container` | Container format (MP4, MKV, WebM) |
| `duration` | Video length in seconds |
| `width` | Video width in pixels |
| `height` | Video height in pixels |
| `fps` | Frames per second |
| `bitrate` | Video bitrate |
| `encoder` | Encoding software |
| `encoderOptions` | Encoding parameters |
| `rotation` | Video rotation metadata |
| `pixelFormat` | Pixel format (yuv420p, etc.) |
| `colorSpace` | Color space (bt709, etc.) |
| `audioStreams` | Array of audio stream info |
| `tags` | Container metadata tags |

**Video-Specific Tamper Detection**:
- Detect editing software (FFmpeg, HandBrake, Premiere, Final Cut Pro)
- Validate realistic FPS (1-300 fps)
- Check for suspiciously low bitrate (<100 kbps)
- Validate codec/container compatibility

### Audio Metadata

**Library Used**: `music-metadata`

**Extracted Fields**:

| Field | Description |
|-------|-------------|
| `codec` | Audio codec (MP3, AAC, FLAC, etc.) |
| `duration` | Audio length in seconds |
| `bitrate` | Audio bitrate |
| `sampleRate` | Sample rate (Hz) |
| `channels` | Number of audio channels |
| `encoder` | Encoding software |
| `tags` | ID3 or similar metadata tags |

**Audio-Specific Tamper Detection**:
- Detect audio editing software (Audacity, Logic Pro, etc.)
- Validate realistic sample rate (8000-192000 Hz)
- Check for unrealistic channel count (1-32)
- Detect suspicious tag patterns

### Filesystem Metadata

**Extracted Fields**:

| Field | Description | Forensic Value |
|-------|-------------|----------------|
| `created` | File creation time (birthtime) | Original file creation |
| `modified` | File modification time (mtime) | Last edit timestamp |
| `accessed` | File access time (atime) | Last access |
| `changed` | File status change time (ctime) | Metadata change |

**Forensic Validation**:
- Check if modified < created (filesystem anomaly)
- Compare EXIF dateTimeOriginal with file created
- Flag if file created >24 hours before photo allegedly taken
- Detect illogical date sequences

---

## Tamper Detection Analysis

### Editing Software Detection

**Detected Software** (20+ signatures):

**Image Editing**:
- Adobe Photoshop
- GIMP
- Lightroom
- Capture One
- Darktable
- RawTherapee
- Canon/Nikon/Sony manufacturer tools

**Video Editing**:
- Adobe Premiere
- Final Cut Pro
- DaVinci Resolve
- FFmpeg
- HandBrake

**Audio Editing**:
- Audacity
- Logic Pro

**Detection Method**:
- Checks `Software` EXIF tag
- Checks `encoder` field in video/audio tags
- Case-insensitive pattern matching

**Impact**: Presence of editing software lowers Authenticity Score by 0.2

### Suspicious Timestamp Detection

**Default Timestamps** (flagged as suspicious):

| Timestamp | Reason |
|-----------|--------|
| `1970:01:01 00:00:00` | Unix epoch (default value) |
| `2000:01:01 00:00:00` | Common default value |
| `1980:01:01 00:00:00` | FAT32 epoch |
| `1904:01:01 00:00:00` | Mac epoch |

**Pre-Digital Era Detection**:
- Any timestamp before `1990-01-01` flagged as suspicious
- Reason: Digital cameras didn't exist before 1990

**Future Date Detection**:
- Any timestamp in the future flagged as tampering
- Indicates clock manipulation or metadata editing

**Impact**: Each suspicious timestamp lowers Authenticity Score by 0.25

### GPS Anomaly Detection

**Invalid Coordinates**:

| Pattern | Reason |
|---------|--------|
| `(0, 0)` | "Null Island" - common default/error value |
| `(90, 180)` | Corner coordinates - likely error |
| `(-90, -180)` | Corner coordinates - likely error |
| `(0.0001, 0)` | Near Null Island - suspicious |
| `lat > ±90` | Impossible latitude |
| `lon > ±180` | Impossible longitude |

**Timezone-GPS Mismatch**:
- Calculate expected timezone from GPS coordinates (using `tz-lookup`)
- Compare with EXIF `offsetTimeOriginal` timezone
- Allow 1 hour tolerance for DST differences
- Flag if difference >60 minutes

**Impact**:
- Invalid coordinates: -0.15 penalty
- Timezone mismatch: -0.2 penalty on Geolocation Score

### Temporal Inconsistency Detection

**Detected Inconsistencies**:

1. **Illogical Sequence**:
   ```
   DateTimeOriginal > DateTimeDigitized
   ```
   Photo cannot be digitized before it was taken.

2. **File Created Before Photo Taken**:
   ```
   FileCreated < DateTimeOriginal - 1 hour
   ```
   File shouldn't exist before photo was captured.

3. **Modified Before Created**:
   ```
   FileModified < FileCreated
   ```
   Filesystem anomaly.

4. **Future Timestamp**:
   ```
   DateTimeOriginal > Current Time
   ```
   Likely tampering or clock manipulation.

5. **Large Date Span**:
   ```
   (Latest Date - Earliest Date) > 7 days
   ```
   Multiple dates spread over time suggests editing.

**Impact**: Each inconsistency lowers Temporal Score by 0.15

### Missing Field Detection

**Critical Fields by Media Type**:

**JPEG Images**:
- `dateTimeOriginal` - When photo was taken
- `cameraMake` or `cameraModel` - Device identification

**All Images**:
- Camera info (make/model)
- GPS data (if location-dependent content)
- ISO, aperture, exposureTime (for photos claiming authenticity)

**Videos**:
- Container metadata tags
- Creation timestamps

**Audio**:
- Metadata tags (ID3, etc.)

**Stripped Metadata Detection**:
```
If (JPEG AND no EXIF AND no camera info AND no GPS AND no ISO):
  Flag as "suspiciously minimal EXIF data"
```

**Impact**:
- Missing critical fields: -0.15 per field on Completeness Score
- Stripped metadata flag set to `true`

### Unrealistic Value Detection

**Invalid Technical Values**:

| Field | Valid Range | Detection |
|-------|-------------|-----------|
| ISO | 50 - 102,400 | Outside range = suspicious |
| Focal Length | 8mm - 2000mm | Outside range = unrealistic |
| Aperture | 0 | Zero = fake value |
| FPS (video) | 1 - 300 | Outside range = suspicious |
| Sample Rate (audio) | 8000 - 192,000 Hz | Outside range = unrealistic |
| Audio Channels | 1 - 32 | Outside range = error |
| Video Bitrate | >100 kbps | <100 = suspiciously low |

**Impact**: Each unrealistic value adds reason, lowers Authenticity Score

---

## Scoring System

### Five-Dimensional Scoring

Metadata analysis produces **five separate scores** (0.0 - 1.0 scale):

1. **Completeness Score** - How much metadata is present
2. **Integrity Score** - Technical consistency of metadata
3. **Authenticity Score** - Likelihood metadata is original/unedited
4. **Temporal Score** - Timestamp consistency and quality
5. **Geolocation Score** - GPS data quality and consistency

Plus an **Overall Confidence Score** combining all factors.

---

### 1. Completeness Score

**Purpose**: Measures how complete the metadata is for the media type.

**Calculation Method**:

```
For each media type, count present vs. expected fields:

Image Fields (11):
  ✓ cameraMake, cameraModel, dateTimeOriginal
  ✓ iso, aperture, exposureTime, focalLength
  ✓ width, height, gps, software

Video Fields (9):
  ✓ codec, container, duration, width, height
  ✓ fps, bitrate, encoder, tags

Audio Fields (6):
  ✓ codec, duration, bitrate, sampleRate
  ✓ channels, tags

Score = (Present Fields / Total Expected Fields)
```

**Scoring Example**:

```
JPEG image with:
  ✓ cameraMake: Canon
  ✓ cameraModel: EOS 5D
  ✓ dateTimeOriginal: 2024-11-15
  ✓ iso: 400
  ✓ aperture: f/2.8
  ✓ width: 1920
  ✓ height: 1080
  ✗ exposureTime: missing
  ✗ focalLength: missing
  ✗ gps: missing
  ✗ software: missing

Score = 7/11 = 0.64 (64%)
```

**Interpretation**:
- **0.9 - 1.0**: Excellent - Almost all metadata present
- **0.7 - 0.89**: Good - Most metadata present
- **0.5 - 0.69**: Fair - Some metadata missing
- **0.3 - 0.49**: Poor - Significant metadata missing
- **0.0 - 0.29**: Very Poor - Heavily stripped metadata

---

### 2. Integrity Score

**Purpose**: Measures technical consistency and absence of errors.

**Calculation Method**:

```
Start with perfect score (1.0)

Penalties:
- Zero values (ISO=0, aperture=0): -0.1 each
- Impossible coordinates: -0.15
- Large date span (>7 days): -0.1
- Very large date span (>30 days): -0.1
- Temporal inconsistencies: -0.05 each

Score = max(0, 1.0 + all penalties)
```

**Scoring Example**:

```
Image with:
  ✗ ISO = 0 (invalid): -0.1
  ✗ Aperture = 0 (invalid): -0.1
  ✓ GPS coords valid
  ✓ Dates within 1 hour

Score = 1.0 - 0.1 - 0.1 = 0.80 (80%)
```

**Interpretation**:
- **0.9 - 1.0**: Excellent - No technical errors
- **0.7 - 0.89**: Good - Minor issues
- **0.5 - 0.69**: Fair - Some inconsistencies
- **0.3 - 0.49**: Poor - Multiple errors
- **0.0 - 0.29**: Very Poor - Severe inconsistencies

---

### 3. Authenticity Score

**Purpose**: Measures likelihood metadata is original and unedited.

**Calculation Method**:

```
Start with perfect score (1.0)

Penalties:
- Editing software detected: -0.2
- Suspicious timestamp (default/future): -0.25
- Stripped metadata: -0.15
- Suspicious camera name: -0.1
- Contains editing software but no camera info: -0.15
- Suspicious values in technical metadata: -0.1 each

Bonuses:
+ Has camera serial number: +0.1
+ Has imageUniqueID: +0.05
+ Complete technical data: +0.1

Score = clamp(0, 1, 1.0 + penalties + bonuses)
```

**Scoring Example**:

```
Image with:
  ✗ Software: Adobe Photoshop CC: -0.2
  ✓ Camera: Canon EOS 5D
  ✓ DateTimeOriginal: valid
  ✓ Serial Number: 1234567890: +0.1
  ✓ Complete technical data: +0.1

Score = 1.0 - 0.2 + 0.1 + 0.1 = 1.0 (100% - but wait!)
Actually capped: Score = 0.90 (Photoshop penalty significant)
```

**Interpretation**:
- **0.9 - 1.0**: Excellent - Likely original, unedited
- **0.7 - 0.89**: Good - Minor editing or processing
- **0.5 - 0.69**: Fair - Likely edited, some concerns
- **0.3 - 0.49**: Poor - Strong editing indicators
- **0.0 - 0.29**: Very Poor - Heavily manipulated

---

### 4. Temporal Score

**Purpose**: Measures timestamp consistency and quality.

**Calculation Method**:

```
Start with perfect score (1.0)

Penalties:
- Temporal inconsistency: -0.15 each
- Date span >24 hours: -0.1
- Date span >7 days: -0.1 additional

Bonuses:
+ Has timezone information: +0.1
+ Has subsecond precision: +0.05
+ All dates consistent: +0.1

Score = clamp(0, 1, 1.0 + penalties + bonuses)
```

**Scoring Example**:

```
Image with:
  ✓ DateTimeOriginal: 2024-11-15 14:32:18
  ✓ DateTimeDigitized: 2024-11-15 14:32:19
  ✓ FileCreated: 2024-11-15 14:35:00
  ✓ Timezone: +01:00: +0.1
  ✓ Subseconds: .123: +0.05
  ✓ Dates consistent: +0.1
  ✓ Span: 3 minutes (no penalty)

Score = 1.0 + 0.1 + 0.05 + 0.1 = 1.25 → capped at 1.00 (100%)
```

**Interpretation**:
- **0.9 - 1.0**: Excellent - Consistent, high-quality timestamps
- **0.7 - 0.89**: Good - Minor inconsistencies
- **0.5 - 0.69**: Fair - Some temporal issues
- **0.3 - 0.49**: Poor - Significant inconsistencies
- **0.0 - 0.29**: Very Poor - Timestamps unreliable

---

### 5. Geolocation Score

**Purpose**: Measures GPS data quality and consistency.

**Calculation Method**:

```
If no GPS data: return 0.5 (neutral)

Start with perfect score (1.0)

Penalties:
- GPS anomaly detected: -0.2 each
- Timezone mismatch: -0.2
- Invalid coordinates: -0.15

Bonuses:
+ GPS timestamp present: +0.1
+ DOP (accuracy) data present: +0.05
  + DOP < 5 (good): +0.05
  + DOP < 2 (excellent): +0.05
+ Satellite count present: +0.05
+ GPS-timezone match: +0.15
+ Movement data (speed/direction): +0.05

Score = clamp(0, 1, 1.0 + penalties + bonuses)
```

**Scoring Example**:

```
Image with GPS:
  ✓ Latitude: 40.7128
  ✓ Longitude: -74.0060 (New York City)
  ✓ Timestamp: 2024-11-15 14:32:18: +0.1
  ✓ DOP: 1.8 (excellent): +0.05 + 0.05 + 0.05
  ✓ Satellites: 12: +0.05
  ✓ Timezone matches (UTC-05:00): +0.15
  ✗ No movement data

Score = 1.0 + 0.1 + 0.15 + 0.05 = 1.30 → capped at 1.00 (100%)
```

**Interpretation**:
- **0.9 - 1.0**: Excellent - High-quality, consistent GPS data
- **0.7 - 0.89**: Good - Reliable GPS data
- **0.5 - 0.69**: Fair - Some GPS issues or no GPS (neutral)
- **0.3 - 0.49**: Poor - GPS data unreliable
- **0.0 - 0.29**: Very Poor - GPS data suspicious/invalid

---

### Overall Confidence Score

**Purpose**: Combined measure of metadata trustworthiness.

**Calculation Method**:

```
Weighted average of:
  - Completeness Score × 0.2
  - Integrity Score × 0.3
  - Authenticity Score × 0.4
  - Temporal Score × 0.05
  - Geolocation Score × 0.05

Additional penalties:
  - Editing software detected: -0.2
  - Suspicious timestamp: -0.25
  - Missing critical data: -0.15
  - Inconsistent data: -0.1
  - Suspicious values: -0.15

Score = clamp(0, 1, weighted_average + penalties)
```

**Interpretation**:
- **0.85 - 1.0**: Very High Confidence - Metadata appears authentic
- **0.70 - 0.84**: High Confidence - Likely authentic with minor concerns
- **0.50 - 0.69**: Medium Confidence - Some concerns, further verification recommended
- **0.30 - 0.49**: Low Confidence - Significant concerns about authenticity
- **0.0 - 0.29**: Very Low Confidence - Strong indicators of tampering/manipulation

---

## Test Scenarios

### Scenario 1: Original Unedited Photo from Modern Camera

**User Story**: As a journalist, I receive a photo directly from a modern camera and want to verify it hasn't been edited.

**Expected Metadata**:
- Complete EXIF data with camera make/model
- Valid timestamps in logical order
- GPS data (if phone camera)
- No editing software tags
- High completeness, integrity, authenticity scores

**Test Coverage**:
- Upload original photo from Canon/Nikon/Sony/iPhone
- Verify all metadata extracted
- Verify high confidence scores (>0.85)
- No tampering flags

---

### Scenario 2: Photo Edited in Photoshop

**User Story**: As a fact-checker, I receive a photo that may have been edited and need to detect manipulation.

**Expected Metadata**:
- Software tag: "Adobe Photoshop"
- Modify dates later than original dates
- Possible missing original camera data
- Lower authenticity score
- Tampering flag: "Editing software detected"

**Test Coverage**:
- Upload Photoshop-edited image
- Verify software detection
- Verify authenticity score lowered
- Verify specific tampering reason listed

---

### Scenario 3: Photo with Stripped Metadata

**User Story**: As a forensic analyst, I receive a photo with suspiciously minimal metadata suggesting scrubbing.

**Expected Metadata**:
- Missing most EXIF fields
- No camera info, no GPS, no technical data
- Low completeness score (<0.3)
- "Stripped metadata" flag set to true
- Specific missing fields listed

**Test Coverage**:
- Upload image with minimal/stripped EXIF
- Verify detection of missing fields
- Verify low completeness score
- Verify stripped metadata flag

---

### Scenario 4: Photo with GPS-Timezone Mismatch

**User Story**: As an investigator, I receive a photo claiming to be from one location, but timestamps don't match.

**Expected Metadata**:
- GPS coordinates present
- EXIF timezone present
- Timezone doesn't match location (e.g., GPS in New York, timezone UTC+8)
- Geolocation anomaly flagged
- Lower geolocation score

**Test Coverage**:
- Upload image with GPS-timezone mismatch
- Verify anomaly detection
- Verify specific anomaly message
- Verify geolocation score reduced

---

### Scenario 5: Video with Suspicious Timestamps

**User Story**: As a content moderator, I need to identify videos with manipulated creation dates.

**Expected Metadata**:
- Video metadata extracted (codec, container, duration)
- Creation timestamp in future or pre-digital era
- Temporal anomaly flagged
- Specific reason: "Timestamp is in the future"
- Lower temporal and overall confidence scores

**Test Coverage**:
- Upload video with suspicious timestamp
- Verify timestamp detection
- Verify anomaly reporting
- Verify score penalties

---

## Test Cases

### Metadata Extraction Tests

#### TC-401: Extract Complete EXIF from Canon DSLR Photo

**Objective**: Verify comprehensive EXIF extraction from modern DSLR camera.

**Prerequisites**:
- Test image: Original unedited photo from Canon EOS 5D Mark IV
- Image should have: Camera info, GPS, full EXIF, not edited

**Test Steps**:
1. Authenticate as user
2. Upload Canon DSLR photo via presigned URL
3. Wait for processing to complete
4. Retrieve media details: `GET /api/media/{mediaId}`
5. Verify metadata extraction completeness

**Expected Result**:
- Status: 200 OK
- `metadata.image` includes:
  - `cameraMake`: "Canon"
  - `cameraModel`: "Canon EOS 5D Mark IV" (or similar)
  - `dateTimeOriginal`: valid date
  - `iso`: 100-6400 range
  - `aperture`: valid f-stop
  - `exposureTime`: valid shutter speed
  - `focalLength`: valid mm value
  - `width` & `height`: image dimensions
  - `software`: null or camera firmware
- `metadata.gps`: GPS coordinates if present
- `metadata.analysis.completenessScore`: >0.8
- `metadata.analysis.confidence`: >0.85

**Status**: ⏳ Pending

---

#### TC-402: Extract EXIF from iPhone Photo with GPS

**Objective**: Verify GPS extraction from iPhone photo with location services enabled.

**Prerequisites**:
- Test image: Photo taken with iPhone (iOS 14+) with location enabled

**Test Steps**:
1. Authenticate as user
2. Upload iPhone photo
3. Wait for processing
4. Retrieve media and examine GPS data

**Expected Result**:
- `metadata.gps` populated with:
  - `latitude` and `longitude`: valid coordinates
  - `altitude`: elevation data
  - `timestamp`: GPS timestamp
  - `processingMethod`: likely "NETWORK" or "GPS"
- `metadata.gps.latitude`: between -90 and 90
- `metadata.gps.longitude`: between -180 and 180
- `metadata.analysis.geolocationScore`: >0.7

**Status**: ⏳ Pending

---

#### TC-403: Extract Video Metadata from MP4

**Objective**: Verify video metadata extraction including codec, resolution, duration.

**Prerequisites**:
- Test video: MP4 file with H.264 codec, 1080p, 30fps

**Test Steps**:
1. Authenticate as user
2. Upload MP4 video
3. Wait for processing (may take 30-60 seconds for video)
4. Retrieve media metadata

**Expected Result**:
- `metadata.video` includes:
  - `codec`: "h264" or "H.264"
  - `container`: "mp4" or "mov"
  - `duration`: video length in seconds
  - `width`: 1920
  - `height`: 1080
  - `fps`: 30
  - `bitrate`: reasonable value (>1000 kbps)
- `metadata.general.duration`: matches video.duration
- `metadata.video.audioStreams`: array with audio stream info

**Status**: ⏳ Pending

---

#### TC-404: Extract Audio Metadata from MP3

**Objective**: Verify audio metadata extraction including codec, bitrate, tags.

**Prerequisites**:
- Test audio: MP3 file with ID3 tags

**Test Steps**:
1. Authenticate as user
2. Upload MP3 audio file
3. Wait for processing
4. Retrieve metadata

**Expected Result**:
- `metadata.audio` includes:
  - `codec`: "mp3"
  - `duration`: audio length in seconds
  - `bitrate`: bitrate in bps
  - `sampleRate`: 44100 or 48000 Hz typical
  - `channels`: 1 (mono) or 2 (stereo)
- `metadata.audio.tags`: object with ID3 tags (title, artist, etc.)
- Completeness score reflects tag presence

**Status**: ⏳ Pending

---

#### TC-405: Extract Filesystem Metadata

**Objective**: Verify filesystem timestamps are captured for all media types.

**Prerequisites**:
- Any media file (image, video, or audio)

**Test Steps**:
1. Authenticate as user
2. Upload media file
3. Wait for processing
4. Retrieve metadata and check filesystem section

**Expected Result**:
- `metadata.filesystem` includes:
  - `created`: File creation timestamp (Date object)
  - `modified`: File modification timestamp
  - `accessed`: File access timestamp
  - `changed`: File metadata change timestamp
- All timestamps are valid Date objects
- Timestamps are in logical order (created <= modified)

**Status**: ⏳ Pending

---

#### TC-406: Extract Temporal Data with Timezone

**Objective**: Verify comprehensive temporal metadata extraction including timezone information.

**Prerequisites**:
- Image with timezone-aware EXIF data (e.g., from recent iPhone/Android)

**Test Steps**:
1. Upload image with timezone data
2. Retrieve metadata
3. Verify temporal section completeness

**Expected Result**:
- `metadata.temporal` includes:
  - `dateTimeOriginal`: Photo taken timestamp
  - `dateTimeDigitized`: Digitization timestamp
  - `offsetTimeOriginal`: Timezone string (e.g., "+01:00")
  - `hasTimezone`: true
  - `allDates`: Array of all extracted dates
  - `dateRange`: Object with earliest, latest, spanSeconds
  - `datesConsistent`: true (if dates are consistent)
- Temporal score: >0.85 for consistent timestamps

**Status**: ⏳ Pending

---

### Tamper Detection Tests

#### TC-407: Detect Photoshop Editing Software

**Objective**: Verify detection of Adobe Photoshop editing.

**Prerequisites**:
- Test image: Photo edited and saved in Adobe Photoshop

**Test Steps**:
1. Upload Photoshop-edited image
2. Wait for processing
3. Retrieve metadata and check analysis

**Expected Result**:
- `metadata.image.software`: Contains "Adobe Photoshop" or "Photoshop"
- `metadata.analysis.reasons`: Includes "Editing software detected: Adobe Photoshop CC" (or similar)
- `metadata.analysis.possibleTampering`: true
- `metadata.analysis.authenticityScore`: Reduced by ~0.2 (compared to baseline)
- `metadata.analysis.confidence`: Lower than unedited equivalent

**Status**: ⏳ Pending

---

#### TC-408: Detect GIMP Editing Software

**Objective**: Verify detection of GIMP (open-source editor).

**Prerequisites**:
- Test image: Photo edited in GIMP

**Test Steps**:
1. Upload GIMP-edited image
2. Retrieve metadata analysis

**Expected Result**:
- Software tag contains "GIMP"
- Analysis reasons: "Editing software detected: GIMP"
- Tampering flag: true
- Authenticity score reduced

**Status**: ⏳ Pending

---

#### TC-409: Detect Video Editing (FFmpeg)

**Objective**: Verify detection of video re-encoding with FFmpeg.

**Prerequisites**:
- Test video: Re-encoded or edited with FFmpeg

**Test Steps**:
1. Upload FFmpeg-encoded video
2. Retrieve metadata

**Expected Result**:
- `metadata.video.encoder` or `metadata.video.tags.encoder`: Contains "FFmpeg"
- Analysis reasons: "Editing software detected: FFmpeg"
- Tampering flag: true
- Authenticity score impacted

**Status**: ⏳ Pending

---

#### TC-410: Detect Stripped Metadata (Minimal EXIF)

**Objective**: Verify detection of images with metadata stripped/scrubbed.

**Prerequisites**:
- Test image: JPEG with all EXIF removed (use tool like `exiftool -all= image.jpg`)

**Test Steps**:
1. Upload image with stripped EXIF
2. Retrieve metadata analysis

**Expected Result**:
- `metadata.image`: Most fields null/undefined
- `metadata.analysis.missingFields`: Array including ["datetimeOriginal", "cameraInfo", ...]
- `metadata.analysis.strippedMetadata`: true
- `metadata.analysis.reasons`: Includes "Suspiciously minimal EXIF data for JPEG image"
- `metadata.analysis.completenessScore`: <0.3
- `metadata.analysis.confidence`: Low (<0.5)

**Status**: ⏳ Pending

---

#### TC-411: Detect Future Timestamp

**Objective**: Verify detection of photos with timestamps in the future.

**Prerequisites**:
- Test image: Photo with `DateTimeOriginal` set to future date (e.g., 2030-01-01)

**Test Steps**:
1. Upload image with future timestamp
2. Retrieve metadata analysis

**Expected Result**:
- `metadata.temporal.dateTimeOriginal`: Future date
- `metadata.analysis.temporalAnomalies`: Includes "DateTimeOriginal is in the future (likely tampered)"
- `metadata.analysis.reasons`: Includes "Timestamp is in the future — possibly tampered."
- `metadata.analysis.possibleTampering`: true
- `metadata.analysis.temporalScore`: Significantly reduced
- `metadata.analysis.confidence`: Low

**Status**: ⏳ Pending

---

#### TC-412: Detect Pre-Digital Era Timestamp

**Objective**: Verify detection of timestamps before digital photography existed.

**Prerequisites**:
- Test image: Photo with DateTimeOriginal before 1990 (e.g., 1985-01-01)

**Test Steps**:
1. Upload image with pre-1990 timestamp
2. Retrieve analysis

**Expected Result**:
- `metadata.temporal.dateTimeOriginal`: Date before 1990-01-01
- Temporal anomalies: "DateTimeOriginal predates digital photography era (suspicious)"
- Analysis reasons: "Timestamp predates digital photography era"
- Tampering flag: true
- Temporal score and confidence reduced

**Status**: ⏳ Pending

---

#### TC-413: Detect Default Unix Epoch Timestamp

**Objective**: Verify detection of default timestamp (1970-01-01 00:00:00).

**Prerequisites**:
- Test image: Photo with DateTimeOriginal set to Unix epoch

**Test Steps**:
1. Upload image with 1970-01-01 timestamp
2. Retrieve analysis

**Expected Result**:
- Temporal anomalies: "DateTimeOriginal predates digital photography era"
- Analysis reasons: "Suspicious default timestamp detected"
- Tampering flag: true
- Confidence reduced

**Status**: ⏳ Pending

---

#### TC-414: Detect Null Island GPS Coordinates

**Objective**: Verify detection of invalid GPS coordinates (0, 0).

**Prerequisites**:
- Test image: Photo with GPS set to (0, 0) - "Null Island"

**Test Steps**:
1. Upload image with (0, 0) GPS coordinates
2. Retrieve analysis

**Expected Result**:
- `metadata.gps.latitude`: 0
- `metadata.gps.longitude`: 0
- `metadata.analysis.geolocationAnomalies`: May include GPS anomaly
- `metadata.analysis.reasons`: "Invalid or suspicious GPS coordinates"
- Geolocation score: Reduced
- Confidence impacted

**Status**: ⏳ Pending

---

#### TC-415: Detect Impossible GPS Coordinates

**Objective**: Verify detection of GPS coordinates outside Earth bounds.

**Prerequisites**:
- Test image: Photo with latitude=95 (impossible, max is 90)

**Test Steps**:
1. Upload image with invalid GPS (lat=95, lon=200)
2. Retrieve analysis

**Expected Result**:
- GPS coordinates present but invalid
- Analysis reasons: "Impossible GPS coordinates detected"
- Geolocation score: Very low
- Confidence significantly reduced

**Status**: ⏳ Pending

---

#### TC-416: Detect GPS-Timezone Mismatch

**Objective**: Verify detection when GPS location doesn't match EXIF timezone.

**Prerequisites**:
- Test image: Photo with GPS in New York (UTC-5) but EXIF timezone +8 (Asia)

**Test Steps**:
1. Upload image with GPS-timezone mismatch
2. Retrieve analysis

**Expected Result**:
- `metadata.gps`: Valid New York coordinates
- `metadata.temporal.offsetTimeOriginal`: "+08:00" or similar
- `metadata.analysis.geolocationAnomalies`: Includes timezone mismatch message
- Example: "GPS location timezone (America/New_York, -300min) doesn't match EXIF timezone offset (+08:00, +480min)"
- `metadata.analysis.gpsTimezoneMatch`: false
- Geolocation score: Reduced by 0.2
- Confidence impacted

**Status**: ⏳ Pending

---

#### TC-417: Detect Temporal Inconsistency (DateTimeOriginal > DateTimeDigitized)

**Objective**: Verify detection of illogical timestamp sequence.

**Prerequisites**:
- Test image: Photo with DateTimeOriginal after DateTimeDigitized

**Test Steps**:
1. Upload image with:
   - DateTimeOriginal: 2024-11-15 14:00:00
   - DateTimeDigitized: 2024-11-15 13:00:00
2. Retrieve analysis

**Expected Result**:
- `metadata.temporal.inconsistencies`: Includes "DateTimeOriginal is after DateTimeDigitized (illogical sequence)"
- `metadata.temporal.datesConsistent`: false
- Temporal score: Reduced by 0.15
- Confidence reduced

**Status**: ⏳ Pending

---

#### TC-418: Detect File Created Before Photo Taken

**Objective**: Verify detection of filesystem anomaly where file predates photo.

**Prerequisites**:
- Test image: Photo with EXIF DateTimeOriginal but file created timestamp is >24 hours earlier

**Test Steps**:
1. Upload image where:
   - DateTimeOriginal: 2024-11-15 14:00:00
   - File created: 2024-11-10 10:00:00 (5 days earlier)
2. Retrieve analysis

**Expected Result**:
- Temporal anomalies: "File created 120 hours before photo was taken (possible tampering)" (or similar)
- `metadata.analysis.fileMetadataMatch`: false
- Temporal score reduced
- Confidence impacted

**Status**: ⏳ Pending

---

#### TC-419: Detect Unrealistic ISO Value

**Objective**: Verify detection of impossible ISO values.

**Prerequisites**:
- Test image: Photo with ISO=500000 (unrealistic, max typical is 102400)

**Test Steps**:
1. Upload image with ISO=500000
2. Retrieve analysis

**Expected Result**:
- `metadata.image.iso`: 500000
- Analysis reasons: "Unrealistic ISO value detected"
- Integrity or authenticity score reduced
- Confidence impacted

**Status**: ⏳ Pending

---

#### TC-420: Detect Zero-Value Technical Metadata

**Objective**: Verify detection of suspicious zero values in technical fields.

**Prerequisites**:
- Test image: Photo with ISO=0, Aperture=0 (fake values)

**Test Steps**:
1. Upload image with zero values
2. Retrieve analysis

**Expected Result**:
- `metadata.image.iso`: 0
- `metadata.image.aperture`: 0
- Analysis reasons: "Suspicious zero values in technical metadata" or "Invalid zero values in technical metadata"
- Integrity score reduced
- Confidence impacted

**Status**: ⏳ Pending

---

### Scoring Validation Tests

#### TC-421: High Completeness Score for Complete Metadata

**Objective**: Verify completeness score calculation for image with all metadata.

**Prerequisites**:
- Test image: High-quality DSLR photo with complete EXIF, GPS, all fields populated

**Test Steps**:
1. Upload complete metadata image
2. Retrieve analysis scores

**Expected Result**:
- `metadata.analysis.completenessScore`: >= 0.9
- Most expected fields present
- Missing fields list is empty or minimal

**Status**: ⏳ Pending

---

#### TC-422: Low Completeness Score for Stripped Metadata

**Objective**: Verify low completeness score for minimal metadata.

**Prerequisites**:
- Test image: JPEG with stripped EXIF (only basic image data remains)

**Test Steps**:
1. Upload stripped metadata image
2. Retrieve analysis scores

**Expected Result**:
- `metadata.analysis.completenessScore`: <= 0.3
- Many missing fields listed
- Stripped metadata flag: true

**Status**: ⏳ Pending

---

#### TC-423: High Integrity Score for Consistent Metadata

**Objective**: Verify integrity score for metadata with no errors or inconsistencies.

**Prerequisites**:
- Test image: Original photo with valid timestamps, realistic values, no anomalies

**Test Steps**:
1. Upload consistent metadata image
2. Retrieve scores

**Expected Result**:
- `metadata.analysis.integrityScore`: >= 0.9
- No zero values
- No impossible coordinates
- Dates in logical order

**Status**: ⏳ Pending

---

#### TC-424: Low Integrity Score for Inconsistent Metadata

**Objective**: Verify low integrity score for metadata with errors.

**Prerequisites**:
- Test image: Photo with ISO=0, dates out of order, invalid GPS

**Test Steps**:
1. Upload inconsistent metadata image
2. Retrieve scores

**Expected Result**:
- `metadata.analysis.integrityScore`: <= 0.6
- Multiple integrity issues detected
- Reasons list populated with specific issues

**Status**: ⏳ Pending

---

#### TC-425: High Authenticity Score for Original Photo

**Objective**: Verify authenticity score for unedited photo.

**Prerequisites**:
- Test image: Original camera photo, no editing software, no suspicious values

**Test Steps**:
1. Upload original unedited photo
2. Retrieve scores

**Expected Result**:
- `metadata.analysis.authenticityScore`: >= 0.85
- No editing software detected
- No suspicious timestamps
- Complete camera info present

**Status**: ⏳ Pending

---

#### TC-426: Low Authenticity Score for Edited Photo

**Objective**: Verify reduced authenticity score for edited photo.

**Prerequisites**:
- Test image: Photo edited in Photoshop with software tag

**Test Steps**:
1. Upload Photoshop-edited image
2. Retrieve scores

**Expected Result**:
- `metadata.analysis.authenticityScore`: Reduced by ~0.2 from baseline
- Editing software penalty applied
- Authenticity score <= 0.75

**Status**: ⏳ Pending

---

#### TC-427: High Temporal Score for Consistent Timestamps

**Objective**: Verify temporal score for consistent, high-quality timestamps.

**Prerequisites**:
- Test image: Photo with timezone, subsecond precision, all dates consistent

**Test Steps**:
1. Upload image with complete temporal data
2. Retrieve scores

**Expected Result**:
- `metadata.analysis.temporalScore`: >= 0.9
- hasTimezone: true bonus (+0.1)
- hasSubseconds: true bonus (+0.05)
- datesConsistent: true bonus (+0.1)
- No inconsistencies

**Status**: ⏳ Pending

---

#### TC-428: Low Temporal Score for Inconsistent Timestamps

**Objective**: Verify reduced temporal score for timestamp issues.

**Prerequisites**:
- Test image: Photo with dates out of order, large date span, future timestamp

**Test Steps**:
1. Upload image with temporal issues
2. Retrieve scores

**Expected Result**:
- `metadata.analysis.temporalScore`: <= 0.5
- Multiple temporal inconsistencies detected
- Penalties applied for each issue

**Status**: ⏳ Pending

---

#### TC-429: High Geolocation Score for Quality GPS Data

**Objective**: Verify geolocation score for high-quality GPS metadata.

**Prerequisites**:
- Test image: Photo with GPS timestamp, low DOP, satellite count, timezone match

**Test Steps**:
1. Upload image with excellent GPS data
2. Retrieve scores

**Expected Result**:
- `metadata.analysis.geolocationScore`: >= 0.95
- GPS timestamp bonus: +0.1
- DOP bonuses: +0.15 (present + good + excellent)
- Satellites bonus: +0.05
- Timezone match bonus: +0.15
- No anomalies

**Status**: ⏳ Pending

---

#### TC-430: Low Geolocation Score for GPS Anomalies

**Objective**: Verify reduced geolocation score for GPS issues.

**Prerequisites**:
- Test image: Photo with GPS-timezone mismatch, invalid coordinates

**Test Steps**:
1. Upload image with GPS anomalies
2. Retrieve scores

**Expected Result**:
- `metadata.analysis.geolocationScore`: <= 0.5
- GPS anomalies detected
- Timezone mismatch penalty: -0.2
- Invalid coordinates penalty: -0.15

**Status**: ⏳ Pending

---

#### TC-431: Neutral Geolocation Score for No GPS

**Objective**: Verify neutral score when GPS data is absent.

**Prerequisites**:
- Test image: Photo without GPS metadata (e.g., from webcam or scanner)

**Test Steps**:
1. Upload image without GPS
2. Retrieve scores

**Expected Result**:
- `metadata.gps`: undefined or null
- `metadata.analysis.geolocationScore`: 0.5 (neutral)
- No GPS anomalies (can't analyze what's not there)

**Status**: ⏳ Pending

---

#### TC-432: Overall Confidence Score Calculation

**Objective**: Verify overall confidence score is weighted average of component scores.

**Prerequisites**:
- Test image with known scores:
  - Completeness: 0.8
  - Integrity: 0.9
  - Authenticity: 0.7
  - Temporal: 0.85
  - Geolocation: 0.75

**Test Steps**:
1. Upload test image
2. Verify overall confidence calculation

**Expected Result**:
- Overall confidence ≈ (0.8×0.2 + 0.9×0.3 + 0.7×0.4 + 0.85×0.05 + 0.75×0.05)
- = 0.16 + 0.27 + 0.28 + 0.0425 + 0.0375 = 0.79
- Plus any additional penalties for specific tampering indicators
- `metadata.analysis.confidence`: Approximately 0.75-0.85 range

**Status**: ⏳ Pending

---

## Edge Cases

### EC-11: Image with Partial EXIF Data

**Objective**: Verify handling of images with some but not all EXIF fields.

**Test Steps**:
1. Upload image with only camera make/model but no technical data (ISO, aperture, etc.)
2. Retrieve metadata

**Expected Result**:
- Extracts available fields
- Completeness score reflects partial data (0.4-0.6 range)
- Missing fields listed in analysis
- No errors from missing fields

**Status**: ⏳ Pending

---

### EC-12: Video Without Container Metadata Tags

**Objective**: Verify handling of video with no metadata tags in container.

**Test Steps**:
1. Upload video with no tags (raw encode without metadata)
2. Retrieve metadata

**Expected Result**:
- `metadata.video.tags`: Empty object `{}` or undefined
- Analysis reasons: "No metadata tags found in video container"
- Completeness score reduced
- No processing errors

**Status**: ⏳ Pending

---

### EC-13: Image with Multiple GPS Timestamps

**Objective**: Verify handling when GPS timestamp differs from EXIF timestamp.

**Test Steps**:
1. Upload image with both GPS timestamp and DateTimeOriginal
2. GPS timestamp is different from EXIF timestamp
3. Retrieve metadata

**Expected Result**:
- Both timestamps extracted
- `metadata.gps.timestamp`: GPS timestamp
- `metadata.temporal.dateTimeOriginal`: EXIF timestamp
- Both included in `allDates` array
- Temporal analysis checks consistency

**Status**: ⏳ Pending

---

### EC-14: Image with Subsecond Precision

**Objective**: Verify extraction of fractional seconds in timestamps.

**Test Steps**:
1. Upload image with SubSecTimeOriginal (e.g., "123" for 0.123 seconds)
2. Retrieve metadata

**Expected Result**:
- `metadata.temporal.subSecTimeOriginal`: "123"
- `metadata.temporal.hasSubseconds`: true
- Temporal score bonus applied (+0.05)

**Status**: ⏳ Pending

---

### EC-15: Image with Timezone Offset

**Objective**: Verify extraction of timezone offset from EXIF.

**Test Steps**:
1. Upload image with OffsetTimeOriginal (e.g., "+05:30" for India)
2. Retrieve metadata

**Expected Result**:
- `metadata.temporal.offsetTimeOriginal`: "+05:30"
- `metadata.temporal.hasTimezone`: true
- Temporal score bonus applied (+0.1)
- Used in GPS-timezone validation if GPS present

**Status**: ⏳ Pending

---

### EC-16: Video with Multiple Audio Streams

**Objective**: Verify extraction of all audio streams from video.

**Test Steps**:
1. Upload video with 2+ audio streams (e.g., multi-language)
2. Retrieve metadata

**Expected Result**:
- `metadata.video.audioStreams`: Array with multiple objects
- Each object includes: codec, channels, sampleRate
- All streams correctly identified

**Status**: ⏳ Pending

---

### EC-17: Image with Unusual Orientation (EXIF Rotation)

**Objective**: Verify extraction of EXIF orientation tag.

**Test Steps**:
1. Upload image with orientation=6 (90° CW rotation)
2. Retrieve metadata

**Expected Result**:
- `metadata.image.orientation`: 6
- Orientation correctly captured
- No errors from non-standard orientation

**Status**: ⏳ Pending

---

### EC-18: Audio File with No ID3 Tags

**Objective**: Verify handling of audio with missing metadata tags.

**Test Steps**:
1. Upload raw audio file (e.g., WAV) with no ID3/tags
2. Retrieve metadata

**Expected Result**:
- Technical data extracted: codec, duration, sampleRate, channels
- `metadata.audio.tags`: Empty or undefined
- Analysis may flag: "No metadata tags found in audio file"
- Completeness score reflects lack of tags

**Status**: ⏳ Pending

---

### EC-19: Image with Very Long Exposure Time

**Objective**: Verify handling of unusual but valid technical values.

**Test Steps**:
1. Upload image with exposureTime="30s" (30-second long exposure)
2. Retrieve metadata

**Expected Result**:
- `metadata.image.exposureTime`: "30" or "30s"
- Value extracted correctly
- No false tampering detection (long exposures are legitimate)
- No unrealistic value flag

**Status**: ⏳ Pending

---

### EC-20: Image with Camera Serial Number

**Objective**: Verify extraction of camera serial number and authenticity bonus.

**Test Steps**:
1. Upload image from camera with serial number in EXIF
2. Retrieve metadata

**Expected Result**:
- `metadata.image.serialNumber`: Camera serial
- Authenticity score bonus applied (+0.1)
- Higher confidence due to unique device identifier

**Status**: ⏳ Pending

---

## Validation Tests

### VAL-01: Metadata Extraction Timeout Handling

**Objective**: Verify graceful handling when metadata extraction times out.

**Test Steps**:
1. Upload extremely large image (>50MB) that may cause timeout
2. Wait for processing
3. Check result

**Expected Result**:
- Either: Metadata extracted successfully (if within 60s timeout)
- Or: Processing fails gracefully with timeout error
- No server crashes
- Media status updated appropriately

**Status**: ⏳ Pending

---

### VAL-02: Corrupted Image File Handling

**Objective**: Verify handling of corrupted/invalid image files.

**Test Steps**:
1. Upload corrupted JPEG file (truncated or invalid header)
2. Wait for processing
3. Check result

**Expected Result**:
- Processing fails gracefully
- Error message indicates corruption
- Media status: "failed" or "corrupted"
- No server crashes

**Status**: ⏳ Pending

---

### VAL-03: Image Format Without EXIF Support

**Objective**: Verify handling of image formats that don't support EXIF (e.g., BMP, GIF).

**Test Steps**:
1. Upload BMP or GIF file (formats without EXIF)
2. Retrieve metadata

**Expected Result**:
- Processing succeeds
- `metadata.image`: Minimal data (width, height)
- No EXIF-specific fields
- Completeness score reflects lack of EXIF
- No errors from absent EXIF

**Status**: ⏳ Pending

---

### VAL-04: Metadata Consistency Across Re-Upload

**Objective**: Verify same file uploaded twice produces identical metadata.

**Test Steps**:
1. Upload test image, note mediaId1
2. Upload same image again, note mediaId2
3. Retrieve metadata for both
4. Compare metadata objects

**Expected Result**:
- Metadata extraction is deterministic
- All fields match between uploads
- Scores are identical
- Analysis results consistent

**Status**: ⏳ Pending

---

### VAL-05: Metadata Persistence After Processing

**Objective**: Verify extracted metadata is permanently stored in database.

**Test Steps**:
1. Upload and process media
2. Retrieve metadata
3. Wait 24 hours (or restart server)
4. Retrieve same media again
5. Compare metadata

**Expected Result**:
- Metadata persists in database
- No re-extraction on subsequent retrievals
- Data remains identical
- Fast retrieval (<100ms)

**Status**: ⏳ Pending

---

## Test Results

### Summary

| Test Category | Total Tests | Passed | Failed | Blocked | Coverage |
|--------------|-------------|---------|---------|---------|----------|
| Metadata Extraction Tests | 6 | 0 | 0 | 0 | 0% |
| Tamper Detection Tests | 14 | 0 | 0 | 0 | 0% |
| Scoring Validation Tests | 12 | 0 | 0 | 0 | 0% |
| Edge Cases | 10 | 0 | 0 | 0 | 0% |
| Validation Tests | 5 | 0 | 0 | 0 | 0% |
| **TOTAL** | **47** | **0** | **0** | **0** | **0%** |

*This section will be updated as testing progresses.*

---

## Issues Found

No issues found yet. This section will be updated as testing progresses.

### Issue Template

```markdown
**Issue ID**: META-XXX
**Title**: Brief description
**Severity**: P0 Critical / P1 High / P2 Medium / P3 Low
**Category**: Extraction / Analysis / Scoring / Performance

**Description**: Detailed description

**Steps to Reproduce**:
1. Step one
2. Step two

**Expected Behavior**: What should happen

**Actual Behavior**: What actually happens

**Impact**: Effect on forensic analysis

**Metadata Sample**: Sample metadata showing issue

**Suggested Fix**: Potential solution

**Status**: Open / In Progress / Fixed / Closed
```

---

## Recommendations

### High Priority

1. **Add Metadata Re-Extraction Endpoint**: For admin/users to trigger re-extraction if initial processing failed or service improved

2. **Expose Individual Score Explanations**: Provide detailed breakdown of how each score was calculated in API response

3. **Add Metadata Comparison Tool**: Allow comparing metadata between two similar media files to detect inconsistencies

4. **Implement Metadata Export**: Allow downloading full metadata report as JSON or CSV for external analysis

5. **Add Custom Tampering Rules**: Allow admins to configure additional suspicious patterns specific to their use case

### Medium Priority

6. **Metadata History Tracking**: Track changes if media is re-processed, showing evolution of scores

7. **Benchmark Dataset**: Create reference dataset of known authentic/tampered media with expected scores

8. **Visual Metadata Report**: Generate visual timeline and map from temporal/GPS data

9. **Batch Metadata Analysis**: Endpoint to analyze metadata consistency across multiple related media files

10. **Metadata Quality Badges**: Assign badges (Gold/Silver/Bronze) based on metadata quality for quick assessment

### Low Priority

11. **Advanced EXIF Fields**: Extract additional niche EXIF fields (lens info, flash compensation, etc.)

12. **Metadata Anomaly Heatmap**: Visualize which metadata aspects are most suspicious

13. **Camera Database Integration**: Cross-reference camera make/model with known camera database for validation

14. **Metadata Search**: Allow searching media by specific metadata values (e.g., all Canon photos, all photos with GPS)

15. **Educational Mode**: Provide explanatory tooltips in UI about what each metadata field means for forensics

---

## Test Execution Notes

- All tests require media files with specific metadata characteristics
- Create comprehensive test media library before starting
- Use tools like `exiftool` to create test files with specific metadata values
- Document exact metadata values in test media for reproducibility
- Performance tests should measure extraction time for various file sizes
- Security tests should ensure no injection attacks through malicious EXIF data

---

**[← Previous: Media Management](./03-media-management.md)** | **[Next: C2PA Content Authenticity →](./05-c2pa-verification.md)**

---

*Last Updated: December 2, 2025*
*Test Status: Not Started*
*Total Test Cases: 47*
