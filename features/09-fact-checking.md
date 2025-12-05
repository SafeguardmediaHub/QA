# Feature 11: Fact Checking

## Overview

The **Fact Checking** feature provides AI-powered claim verification by automatically extracting verifiable claims from content (text, OCR, transcripts) and cross-referencing them against authoritative fact-checking databases. The system leverages the Google Fact Check API and IFCN-certified fact-checkers to provide credibility scores, verdicts, and evidence-based assessments of claims.

### What is Fact Checking?

Fact checking is the process of verifying the accuracy of claims made in content by:

1. **Extracting Verifiable Claims**: Identifying factual assertions that can be verified (statistical claims, attributions, causal statements, temporal claims)
2. **Retrieving Evidence**: Querying authoritative fact-checking databases (Google Fact Check API, IFCN-certified sources)
3. **Scoring Credibility**: Computing weighted credibility scores based on publisher authority, recency, and consensus

This feature helps users determine the truthfulness of content by providing:

- **Verdict Classification**: TRUE, MOSTLY_TRUE, MIXED, MOSTLY_FALSE, FALSE
- **Credibility Scores**: 0-100 scale with confidence levels (HIGH, MEDIUM, LOW)
- **Source Attribution**: Links to fact-checking sources with publisher credibility ratings
- **Consensus Analysis**: Agreement rates across multiple fact-checkers

### Why is Fact Checking Important?

- **Combat Misinformation**: Identify and flag false or misleading claims in content
- **Evidence-Based Verification**: Provide authoritative sources for claim verification
- **Credibility Assessment**: Quantify the reliability of information using weighted algorithms
- **Publisher Authority**: Prioritize IFCN-certified and reputable fact-checkers
- **Temporal Context**: Weight recent fact-checks more heavily than outdated ones
- **Consensus Building**: Aggregate multiple fact-check sources for robust verdicts

---

## Architecture

The Fact Checking system uses a **3-stage asynchronous pipeline** with background job processing:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         FACT CHECKING PIPELINE                          │
└─────────────────────────────────────────────────────────────────────────┘

Stage 1: CLAIM EXTRACTION
┌──────────────────────────────────────────────────────────────────────┐
│  User Submits Content                                                │
│  (text / OCR / transcript)                                           │
│         │                                                             │
│         ▼                                                             │
│  ┌──────────────────────────────┐                                    │
│  │ ClaimExtractionService       │                                    │
│  │ - Pattern Matching           │                                    │
│  │   • Statistical Claims       │                                    │
│  │   • Attribution Claims       │                                    │
│  │   • Causal Claims            │                                    │
│  │   • Temporal Claims          │                                    │
│  │ - Entity Extraction          │                                    │
│  │ - Text Normalization         │                                    │
│  └──────────────────────────────┘                                    │
│         │                                                             │
│         ▼                                                             │
│  Store Claims in MongoDB                                             │
│  (FactCheckClaim model)                                              │
└──────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
Stage 2: EVIDENCE RETRIEVAL
┌──────────────────────────────────────────────────────────────────────┐
│  For Each Claim:                                                     │
│         │                                                             │
│         ▼                                                             │
│  ┌──────────────────────────────┐                                    │
│  │ Check Cache First            │                                    │
│  │ (Normalized Text Hash)       │                                    │
│  └──────────────────────────────┘                                    │
│         │                                                             │
│         ├─── Cache Hit ──────────┐                                   │
│         │                         │                                   │
│         ▼                         │                                   │
│  ┌──────────────────────────────┐│                                   │
│  │ GoogleFactCheckService       ││                                   │
│  │ - Quota Management           ││                                   │
│  │ - Retry Logic (exponential)  ││                                   │
│  │ - API Request                ││                                   │
│  │ - Response Parsing           ││                                   │
│  └──────────────────────────────┘│                                   │
│         │                         │                                   │
│         ▼                         │                                   │
│  Store Verdicts in MongoDB       │                                   │
│  (FactCheckVerdict model) ───────┘                                   │
└──────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
Stage 3: CREDIBILITY SCORING
┌──────────────────────────────────────────────────────────────────────┐
│  For Each Claim with Verdicts:                                      │
│         │                                                             │
│         ▼                                                             │
│  ┌──────────────────────────────┐                                    │
│  │ CredibilityScoringService    │                                    │
│  │                              │                                    │
│  │ Compute Weighted Scores:     │                                    │
│  │ ┌────────────────────────┐   │                                    │
│  │ │ Publisher Credibility  │   │                                    │
│  │ │ - IFCN Certified: 1.5x │   │                                    │
│  │ │ - Reputable: 1.2x      │   │                                    │
│  │ │ - Unknown: 1.0x        │   │                                    │
│  │ └────────────────────────┘   │                                    │
│  │           ×                   │                                    │
│  │ ┌────────────────────────┐   │                                    │
│  │ │ Recency Multiplier     │   │                                    │
│  │ │ (fresher = higher)     │   │                                    │
│  │ └────────────────────────┘   │                                    │
│  │           ×                   │                                    │
│  │ ┌────────────────────────┐   │                                    │
│  │ │ Verdict Base Score     │   │                                    │
│  │ │ TRUE: 100              │   │                                    │
│  │ │ MOSTLY_TRUE: 75        │   │                                    │
│  │ │ MIXED: 50              │   │                                    │
│  │ │ MOSTLY_FALSE: 25       │   │                                    │
│  │ │ FALSE: 0               │   │                                    │
│  │ └────────────────────────┘   │                                    │
│  │           =                   │                                    │
│  │   Final Weighted Score        │                                    │
│  │                              │                                    │
│  │ Compute Consensus:           │                                    │
│  │ - Agreement Rate             │                                    │
│  │ - Confidence Level           │                                    │
│  │ - Final Verdict              │                                    │
│  └──────────────────────────────┘                                    │
│         │                                                             │
│         ▼                                                             │
│  Cache Score + Return Result                                         │
└──────────────────────────────────────────────────────────────────────┘
```

## How It Works

### Workflow: Content Analysis (Full Pipeline)

```
1. User Submits Content
   POST /api/fact-check/analyze
   {
     "content": "The Earth's population reached 8 billion in 2022.",
     "content_type": "text"
   }
   │
   ▼
2. System Creates Background Job
   Response (202 Accepted):
   {
     "job_id": "claim-extract-12345",
     "status": "processing",
     "estimated_completion_seconds": 60
   }
   │
   ▼
3. Stage 1: Claim Extraction (Background)
   - Pattern matching identifies claims
   - Extracts: "The Earth's population reached 8 billion in 2022"
   - Pattern: STATISTICAL (population numbers)
   - Confidence: 0.92
   - Stores claim in MongoDB
   │
   ▼
4. Stage 2: Evidence Retrieval (Background)
   - Normalizes claim text
   - Checks cache (miss)
   - Queries Google Fact Check API
   - Receives 3 verdicts from fact-checkers:
     • IFCN Source 1: "True" (reviewed 2 days ago)
     • IFCN Source 2: "True" (reviewed 5 days ago)
     • Reputable Source: "Mostly True" (reviewed 30 days ago)
   - Stores verdicts in MongoDB
   │
   ▼
5. Stage 3: Credibility Scoring (Background)
   - Computes weighted scores:
     • IFCN Source 1: 100 × 1.5 × 0.98 = 147
     • IFCN Source 2: 100 × 1.5 × 0.95 = 142.5
     • Reputable Source: 75 × 1.2 × 0.70 = 63
   - Average: (147 + 142.5 + 63) / 3 = 117.5 (normalized to 100)
   - Consensus: 100% (3/3 agree on TRUE/MOSTLY_TRUE)
   - Final Verdict: "Verified True"
   - Confidence: HIGH
   - Caches result
   │
   ▼
6. User Polls Job Status
   GET /api/fact-check/job/claim-extract-12345
   │
   ▼
7. System Returns Complete Results
   {
     "status": "completed",
     "claims": [
       {
         "claim_id": "claim-abc123",
         "text": "The Earth's population reached 8 billion in 2022",
         "credibility_score": 98,
         "reliability_index": 95,
         "verdict": "Verified True",
         "confidence": "High",
         "verdicts": [
           {
             "source": "FactCheck.org",
             "rating": "True",
             "review_url": "https://factcheck.org/...",
             "reviewed_at": "2025-12-01T10:00:00Z",
             "publisher_credibility": "ifcn_certified",
             "weighted_score": 147
           },
           // ... 2 more verdicts
         ]
       }
     ],
     "summary": {
       "total_claims": 1,
       "verified_true": 1,
       "verified_false": 0,
       "mixed": 0
     }
   }
```

### Workflow: Single Claim Verification (Fast Path)

```
1. User Submits Single Claim
   POST /api/fact-check/verify-claim
   {
     "claim_text": "COVID-19 vaccines contain microchips"
   }
   │
   ▼
2. System Checks Cache
   - Normalizes claim text
   - Cache hit ──────┐
   - Cache miss ─┐   │
                 │   │
                 ▼   │
3. Query Google API │
   - Returns 5 verdicts
   - All rated: "False"
                 │   │
                 ▼   │
4. Return Results ◄──┘
   {
     "verdicts_found": 5,
     "verdicts": [
       {
         "source": "Snopes",
         "rating": "False",
         "textual_rating": "Conspiracy Theory - False",
         "review_url": "https://snopes.com/...",
         "publisher_credibility": "ifcn_certified"
       },
       // ... 4 more verdicts
     ]
   }
```

### Claim Pattern Types

The system identifies 4 types of verifiable claims:

1. **Statistical Claims**

   - Examples: "70% of scientists agree...", "Population increased by 2 million..."
   - Pattern: Numbers, percentages, quantities
   - Confidence: HIGH (easily verifiable against data)

2. **Attribution Claims**

   - Examples: "President Biden said...", "According to WHO..."
   - Pattern: Quoted statements, attributed facts
   - Confidence: MEDIUM-HIGH (depends on source availability)

3. **Causal Claims**

   - Examples: "Vaccines cause autism", "Climate change leads to..."
   - Pattern: Cause-effect relationships
   - Confidence: MEDIUM (requires scientific consensus)

4. **Temporal Claims**
   - Examples: "Event occurred in 2020", "Before the pandemic..."
   - Pattern: Date references, time-based assertions
   - Confidence: HIGH (easily verifiable against timelines)

### Verdict Rating System

| Rating           | Score | Description                                       |
| ---------------- | ----- | ------------------------------------------------- |
| **TRUE**         | 100   | Claim is accurate and supported by evidence       |
| **MOSTLY_TRUE**  | 75    | Claim is largely accurate with minor inaccuracies |
| **MIXED**        | 50    | Claim contains both true and false elements       |
| **MOSTLY_FALSE** | 25    | Claim is largely inaccurate with some truth       |
| **FALSE**        | 0     | Claim is completely false or fabricated           |

### Publisher Credibility Multipliers

| Type               | Multiplier | Description                                   |
| ------------------ | ---------- | --------------------------------------------- |
| **IFCN_CERTIFIED** | 1.5×       | International Fact-Checking Network certified |
| **REPUTABLE**      | 1.2×       | Established fact-checking organization        |
| **UNKNOWN**        | 1.0×       | No verified credentials                       |

### Recency Multiplier Formula

```
recency_multiplier = 1.0 - (days_since_review / 365)
Capped at minimum: 0.5
Capped at maximum: 1.0

Examples:
- Reviewed 2 days ago: 1.0 - (2/365) = 0.995 ≈ 1.0
- Reviewed 30 days ago: 1.0 - (30/365) = 0.918
- Reviewed 180 days ago: 1.0 - (180/365) = 0.507
- Reviewed 365+ days ago: capped at 0.5
```

---

## Real-World Test Scenarios

### Scenario 1: News Article Fact-Checking ✅

**Context**: A journalist uploads an article claiming "Renewable energy now accounts for 40% of global electricity production."

**Test Flow**:

1. Upload article text via POST /analyze
2. System extracts statistical claim with HIGH confidence
3. Google Fact Check API returns 4 verdicts:
   - IEA (IFCN): "Mostly True" (38.5% actual figure)
   - PolitiFact: "Mostly True"
   - FactCheck.org: "Mostly True"
   - Local news: "True" (UNKNOWN credibility)
4. Credibility score: 78/100 (Mostly True, High Confidence)
5. Journalist receives detailed breakdown with source links

**Expected Result**: Claim verified as substantially accurate with minor numerical variance

---

### Scenario 2: Social Media Misinformation Detection ✅

**Context**: Content moderator checks viral post: "COVID-19 vaccines cause infertility in women."

**Test Flow**:

1. Submit claim via POST /verify-claim (fast path)
2. Cache hit (claim previously checked)
3. Returns 7 verdicts within 200ms:
   - All 7 sources rate as "False"
   - All from IFCN-certified fact-checkers
   - Reviews dated within last 6 months
4. Credibility score: 1/100 (Verified False, High Confidence)
5. Moderator flags content with authoritative sources

**Expected Result**: Immediate detection of false claim with high-confidence verdict

---

### Scenario 3: Multi-Claim Document Analysis ✅

**Context**: Research assistant analyzes political speech transcript containing 12 factual claims.

**Test Flow**:

1. Upload full transcript (3,200 characters) via POST /analyze
2. System extracts 12 claims:
   - 5 statistical claims (unemployment, GDP, immigration numbers)
   - 4 attribution claims (opponent statements, expert quotes)
   - 2 temporal claims (event dates)
   - 1 causal claim (policy impact)
3. Evidence retrieval finds verdicts for 9/12 claims
4. 3 claims return "no verdict" (too specific/local)
5. Results show:
   - 4 Verified True (credibility: 85-95)
   - 3 Mostly True (credibility: 70-80)
   - 2 Mixed (credibility: 45-55)
   - 3 Unknown (no verdicts)

**Expected Result**: Comprehensive breakdown showing 7/9 verified claims substantially accurate

---

### Scenario 4: Historical Claim with Outdated Verdicts ✅

**Context**: User fact-checks claim from 2020: "There is no effective vaccine for COVID-19."

**Test Flow**:

1. Submit claim via POST /verify-claim
2. System finds 6 verdicts:
   - 3 from March 2020: "True" (no vaccine existed then)
   - 3 from December 2020-2025: "False" (vaccines developed)
3. Recency multipliers heavily weight recent verdicts:
   - 2020 verdicts: 0.5× multiplier (capped minimum)
   - 2025 verdicts: 0.95-1.0× multiplier
4. Final verdict: "False" (current context)
5. Response notes temporal context and verdict evolution

**Expected Result**: System correctly identifies claim as false in current context despite historically accurate verdicts

---

### Scenario 5: Quota Exhaustion Handling ✅

**Context**: High-volume day with 10,000+ claims processed. Google API quota exhausted at 5:00 PM.

**Test Flow**:

1. User submits claim at 5:15 PM
2. System creates job successfully (202 Accepted)
3. Background worker checks quota before API call
4. Quota check fails (daily limit reached)
5. Job marked as FAILED with error: "QUOTA_EXCEEDED"
6. User polls job status, receives:
   - Status: "failed"
   - Error: "QUOTA_EXCEEDED"
   - Message: "Google Fact Check API daily quota exceeded. Resets at midnight UTC."
7. User retries next day, claim processes successfully via cache

**Expected Result**: Graceful quota handling with clear error messaging and retry guidance

---

### Scenario 6: Cache Performance ✅

**Context**: Popular false claim goes viral: "5G towers spread coronavirus."

**Test Flow**:

1. First user submits claim (11:00 AM)
   - Cache miss
   - API call retrieves 8 verdicts (all "False")
   - Processing time: 3.2 seconds
   - Result cached
2. Second user submits same claim (11:15 AM)
   - Claim normalized to same text
   - Cache hit
   - Processing time: 180ms
   - No API quota consumed
3. Over next 24 hours: 247 users check same claim
   - 246 cache hits
   - 1 API call total
   - Average response time: 200ms

**Expected Result**: Cache dramatically reduces API usage and improves response times

---

## Test Cases

### Content Analysis Tests (POST /analyze)

#### TC-FC-001: Valid Text Content Analysis ✅

**Preconditions**: User authenticated, valid text content
**Steps**:

1. POST /api/fact-check/analyze with:
   ```json
   {
     "content": "The World Health Organization declared COVID-19 a pandemic on March 11, 2020.",
     "content_type": "text"
   }
   ```
2. Verify 202 Accepted response
3. Extract job_id from response
4. Poll GET /job/:jobId every 3 seconds

**Expected Result**:

- Job created successfully with valid job_id
- Status transitions: processing → completed
- Claim extracted: "World Health Organization declared COVID-19 a pandemic on March 11, 2020"
- Pattern matched: TEMPORAL
- Verdicts found: ≥1
- Credibility score: 90-100 (verified true)
- Verdict: "Verified True"

**Response Time**: < 60 seconds total processing

---

#### TC-FC-002: OCR Content with Multiple Claims ✅

**Preconditions**: User authenticated, media item exists
**Steps**:

1. POST /api/fact-check/analyze with:
   ```json
   {
     "content": "Unemployment rate: 3.7%. Inflation at 2.3%. GDP growth: 4.1% annually. President announces tax cuts.",
     "content_type": "ocr",
     "media_id": "media-abc123"
   }
   ```
2. Poll for job completion

**Expected Result**:

- 4 claims extracted:
  - Claim 1: Unemployment rate 3.7% (STATISTICAL)
  - Claim 2: Inflation at 2.3% (STATISTICAL)
  - Claim 3: GDP growth 4.1% (STATISTICAL)
  - Claim 4: President announces tax cuts (ATTRIBUTION)
- Each claim has individual credibility score
- Claims linked to media_id
- Summary shows breakdown by verdict type

---

#### TC-FC-003: Content Below Minimum Length ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /api/fact-check/analyze with content shorter than minimum (e.g., 15 characters)
2. Observe response

**Expected Result**:

- 400 Bad Request
- Error message: "Content too short. Minimum {X} characters required"
- No job created

---

#### TC-FC-004: Content Exceeding Maximum Length ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /api/fact-check/analyze with content longer than maximum (e.g., 10,000 characters)
2. Observe response

**Expected Result**:

- 400 Bad Request
- Error message: "Content too long. Maximum {Y} characters allowed"
- No job created

---

#### TC-FC-005: Invalid Content Type ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /api/fact-check/analyze with invalid content_type: "video"

**Expected Result**:

- 400 Bad Request
- Validation error: "content_type must be one of: text, ocr, transcript, user_submission"

---

#### TC-FC-006: Transcript Analysis ✅

**Preconditions**: User authenticated, video transcript extracted
**Steps**:

1. POST /api/fact-check/analyze with content_type: "transcript"
2. Content: 500-word speech transcript with 8 factual claims

**Expected Result**:

- Job processes successfully
- 6-8 claims extracted (depending on verifiability)
- Pattern distribution: mix of STATISTICAL, ATTRIBUTION, TEMPORAL
- Processing time: 45-90 seconds (multiple API calls)

---

#### TC-FC-007: Content with No Verifiable Claims ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /api/fact-check/analyze with:
   ```json
   {
     "content": "I really enjoyed the movie. The cinematography was beautiful and the acting was superb. I would recommend it to my friends.",
     "content_type": "text"
   }
   ```

**Expected Result**:

- Job completes successfully
- 0 claims extracted
- Response: "No verifiable claims found in content"
- No API calls made to Google Fact Check

---

#### TC-FC-008: Unauthenticated Request ✅

**Preconditions**: No authentication token
**Steps**:

1. POST /api/fact-check/analyze without Authorization header

**Expected Result**:

- 401 Unauthorized
- Error message: "Authentication required"

---

#### TC-FC-009: Malformed Media ID ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /api/fact-check/analyze with media_id: "invalid-id-format"

**Expected Result**:

- 400 Bad Request
- Validation error: "media_id must be a valid MongoDB ObjectId"

---

#### TC-FC-010: Queue Service Unavailable ✅

**Preconditions**: Bull queue service down
**Steps**:

1. POST /api/fact-check/analyze with valid content

**Expected Result**:

- 500 Internal Server Error
- Error message: "Fact-checking service temporarily unavailable"
- Job not created

---

### Job Status Tests (GET /job/:jobId)

#### TC-FC-011: Poll Job During Processing ✅

**Preconditions**: Job created and processing
**Steps**:

1. Create job via POST /analyze
2. Immediately GET /job/:jobId (within 2 seconds)

**Expected Result**:

- 200 OK
- status: "processing"
- progress: 0-100 (percentage)
- stage: "claim_extraction" or "evidence_retrieval" or "credibility_scoring"

---

#### TC-FC-012: Retrieve Completed Job Results ✅

**Preconditions**: Job completed successfully
**Steps**:

1. GET /job/:jobId for completed job

**Expected Result**:

- 200 OK
- status: "completed"
- claims array populated with all extracted claims
- Each claim includes:
  - claim_id, text, credibility_score, reliability_index, verdict, confidence
  - verdicts array with all fact-check sources
- summary object with counts

---

#### TC-FC-013: Non-Existent Job ID ✅

**Preconditions**: User authenticated
**Steps**:

1. GET /job/nonexistent-job-id-12345

**Expected Result**:

- 404 Not Found
- Error message: "Job not found"

---

#### TC-FC-014: Failed Job Retrieval ✅

**Preconditions**: Job failed due to API error
**Steps**:

1. GET /job/:jobId for failed job

**Expected Result**:

- 200 OK (job exists, but failed)
- status: "failed"
- error field populated with error code (e.g., "QUOTA_EXCEEDED", "API_ERROR")
- error_details with human-readable message

---

#### TC-FC-015: Job with No Verdicts Found ✅

**Preconditions**: Job completed but no fact-checks exist for claims
**Steps**:

1. GET /job/:jobId where all claims returned 0 verdicts

**Expected Result**:

- 200 OK
- status: "completed"
- claims array populated
- Each claim has:
  - credibility_score: 0
  - verdict: "Unknown"
  - confidence: "Low"
  - verdicts: [] (empty array)

---

#### TC-FC-016: Unauthenticated Job Status Request ✅

**Preconditions**: No authentication
**Steps**:

1. GET /job/:jobId without token

**Expected Result**:

- 401 Unauthorized

---

#### TC-FC-017: Access Other User's Job ✅

**Preconditions**: User A authenticated, Job created by User B
**Steps**:

1. User A attempts GET /job/:jobIdCreatedByUserB

**Expected Result**:

- 403 Forbidden (if authorization check implemented)
- OR 404 Not Found (if job scoped to user)

---

#### TC-FC-018: Malformed Job ID ✅

**Preconditions**: User authenticated
**Steps**:

1. GET /job/123-invalid-format

**Expected Result**:

- 400 Bad Request
- Validation error: "Invalid job ID format"

---

### Single Claim Verification Tests (POST /verify-claim)

#### TC-FC-019: Verify True Claim with Multiple Sources ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /api/fact-check/verify-claim with:
   ```json
   { "claim_text": "The Earth orbits around the Sun" }
   ```

**Expected Result**:

- 200 OK (synchronous response)
- verdicts_found: ≥3
- All verdicts rating: "True"
- All sources IFCN-certified or reputable
- Response time: < 2 seconds

---

#### TC-FC-020: Verify False Claim (Conspiracy Theory) ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /verify-claim: "5G technology spreads coronavirus"

**Expected Result**:

- verdicts_found: ≥5
- All ratings: "False"
- textual_rating includes: "Debunked", "Conspiracy Theory", "No Evidence"
- High credibility multipliers (IFCN sources)

---

#### TC-FC-021: Claim with Mixed Verdicts ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /verify-claim with nuanced claim (e.g., "Coffee consumption reduces risk of Type 2 diabetes")

**Expected Result**:

- verdicts_found: ≥2
- Ratings vary: "True", "Mostly True", "Mixed"
- Different textual_ratings reflecting nuance
- Response includes all conflicting verdicts

---

#### TC-FC-022: Claim Not Found in Fact-Check Database ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /verify-claim with obscure/local claim: "Main Street Bakery won best croissant award"

**Expected Result**:

- 200 OK
- verdicts_found: 0
- message: "This claim has not been fact-checked by any known sources yet."
- verdicts: [] (empty array)

---

#### TC-FC-023: Claim Text Too Short ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /verify-claim with claim_text: "True" (4 characters)

**Expected Result**:

- 400 Bad Request
- Error: "Claim too short. Minimum {X} characters required"

---

#### TC-FC-024: Claim Text Too Long ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /verify-claim with claim_text exceeding maximum (e.g., 2000 characters)

**Expected Result**:

- 400 Bad Request
- Error: "Claim too long. Maximum {Y} characters allowed"

---

#### TC-FC-025: Cache Hit for Popular Claim ✅

**Preconditions**: Claim previously checked and cached
**Steps**:

1. POST /verify-claim with frequently checked claim
2. Monitor response time and API calls

**Expected Result**:

- 200 OK
- Response time: < 300ms (cache retrieval)
- No Google API call made (check logs/monitoring)
- Results identical to cached version

---

#### TC-FC-026: Rate Limiting on Verify Claim ✅

**Preconditions**: User has made many requests in short time
**Steps**:

1. Make 100 POST /verify-claim requests in 60 seconds

**Expected Result**:

- First N requests: 200 OK
- Subsequent requests: 429 Too Many Requests
- Error: "Rate limit exceeded. Try again in {X} seconds"
- Retry-After header present

---

#### TC-FC-027: Quota Exhausted on Verify Claim ✅

**Preconditions**: Google API daily quota exhausted
**Steps**:

1. POST /verify-claim with uncached claim

**Expected Result**:

- 503 Service Unavailable
- Error: "QUOTA_EXCEEDED"
- Message: "Google Fact Check API daily quota exceeded. Please try again after {reset_time}"

---

#### TC-FC-028: Special Characters in Claim Text ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /verify-claim with claim containing special characters: "Climate change is "real" & affects <everyone>"

**Expected Result**:

- 200 OK
- Claim normalized properly (quotes/brackets handled)
- Verdicts retrieved successfully

---

### Claim Retrieval Tests (GET /claims/:mediaId, GET /claim/:claimId)

#### TC-FC-029: Get All Claims for Media Item ✅

**Preconditions**: Media item has 5 fact-checked claims
**Steps**:

1. GET /api/fact-check/claims/media-abc123

**Expected Result**:

- 200 OK
- total_claims: 5
- claims array with all 5 claims
- Each claim includes: claim_id, original_text, status, credibility_score, verdict, verdicts_count
- Sorted by created_at (newest first)

---

#### TC-FC-030: Get Claims for Media with No Claims ✅

**Preconditions**: Media item exists but has no claims
**Steps**:

1. GET /claims/:mediaId

**Expected Result**:

- 200 OK
- total_claims: 0
- claims: [] (empty array)
- Message: "No claims found for this media item"

---

#### TC-FC-031: Get Claims for Non-Existent Media ✅

**Preconditions**: Media ID does not exist
**Steps**:

1. GET /claims/nonexistent-media-id

**Expected Result**:

- 404 Not Found
- Error: "Media item not found"

---

#### TC-FC-032: Get Single Claim Details ✅

**Preconditions**: Claim exists with 3 verdicts
**Steps**:

1. GET /api/fact-check/claim/claim-xyz789

**Expected Result**:

- 200 OK
- claim object with complete details
- verdicts array with all 3 verdicts (full details)
- credibility_score object with:
  - credibility_score, reliability_index, verdict, confidence
  - consensus (agreement_rate, verdict_distribution)
  - breakdown (weighted scores, publisher info, recency)

---

#### TC-FC-033: Get Claim with No Verdicts ✅

**Preconditions**: Claim marked as "no_verdict"
**Steps**:

1. GET /claim/:claimId

**Expected Result**:

- 200 OK
- claim object populated
- verdicts: [] (empty)
- credibility_score shows:
  - credibility_score: 0
  - verdict: "Unknown"
  - confidence: "Low"

---

#### TC-FC-034: Get Non-Existent Claim ✅

**Preconditions**: Claim ID does not exist
**Steps**:

1. GET /claim/fake-claim-id-999

**Expected Result**:

- 404 Not Found
- Error: "Claim not found"

---

#### TC-FC-035: Unauthorized Access to Claim ✅

**Preconditions**: Claim belongs to different user (if access control implemented)
**Steps**:

1. User A tries to GET claim created by User B

**Expected Result**:

- 403 Forbidden OR 404 Not Found (depending on implementation)

---

#### TC-FC-036: Claims Filtered by Status ✅

**Preconditions**: Media has claims in various statuses
**Steps**:

1. GET /claims/:mediaId?status=completed

**Expected Result** (if filtering implemented):

- Returns only claims with status: "completed"
- Excludes pending, processing, failed claims

---

### Admin Statistics Tests (GET /stats)

#### TC-FC-037: Admin Retrieves Global Stats ✅

**Preconditions**: Admin authenticated, system has historical data
**Steps**:

1. GET /api/fact-check/stats

**Expected Result**:

- 200 OK
- claims object with total, by_status, by_content_type, by_pattern
- verdicts object with total, by_rating, by_publisher_credibility
- api_usage with quota info, cache_hit_rate
- processing metrics (avg time, failure rate)
- top_sources list

---

#### TC-FC-038: Stats with Date Range Filter ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /stats?start_date=2025-11-01&end_date=2025-11-30

**Expected Result**:

- 200 OK
- period object shows specified date range
- All statistics filtered to November 2025 only
- Excludes data outside date range

---

#### TC-FC-039: Stats for Specific User ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /stats?user_id=user-123

**Expected Result**:

- 200 OK
- All statistics scoped to user-123's activity
- Claims, verdicts, processing stats for that user only

---

#### TC-FC-040: Non-Admin Access to Stats ✅

**Preconditions**: Regular user (non-admin) authenticated
**Steps**:

1. GET /stats

**Expected Result**:

- 403 Forbidden
- Error: "Administrator privileges required"

---

#### TC-FC-041: Stats with Invalid Date Format ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /stats?start_date=invalid-date-format

**Expected Result**:

- 400 Bad Request
- Validation error: "Invalid date format. Use ISO 8601 (YYYY-MM-DD)"

---

#### TC-FC-042: Stats Quota Information ✅

**Preconditions**: Admin authenticated, quota data available
**Steps**:

1. GET /stats
2. Examine api*usage.quota*\* fields

**Expected Result**:

- quota_remaining shows accurate count
- quota_limit shows daily/monthly limit
- quota_reset_date shows next reset time
- google_fact_check_calls matches actual API calls

---

### Credibility Scoring Tests

#### TC-FC-043: High Credibility Score (IFCN Consensus) ✅

**Preconditions**: Claim with 5 IFCN verdicts, all "True", all recent
**Steps**:

1. Retrieve claim details with verdicts
2. Examine credibility_score object

**Expected Result**:

- credibility_score: 95-100
- reliability_index: 95-100
- verdict: "Verified True"
- confidence: "High"
- consensus.agreement_rate: 1.0
- breakdown shows high credibility_multipliers (1.5) and recency_multipliers (>0.9)

---

#### TC-FC-044: Low Credibility Score (False Consensus) ✅

**Preconditions**: Claim with 4 verdicts, all "False", IFCN sources
**Steps**:

1. Examine credibility scoring

**Expected Result**:

- credibility_score: 0-5
- reliability_index: 0-10
- verdict: "Verified False"
- confidence: "High"
- consensus.agreement_rate: 1.0
- All weighted_scores: 0

---

#### TC-FC-045: Mixed Verdict Scenario ✅

**Preconditions**: Claim with conflicting verdicts (2 True, 2 False, 1 Mixed)
**Steps**:

1. Analyze credibility scoring

**Expected Result**:

- credibility_score: 40-60
- verdict: "Mixed" or "Conflicting Evidence"
- confidence: "Medium" or "Low"
- consensus.agreement_rate: < 0.5
- verdict_distribution shows spread

---

#### TC-FC-046: Publisher Credibility Weighting ✅

**Preconditions**: 2 verdicts: 1 IFCN (False), 1 Unknown (True)
**Steps**:

1. Compare weighted_scores

**Expected Result**:

- IFCN verdict has 1.5× multiplier
- Unknown verdict has 1.0× multiplier
- Final verdict: "False" (IFCN weighted more heavily)
- credibility_score reflects publisher authority

---

#### TC-FC-047: Recency Impact on Scoring ✅

**Preconditions**: 2 verdicts: 1 recent (5 days, True), 1 old (365 days, False)
**Steps**:

1. Compare recency_multipliers and weighted_scores

**Expected Result**:

- Recent verdict: recency_multiplier ≈ 0.986
- Old verdict: recency_multiplier = 0.5 (capped)
- Recent verdict weighted nearly 2× more heavily
- Final verdict favors recent fact-check

---

#### TC-FC-048: Single Verdict Confidence ✅

**Preconditions**: Claim with only 1 verdict
**Steps**:

1. Check confidence level

**Expected Result**:

- confidence: "Medium" or "Low" (not "High")
- credibility_score reflects single source limitation
- consensus shows low agreement_rate (1 verdict = limited consensus)

---

#### TC-FC-049: High Verdict Count Increases Confidence ✅

**Preconditions**: Claim with 10+ verdicts, strong consensus
**Steps**:

1. Compare to claim with 2 verdicts

**Expected Result**:

- 10-verdict claim: confidence "High", reliability_index higher
- 2-verdict claim: confidence "Medium", reliability_index lower
- More verdicts = higher confidence (given consensus)

---

#### TC-FC-050: Reliability Index Calculation ✅

**Preconditions**: Various claims with different verdict counts and agreement
**Steps**:

1. Compare reliability_index values

**Expected Result**:

- reliability_index considers:
  - Number of verdicts (more = higher)
  - Agreement rate (consensus = higher)
  - Publisher credibility (IFCN = higher)
  - Recency (fresher = higher)
- Range: 0-100
- Independent metric from credibility_score

---

### Cache Behavior Tests

#### TC-FC-051: Cache Miss on First Request ✅

**Preconditions**: New claim never checked before
**Steps**:

1. POST /verify-claim with novel claim
2. Monitor cache and API calls

**Expected Result**:

- Cache miss
- Google API call made
- Verdicts retrieved and stored
- Result cached with TTL
- Subsequent identical requests hit cache

---

#### TC-FC-052: Cache Hit on Subsequent Request ✅

**Preconditions**: Claim already cached
**Steps**:

1. POST /verify-claim with previously checked claim
2. Monitor response time

**Expected Result**:

- Cache hit
- No API call made
- Response time: < 300ms
- Results identical to cached version
- No quota consumed

---

#### TC-FC-053: Text Normalization for Cache Key ✅

**Preconditions**: User authenticated
**Steps**:

1. POST /verify-claim: "The Earth is ROUND!!!"
2. POST /verify-claim: "the earth is round"
3. POST /verify-claim: "The earth is round."

**Expected Result**:

- All 3 requests generate same cache key
- First request: cache miss, API call
- Second request: cache hit
- Third request: cache hit
- Normalization handles: capitalization, punctuation, whitespace

---

#### TC-FC-054: Cache Expiration ✅

**Preconditions**: Claim cached with 24-hour TTL, 25 hours elapsed
**Steps**:

1. POST /verify-claim with expired cache entry

**Expected Result**:

- Cache miss (expired)
- New API call made
- Fresh verdicts retrieved
- New cache entry created

---

#### TC-FC-055: Cache During Quota Exhaustion ✅

**Preconditions**: API quota exhausted, claim exists in cache
**Steps**:

1. POST /verify-claim with cached claim

**Expected Result**:

- 200 OK (cache hit)
- No API call attempted
- Results returned from cache
- No quota error

---

#### TC-FC-056: Cache Key Collision Prevention ✅

**Preconditions**: Two different claims that could hash similarly
**Steps**:

1. Submit two claims with similar text but different meaning
2. Verify separate cache entries

**Expected Result**:

- Different cache keys generated
- No collision
- Each claim cached independently

---

### API Quota Management Tests

#### TC-FC-057: Quota Tracking Accuracy ✅

**Preconditions**: Admin access, quota counter at known value
**Steps**:

1. Make 10 /verify-claim requests (all cache misses)
2. Check GET /stats for quota info

**Expected Result**:

- google_fact_check_calls increased by 10
- quota_remaining decreased by 10
- Accurate tracking

---

#### TC-FC-058: Quota Pre-Check Before API Call ✅

**Preconditions**: Quota at limit (0 remaining)
**Steps**:

1. POST /verify-claim with uncached claim

**Expected Result**:

- System checks quota before making API call
- Quota exhausted detected
- No API call made (prevents over-limit charges)
- 503 Service Unavailable returned

---

#### TC-FC-059: Quota Reset Timing ✅

**Preconditions**: Quota exhausted yesterday, new day started
**Steps**:

1. Check quota status via GET /stats
2. POST /verify-claim

**Expected Result**:

- quota_remaining reset to full limit
- quota_reset_date updated to next reset
- API calls work normally

---

#### TC-FC-060: Quota Exhaustion During Job Processing ✅

**Preconditions**: Job with 5 claims, quota has space for 3 API calls
**Steps**:

1. Process job
2. Observe behavior when quota exhausted mid-job

**Expected Result**:

- First 3 claims processed successfully
- 4th claim fails with QUOTA_EXCEEDED
- Job marked as partially completed
- Claims 4-5 marked as failed with quota error
- User notified of partial results

---

### Edge Cases and Error Handling

#### TC-FC-061: Empty Claim Text After Normalization ✅

**Preconditions**: User submits claim with only special characters: "!@#$%^&\*()"
**Steps**:

1. POST /verify-claim

**Expected Result**:

- 400 Bad Request
- Error: "Claim text contains no verifiable content after normalization"

---

#### TC-FC-062: Very Long Claim with Multiple Assertions ✅

**Preconditions**: Claim contains 10+ distinct factual assertions in one sentence
**Steps**:

1. POST /verify-claim with complex multi-claim sentence

**Expected Result**:

- System may extract first/primary claim OR
- Error suggesting use of /analyze endpoint for multi-claim content

---

#### TC-FC-063: Non-English Claim ✅

**Preconditions**: Claim in Spanish: "La Tierra es plana"
**Steps**:

1. POST /verify-claim

**Expected Result**:

- System attempts Google API query
- If Google supports language: verdicts returned (if available)
- If unsupported: verdicts_found: 0 (no results)
- languageCode parameter sent as configured (default: "en")

---

#### TC-FC-064: Malformed JSON in Request ✅

**Preconditions**: User sends invalid JSON
**Steps**:

1. POST /analyze with malformed JSON body

**Expected Result**:

- 400 Bad Request
- Error: "Invalid JSON format"

---

#### TC-FC-065: Missing Required Fields ✅

**Preconditions**: User omits content_type
**Steps**:

1. POST /analyze with only "content" field

**Expected Result**:

- 400 Bad Request
- Validation error: "content_type is required"

---

#### TC-FC-066: Google API Returns Malformed Response ✅

**Preconditions**: Google API returns unexpected structure (mock/test)
**Steps**:

1. Trigger API call with mocked malformed response

**Expected Result**:

- System logs error
- Retries with exponential backoff (if retryable)
- If persistent: Job fails gracefully
- Error: "External fact-check service returned invalid data"

---

#### TC-FC-067: Google API 500 Error with Retry ✅

**Preconditions**: Google API temporarily unavailable (500 response)
**Steps**:

1. POST /verify-claim
2. Monitor retry behavior

**Expected Result**:

- First attempt: 500 error
- System retries after 1 second
- Second attempt: 500 error
- System retries after 2 seconds
- Third attempt: 500 error
- System retries after 4 seconds
- Fourth attempt: Success OR final failure
- Max 3 retries (4 total attempts)

---

#### TC-FC-068: Network Timeout During API Call ✅

**Preconditions**: Google API call times out
**Steps**:

1. Simulate network timeout

**Expected Result**:

- Timeout detected
- Retry logic triggered (timeout is retryable)
- Exponential backoff applied
- After max retries: Job fails with "NETWORK_TIMEOUT" error

---

#### TC-FC-069: Concurrent Requests for Same Claim ✅

**Preconditions**: 10 users simultaneously submit identical claim (cache miss)
**Steps**:

1. Send 10 simultaneous POST /verify-claim requests

**Expected Result**:

- Ideal: Only 1 API call made (deduplication/locking)
- Acceptable: Multiple API calls but all return same results
- All 10 requests receive 200 OK
- Results cached
- No race conditions or inconsistencies

---

#### TC-FC-070: Database Connection Failure During Job ✅

**Preconditions**: MongoDB connection lost mid-job
**Steps**:

1. Process job
2. Simulate DB connection failure after claim extraction

**Expected Result**:

- Job fails gracefully
- Error logged with details
- Job marked as FAILED
- User receives error: "Database error - please retry"
- No data corruption

---

## Security Tests

### Authentication & Authorization

#### TC-FC-SEC-001: JWT Token Validation ✅

**Steps**:

1. Attempt all endpoints with expired JWT token

**Expected Result**:

- 401 Unauthorized for all protected endpoints
- Error: "Token expired"

---

#### TC-FC-SEC-002: Invalid JWT Signature ✅

**Steps**:

1. Modify JWT signature and send requests

**Expected Result**:

- 401 Unauthorized
- Error: "Invalid token signature"

---

#### TC-FC-SEC-003: Admin Endpoint Access Control ✅

**Steps**:

1. Regular user (non-admin) attempts GET /stats

**Expected Result**:

- 403 Forbidden
- Error: "Administrator privileges required"

---

#### TC-FC-SEC-004: SQL Injection in Claim Text ✅

**Steps**:

1. POST /verify-claim with: "'; DROP TABLE claims; --"

**Expected Result**:

- Claim treated as literal text (no SQL execution)
- Normalized and queried safely
- No database impact

---

#### TC-FC-SEC-005: NoSQL Injection Attempt ✅

**Steps**:

1. POST /analyze with claim_id parameter: {"$ne": null}

**Expected Result**:

- Input validation rejects non-string values
- 400 Bad Request
- No database query executed

---

#### TC-FC-SEC-006: XSS in Claim Text ✅

**Steps**:

1. POST /verify-claim with: "<script>alert('XSS')</script>"

**Expected Result**:

- Claim stored as plain text
- Response properly escaped/sanitized
- No script execution in client
- Verdicts retrieved normally (if exist)

---

#### TC-FC-SEC-007: Path Traversal in Job ID ✅

**Steps**:

1. GET /job/../../../etc/passwd

**Expected Result**:

- Route validation rejects path traversal
- 400 Bad Request OR 404 Not Found

---

#### TC-FC-SEC-008: Excessive Claim Submissions (DoS Prevention) ✅

**Steps**:

1. Submit 1000 /analyze requests in 10 seconds

**Expected Result**:

- Rate limiting activated
- Requests throttled after threshold
- 429 Too Many Requests
- User/IP temporarily blocked

---

#### TC-FC-SEC-009: Oversized Request Payload ✅

**Steps**:

1. POST /analyze with 50MB request body

**Expected Result**:

- Request rejected before processing
- 413 Payload Too Large
- Server remains stable

---

#### TC-FC-SEC-010: Header Injection Attempt ✅

**Steps**:

1. Send request with malicious headers (CRLF injection)

**Expected Result**:

- Headers sanitized/rejected
- No header injection successful
- Request processed safely or rejected

---

## Performance Tests

#### TC-FC-PERF-001: Single Claim Processing Time ✅

**Preconditions**: Cache miss, normal API response time
**Steps**:

1. POST /verify-claim with uncached claim
2. Measure end-to-end response time

**Expected Result**:

- Response time: < 3 seconds (95th percentile)
- Average: 1-2 seconds

---

#### TC-FC-PERF-002: Multi-Claim Job Processing Time ✅

**Preconditions**: Content with 10 extractable claims
**Steps**:

1. POST /analyze
2. Measure time from job creation to completion

**Expected Result**:

- Processing time: < 90 seconds
- Average: 45-60 seconds
- Time scales linearly with claim count

---

#### TC-FC-PERF-003: Cache Response Time ✅

**Preconditions**: Claim in cache
**Steps**:

1. POST /verify-claim (cache hit)
2. Measure response time

**Expected Result**:

- Response time: < 300ms (95th percentile)
- Average: 150-200ms
- 10× faster than cache miss

---

#### TC-FC-PERF-004: Concurrent Request Handling ✅

**Preconditions**: Load testing environment
**Steps**:

1. Send 100 concurrent /verify-claim requests (various claims)
2. Measure throughput and response times

**Expected Result**:

- All requests complete successfully
- Average response time: < 5 seconds
- No timeouts
- Server CPU/memory within limits

---

#### TC-FC-PERF-005: Database Query Performance ✅

**Preconditions**: 10,000+ claims in database
**Steps**:

1. GET /claims/:mediaId for media with 50 claims
2. Measure query time

**Expected Result**:

- Response time: < 500ms
- Database indexed on media_id and user_id
- Efficient pagination (if implemented)

---

#### TC-FC-PERF-006: Large Content Processing ✅

**Preconditions**: Maximum allowed content size (e.g., 5000 characters)
**Steps**:

1. POST /analyze with 5000-character content
2. Measure processing time

**Expected Result**:

- Processing completes successfully
- Time: < 120 seconds
- Memory usage reasonable (< 500MB per job)

---

#### TC-FC-PERF-007: Cache Hit Rate Over Time ✅

**Preconditions**: 24-hour production simulation
**Steps**:

1. Simulate realistic traffic pattern
2. Measure cache hit rate via GET /stats

**Expected Result**:

- cache_hit_rate: > 20% (depends on content diversity)
- Frequently fact-checked claims show high hit rate
- Quota usage significantly reduced

---

#### TC-FC-PERF-008: Job Queue Throughput ✅

**Preconditions**: 1000 jobs queued
**Steps**:

1. Monitor job completion rate
2. Measure queue processing time

**Expected Result**:

- Jobs processed: 10-20 per minute (per worker)
- Queue drains within reasonable time
- No job starvation
- Failed jobs < 2%

---

## Conclusion

This test plan covers **70 comprehensive test cases** across all aspects of the Fact Checking feature:

- **Content Analysis**: 10 tests
- **Job Status**: 8 tests
- **Single Claim Verification**: 10 tests
- **Claim Retrieval**: 8 tests
- **Admin Statistics**: 6 tests
- **Credibility Scoring**: 8 tests
- **Cache Behavior**: 6 tests
- **Quota Management**: 4 tests
- **Edge Cases**: 10 tests
- **Security**: 10 tests
- **Performance**: 8 tests

---

**Next Feature**: [Feature 10: Fact Checking →](./09-fact-checking.md)

**Previous Feature**: [← Feature 08: Reverse Lookup ](./08-reverse-lookup.md)
