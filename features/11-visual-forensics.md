**Visual Forensics Backend - Technical Dossier**

**1\. Introduction**

Visual Forensics is an image authenticity verification service designed to detect manipulation, tampering, and AI-generated content through comprehensive forensic analysis. The backend is a Python/Flask service built with OpenCV, PIL, NumPy, scikit-image, and SciPy for advanced image analysis and forgery detection.

The primary purpose is to help organizations verify image authenticity for journalism, legal evidence, content moderation, deepfake detection, and digital asset verification. The solution focuses on multi-method detection: Error Level Analysis (ELA), noise pattern analysis, copy-move detection, JPEG compression artifacts, AI generation detection, and metadata extraction.

**Deployment environments:**

- **Production**: Flask service (port 4000) with 100MB upload limit, comprehensive forensic analysis
- **Staging**: Parity environment with verbose logging
- **CI/CD**: Containerized deployment with OpenCV and scientific computing libraries

**2\. System Architecture**

The backend is a Flask application structured around forensic analysis pipelines: each analysis method runs independently with error handling, results are aggregated into a comprehensive report, and visualizations (heatmaps) highlight suspicious regions.

**Key components:**

- **Configuration**: 100MB limit, /tmp storage, image format whitelist (JPG, PNG, BMP, TIFF, WebP)
- **Routes**: /analyze (single), /batch (multiple), /compare (similarity), /metadata (EXIF only), /test (web UI)
- **Core Analysis Functions**: 8 independent forensic methods + metadata extraction
- **Visualization**: Heatmap generation for each detection method + combined heatmap
- **Summary Generation**: User-friendly non-technical summary with risk assessment

**Request lifecycle:**

- POST request with image file
- UUID-based file storage
- Parallel execution of forensic analyses
- Heatmap generation for visual evidence
- Aggregation into JSON report
- User-friendly summary generation
- ZIP packaging (JSON + heatmaps)
- Cleanup temporary files

**3\. Core Modules**

**Routes**

**POST /analyze**

- Single image forensic analysis
- Parameters: image (required), format (json/zip)
- Returns: ZIP with forensic_report.json + combined_heatmap.png

**POST /batch**

- Multiple image analysis
- Parameters: images (multiple files)
- Returns: ZIP with individual reports per image + batch_summary.json

**POST /compare**

- Image similarity comparison (SSIM)
- Parameters: image1, image2
- Returns: JSON with similarity score + hash comparison

**POST /metadata**

- EXIF/metadata extraction only
- Parameters: image
- Returns: JSON with file metadata, EXIF data, GPS info, hashes

**GET /test**

- Browser-based testing interface
- Real-time analysis with visualization display

**4\. Forensic Analysis Methods**

**4.1 Metadata Extraction**

**Function**: extract_metadata(image_path)

Extracts comprehensive metadata from image files:

**File Metadata:**

- Filename, format, dimensions (WxH)
- File size (bytes and MB)
- Creation/modification timestamps
- Color mode (RGB, RGBA, Grayscale)

**EXIF Data:**

- Camera make/model
- Lens information
- Exposure settings (ISO, aperture, shutter speed)
- GPS coordinates (if present)
- Software used (editing software detection)
- Date/time original

**Verification Hashes:**

- MD5 hash
- SHA-256 hash

**Use Case**: Detect metadata stripping (sign of editing), verify GPS location, identify camera source

**4.2 Error Level Analysis (ELA)**

**Function**: error_level_analysis(image_path, output_path, quality=95)

Detects manipulated regions by analyzing JPEG compression inconsistencies.

**How It Works:**

- Load original image
- Re-save at specified JPEG quality (95%)
- Calculate pixel-by-pixel difference between original and re-compressed
- Enhance differences (multiply by 10) for visibility
- Generate heatmap (red/yellow = high manipulation, blue/green = low)
- Calculate manipulation score (mean difference)

**Output:**

- ela_heatmap.png: Visual heatmap
- ela_score: Numeric score (0-100)
- interpretation: Low (&lt;8), Medium (8-15), High (&gt;15)

**Detection Strength:**

- Excellent for: Photoshopped images, spliced regions, clone stamp edits
- Limitations: Less effective on PNG/uncompressed formats

**4.3 Noise Analysis**

**Function**: noise_analysis(image_path, output_path)

Detects inconsistent noise patterns across the image (sign of splicing/compositing).

**How It Works:**

- Convert to grayscale
- Apply Gaussian blur to extract base image
- Subtract blurred from original to isolate noise
- Calculate local noise variance (15x15 pixel windows)
- Normalize and create heatmap
- Calculate inconsistency score (standard deviation of variance)

**Output:**

- noise_heatmap.png: Noise inconsistency visualization
- noise_inconsistency_score: Numeric score
- interpretation: Low (&lt;25), Medium (25-50), High (&gt;50)

**Detection Strength:**

- Excellent for: Images from multiple cameras/sources, pasted objects
- Limitation: Natural images may have varying noise (sky vs. ground)

**4.4 Copy-Move Detection**

**Function**: copy_move_detection(image_path, output_path, block_size=32)

Detects duplicated regions (cloned/copied areas within the same image).

**How It Works:**

- Divide image into 32x32 pixel blocks
- Extract DCT (Discrete Cosine Transform) features from each block
- Compare all blocks pairwise using Euclidean distance
- Flag blocks with high feature similarity but spatial separation
- Mark suspicious regions on heatmap
- Calculate clone score (percentage of image affected)

**Optimization:**

- Resize large images to 800px max dimension for speed
- Limit comparisons to 1000 blocks maximum
- Step size = block size (avoid overlapping blocks)

**Output:**

- clone_detection.png: Heatmap overlay on original
- clone_score: Percentage of image with duplication (0-100)
- matches_found: Number of duplicate block pairs
- interpretation: Low (&lt;2%), Medium (2-5%), High (&gt;5%)

**Detection Strength:**

- Excellent for: Object removal (content-aware fill), cloning, duplicate regions
- Limitation: Computationally intensive on very large images

**4.5 JPEG Compression Analysis**

**Function**: jpeg_compression_analysis(image_path)

Detects multiple re-compression cycles (sign of editing).

**How It Works:**

- Check if file is JPEG (only works on JPEGs)
- Estimate quality from bytes-per-pixel ratio
- Detect 8x8 block artifacts at boundaries
- Calculate artifact score (discontinuities at block edges)
- Assess double-compression likelihood

**Quality Estimation:**

- High (90-100): >2 bytes/pixel
- Medium (70-89): 1-2 bytes/pixel
- Low (<70): <1 byte/pixel

**Block Artifact Detection:**

- Check discontinuities at 8x8 pixel boundaries (JPEG DCT blocks)
- High artifact score = multiple re-saves

**Output:**

- quality_estimate: Quality range
- compression_level: Low/Medium/High
- artifact_score: Block discontinuity percentage
- double_compression_likelihood: Low/Medium/High
- interpretation: "Single compression (likely original)" vs "Possibly re-compressed (edited)"

**Detection Strength:**

- Excellent for: Re-saved/re-edited JPEGs
- Limitation: Only works on JPEG format

**4.6 AI Generation Detection**

**Function**: detect_ai_generated(image_path, output_path)

**2025 State-of-the-Art Detection** - Updated for modern GANs, Stable Diffusion, Midjourney, DALL-E.

**How It Works:**

**1\. Frequency Domain Analysis**

- FFT transform to frequency space
- Measure high-frequency vs low-frequency energy ratio
- AI images lack natural high-frequency details (overly smooth)
- **Threshold**: freq_ratio < 0.18 → +20 AI score

**2\. Saturation Uniformity**

- Convert to HSV color space
- Calculate standard deviation of saturation channel
- AI images have unnaturally uniform saturation
- **Threshold**: sat_std < 25 → +20 AI score

**3\. Texture Variance (Laplacian)**

- Apply Laplacian filter (edge detection)
- Calculate variance across image
- AI images have extremely low texture variance (too smooth)
- **Threshold**: lap_var < 50 → +25 AI score

**4\. Frequency Spectrum Flatness**

- Compute DFT magnitude spectrum
- Measure standard deviation of log spectrum
- Modern GANs have flat frequency fingerprints
- **Threshold**: spectrum_std < 1.1 → +20 AI score

**5\. Edge Density**

- Canny edge detection
- Calculate percentage of edge pixels
- AI images have overly smooth edges (under-detected)
- **Threshold**: edge_density < 0.012 → +20 AI score

**6\. LBP Entropy (Strongest Signal)**

- Local Binary Patterns (texture descriptor)
- Calculate histogram entropy
- AI images have very low texture entropy
- **Threshold**: lbp_entropy < 4.8 → +25 AI score

**7\. Face-Specific Artifacts** (if faces detected)

- Unusually high facial symmetry (>0.85 SSIM) → +10 AI score
- Smooth face boundary (blend artifacts) → +10 AI score

**Block-Level Suspicion Heatmap:**

- Divide image into 32x32 blocks
- Calculate per-block Laplacian variance and standard deviation
- Low variance blocks = suspicious
- Generate heatmap overlay

**Output:**

- ai_generation_score: 0-100 (capped at 100)
- verdict: "Very Likely AI" (≥85), "Likely AI" (≥60), "Possibly AI" (≥40), "Likely Real" (<40)
- confidence: High/Medium
- indicators: List of detected AI fingerprints
- faces_detected: Number of faces found
- analysis_details: Raw scores for each method
- heatmap: Suspicious region visualization

**Detection Strength:**

- **Excellent for**: Stable Diffusion, Midjourney, DALL-E 3, GANs, StyleGAN
- **Limitation**: High-quality AI images with post-processing may evade detection

**4.7 Reverse Image Search Preparation**

**Function**: reverse_image_search(image_path)

Generates information for manual reverse image search.

**Output:**

- Image MD5 hash
- Dimensions
- Links to search engines: Google Images, Yandex, TinEye, Bing
- Instructions for manual upload

**Note**: Automated reverse search requires API keys (not implemented)

**4.8 Enhancement & Analysis**

**Function**: enhance_and_analyze(image_path, output_dir)

Additional image analysis techniques:

**Luminance Enhancement:**

- Histogram equalization for better visibility
- Saves as enhanced_luminance.png

**Edge Detection:**

- Canny edge detection (100-200 thresholds)
- Saves as edge_detection.png

**Histogram Analysis:**

- Detect histogram peaks (manipulation signature)
- Mean brightness and standard deviation
- **Normal distribution**: 15-40 peaks
- **Suspicious**: &lt;15 or &gt;40 peaks

**Frequency Analysis (DCT):**

- Discrete Cosine Transform
- Visual representation of frequency content
- Saves as frequency_analysis.png

**4.9 Combined Heatmap**

**Function**: create_combined_heatmap(output_dir)

Merges all individual heatmaps into single comprehensive view:

- Load ELA, noise, and clone detection heatmaps
- Resize all to same dimensions
- Average pixel values across all heatmaps
- Add title and legend
- Save as combined_heatmap.png

**Purpose**: Single visualization showing all suspicious regions

**4.10 User-Friendly Summary**

**Function**: generate_user_friendly_summary(report)

Translates technical findings into plain language:

**Status Emoji:**

- 🔴 HIGH RISK (>60% tampering likelihood)
- 🟡 MEDIUM RISK (31-60%)
- 🟢 LOW RISK (0-30%)

**Issues Found (Negative Findings):**

- Uneven compression levels (ELA score)
- Noise pattern inconsistencies
- Copied/cloned regions
- Multiple re-saves detected
- Missing metadata
- AI generation indicators

**Positive Findings:**

- Consistent compression patterns
- Uniform noise distribution
- No copy-paste manipulation
- Single compression (not re-edited)
- Original camera metadata intact
- No AI generation indicators

**Recommendation:**

- Contextual advice based on risk level

**AI Detection Section:**

- AI score (0-100)
- Verdict (Very Likely AI / Likely AI / Possibly AI / Likely Real)
- List of specific indicators found

**5\. Security & Validation**

**Input Validation:**

- Extension whitelist: JPG, JPEG, PNG, BMP, TIFF, WebP
- 100MB file size limit
- UUID-based internal filenames (prevents collisions)

**Error Handling:**

- Try-catch in all analysis functions
- Corrupted file detection
- Graceful degradation (partial results if some analyses fail)
- Cleanup in finally blocks

**Resource Management:**

- Immediate file deletion after processing
- Temporary directory cleanup
- In-memory processing where possible

**6\. Edge Cases & Robustness**

**PNG/WebP/GIF Files**

**Challenge**: ELA and JPEG analysis don't work on non-JPEG formats

**Solution**:

- ELA skipped or returns N/A for PNG/WebP
- JPEG analysis returns format notification
- Noise/clone detection still work
- Overall assessment adjusts scoring logic

**AI-Generated Images**

**Modern AI Detection (2025)**:

- Detects Stable Diffusion, Midjourney, DALL-E 3
- Uses frequency analysis + texture analysis + LBP entropy
- 6+ independent indicators for high confidence
- Heatmap shows overly-smooth regions

**Low-Resolution Images**

**Challenge**: Small images lack sufficient data for block-based analysis

**Solution**:

- Copy-move detection automatically resizes to 800px max
- Smaller block sizes for tiny images
- Warning in report if resolution too low

**Corrupted/Partial Files**

**Challenge**: Truncated downloads, damaged files

**Solution**:

- PIL/OpenCV raise exceptions on invalid files
- HTTP 400 response with "Cannot open image file"
- No partial processing (fail fast)

**High-Resolution Images (>10MP)**

**Challenge**: Memory usage and processing time

**Solution**:

- Copy-move detection auto-resizes to 800px
- Block-based processing (constant memory)
- Typical 20MP image processes in 15-30 seconds

**7\. API Reference**

**POST /analyze**

**Request:**

Content-Type: multipart/form-data

Fields:

\- image: (file, required) Image file

\- format: (string, optional) "json" or "zip", default: "zip"

**Response (format=zip):**

200 OK

Content-Type: application/zip

Content-Disposition: attachment; filename="forensic_analysis_YYYYMMDD_HHMMSS.zip"

ZIP contents:

\- forensic_report.json

\- combined_heatmap.png

**Response (format=json):**

{

"analysis_timestamp": "2025-12-19T14:30:52",

"image_file": "photo.jpg",

"metadata": { ... },

"manipulation_detection": {

"ela": { "ela_score": 12.5, "interpretation": "Medium" },

"noise": { "noise_inconsistency_score": 18.3, "interpretation": "Low" },

"copy_move": { "clone_score": 1.2, "matches_found": 5 },

"jpeg_compression": { "double_compression_likelihood": "Low" },

"ai_generated": { "ai_generation_score": 35, "verdict": "Likely Real" }

},

"overall_assessment": {

"tampering_likelihood": 25,

"verdict": "Likely Authentic",

"confidence": "High"

},

"user_friendly_summary": {

"status": "🟢 LOW RISK",

"trust_level": "Likely Authentic",

"tampering_probability": "25%",

"issues_found": \[ ... \],

"positive_findings": \[ ... \],

"recommendation": "...",

"ai_detection": {

"score": 35,

"verdict": "Likely Real",

"indicators": \[ ... \]

}

}

}

**Error Responses:**

- 400 NO_FILE: No image uploaded
- 400 INVALID_FORMAT: Unsupported extension
- 400 VALIDATION_ERROR: Corrupted file
- 413 FILE_TOO_LARGE: Exceeds 100MB
- 500 ANALYSIS_ERROR: Processing failure

**POST /batch**

**Request:**

Content-Type: multipart/form-data

Fields:

\- images: (files, required) Multiple image files

**Response:**

200 OK

Content-Type: application/zip

ZIP structure:

/batch_summary.json

/image1_name/

\- report.json

\- combined_heatmap.png

/image2_name/

\- report.json

\- combined_heatmap.png

**batch_summary.json:**

{

"total_files": 5,

"successful": 4,

"failed": 1,

"results": \[

{

"filename": "photo1.jpg",

"status": "success",

"tampering_likelihood": 15,

"verdict": "Likely Authentic"

}

\],

"timestamp": "2025-12-19T14:30:52"

}

**POST /compare**

**Request:**

Content-Type: multipart/form-data

Fields:

\- image1: (file, required)

\- image2: (file, required)

**Response:**

{

"image1": "photo1.jpg",

"image2": "photo2.jpg",

"similarity": {

"similarity_score": 0.9234,

"interpretation": "Very Similar"

},

"hashes_match": false,

"image1_hash": "a1b2c3d4...",

"image2_hash": "e5f6g7h8..."

}

**POST /metadata**

**Request:**

Content-Type: multipart/form-data

Fields:

\- image: (file, required)

**Response:**

{

"filename": "photo.jpg",

"format": "JPEG",

"dimensions": "1920x1080",

"file_size_mb": 2.3,

"exif": {

"Make": "Canon",

"Model": "EOS 5D Mark IV",

"DateTimeOriginal": "2025:12:19 10:30:45",

"GPSInfo": {

"GPSLatitude": "37.7749",

"GPSLongitude": "-122.4194"

}

},

"md5": "a1b2c3d4e5f6...",

"sha256": "1a2b3c4d5e6f..."

}

**GET /health**

**Response:**

{

"status": "healthy",

"timestamp": "2025-12-19T14:30:52"

}

**8\. Scalability & Performance**

**Memory Efficiency:**

- Block-based processing (constant memory)
- Immediate file cleanup
- In-memory ZIP streaming

**Processing Time (typical hardware):**

- Small image (<1MP): 2-5 seconds
- Medium (5MP): 8-15 seconds
- Large (20MP): 20-40 seconds

**Factors affecting speed:**

- Resolution (higher = slower)
- Format (PNG faster than JPEG for some analyses)
- Number of detected duplicates (copy-move scales with matches)

**Concurrency:**

- Stateless design
- UUID-based file isolation
- Multi-worker gunicorn support

**9\. Deployment**

**Dependencies:**

flask==3.0.0

opencv-python-headless==4.8.1

numpy==1.26.0

pillow==10.1.0

scikit-image==0.22.0

scipy==1.11.4

**Docker Deployment:**

FROM python:3.11-slim

RUN apt-get update && apt-get install -y \\

libgl1-mesa-glx \\

libglib2.0-0 \\

&& rm -rf /var/lib/apt/lists/\*

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

EXPOSE 4000

CMD \["gunicorn", "--bind", "0.0.0.0:4000", "--workers", "4", "--timeout", "120", "main:app"\]

**Environment Variables:**

FLASK_APP=main.py

FLASK_ENV=production

MAX_CONTENT_LENGTH=104857600 # 100MB

UPLOAD_FOLDER=/tmp/uploads

OUTPUT_FOLDER=/tmp/outputs

**10\. Testing**

**cURL Examples:**

\# Basic analysis (returns ZIP)

curl -X POST <http://localhost:4000/analyze> \\

\-F "image=@photo.jpg" \\

\-o forensic_report.zip

\# JSON format (web UI)

curl -X POST <http://localhost:4000/analyze> \\

\-F "image=@photo.jpg" \\

\-F "format=json" | jq .

\# Batch analysis

curl -X POST <http://localhost:4000/batch> \\

\-F "images=@photo1.jpg" \\

\-F "images=@photo2.jpg" \\

\-F "images=@photo3.jpg" \\

\-o batch_report.zip

\# Compare images

curl -X POST <http://localhost:4000/compare> \\

\-F "image1=@original.jpg" \\

\-F "image2=@edited.jpg" | jq .

\# Metadata only

curl -X POST <http://localhost:4000/metadata> \\

\-F "image=@photo.jpg" | jq .

**Test Scenarios:**

- AI-generated image (Midjourney, Stable Diffusion)
- Photoshopped image (clone stamp, content-aware fill)
- Spliced image (multiple sources combined)
- Re-saved JPEG (multiple compressions)
- Pristine camera original (EXIF intact)
- Metadata-stripped image
- PNG screenshot vs JPEG photo

**11\. Future Enhancements**

**Planned Features:**

- Advanced AI model integration (dedicated CNN classifier)
- Blockchain verification (image fingerprinting)
- Temporal consistency check (video frame analysis)
- Shadow/lighting inconsistency detection
- Perspective/geometry analysis
- Face morphing detection
- Automated reverse image search (API integration)
- Real-time streaming analysis
- Batch processing optimization (parallel workers)

**Integration Points:**

- Content management systems (WordPress, Drupal)
- Social media platforms (automated fact-checking)
- Legal/forensic software
- Journalism verification workflows
- E-commerce fraud detection

**12\. Appendices**

**Glossary**

- **ELA**: Error Level Analysis - JPEG compression inconsistency detection
- **DCT**: Discrete Cosine Transform - frequency domain representation
- **LBP**: Local Binary Patterns - texture descriptor
- **SSIM**: Structural Similarity Index - image similarity metric
- **EXIF**: Exchangeable Image File Format - camera metadata
- **Clone Detection**: Finding duplicated regions within same image
- **Heatmap**: Color-coded visualization of suspicious regions

**Configuration Reference**

\# Flask Configuration

MAX_CONTENT_LENGTH = 100 \* 1024 \* 1024 # 100MB

UPLOAD_FOLDER = '/tmp/uploads'

OUTPUT_FOLDER = '/tmp/outputs'

\# Allowed Extensions

ALLOWED_EXTENSIONS = {'jpg', 'jpeg', 'png', 'bmp', 'tiff', 'webp'}

\# ELA Settings

ELA_QUALITY = 95

ELA_ENHANCEMENT_FACTOR = 10

\# Copy-Move Detection

BLOCK_SIZE = 32

MAX_COMPARISONS = 1000

\# AI Detection Thresholds (2025)

FREQ_RATIO_THRESHOLD = 0.18

SAT_STD_THRESHOLD = 25

LAP_VAR_THRESHOLD = 50

SPECTRUM_STD_THRESHOLD = 1.1

EDGE_DENSITY_THRESHOLD = 0.012

LBP_ENTROPY_THRESHOLD = 4.8

\# Noise Analysis

NOISE_WINDOW_SIZE = 15

MEDIAN_BLUR_KERNEL = 3

**Error Codes**

| **Code** | **HTTP** | **Description** |
| --- | --- | --- |
| NO_FILE | 400 | No image uploaded |
| EMPTY_FILENAME | 400 | File field empty |
| INVALID_FORMAT | 400 | Unsupported extension |
| VALIDATION_ERROR | 400 | Corrupted file |
| ANALYSIS_ERROR | 500 | Processing failure |
| FILE_TOO_LARGE | 413 | Exceeds 100MB |

**Support & Maintenance**

Update documentation when:

- Adding new forensic methods
- Modifying AI detection algorithms
- Changing threshold values
- Supporting new image formats
- Implementing planned enhancements

For issues, refer to structured logs and review individual analysis function outputs in the JSON report.
