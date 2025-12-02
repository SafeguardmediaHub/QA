# Environment Setup Guide
## Preparing Your Testing Environment

---

**[← Back to Test Plan](./TEST-PLAN.md)** | **[Next: Test Data →](./TEST-DATA.md)**

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [System Requirements](#system-requirements)
4. [Installation Steps](#installation-steps)
5. [Service Configuration](#service-configuration)
6. [Environment Variables](#environment-variables)
7. [Database Setup](#database-setup)
8. [AWS S3 Configuration](#aws-s3-configuration)
9. [Email Service Setup](#email-service-setup)
10. [Testing Tools Setup](#testing-tools-setup)
11. [Verification](#verification)
12. [Troubleshooting](#troubleshooting)

---

## Overview

This guide provides step-by-step instructions for setting up a complete testing environment for the SafeguardMedia Backend API. By the end of this setup, you will have:

- ✅ A running instance of the SafeguardMedia API
- ✅ MongoDB database configured and seeded with test data
- ✅ Redis cache and queue system operational
- ✅ AWS S3 bucket for file storage
- ✅ Email service for testing notifications
- ✅ API testing tools configured
- ✅ Test user accounts created
- ✅ Test files prepared

**Estimated Setup Time**: 1-2 hours

**Skill Level Required**: Intermediate (familiarity with Node.js, databases, command line)

---

## Prerequisites

Before beginning setup, ensure you have:

### Required Software

| Software | Version | Purpose | Installation Link |
|----------|---------|---------|-------------------|
| **Node.js** | 18.x or higher | Runtime environment | [nodejs.org](https://nodejs.org) |
| **pnpm** | 8.x or higher | Package manager | `npm install -g pnpm` |
| **MongoDB** | 5.x or higher | Primary database | [mongodb.com](https://www.mongodb.com/try/download/community) |
| **Redis** | 6.x or higher | Cache & queue backend | [redis.io](https://redis.io/download) |
| **Git** | Latest | Source control | [git-scm.com](https://git-scm.com) |

### Optional but Recommended

- **MongoDB Compass** - GUI for MongoDB database management
- **RedisInsight** - GUI for Redis management
- **Postman** or **Insomnia** - API testing tool with GUI
- **VS Code** - Text editor for viewing logs and configuration

### Required Access

- **AWS Account** - For S3 file storage (or test credentials)
- **Email Service** - SMTP credentials (can use Mailtrap, Mailhog, Gmail)
- **API Keys** - For external services (Gemini AI, deepfake detection)

### Knowledge Prerequisites

Basic understanding of:
- Command line/terminal usage
- REST APIs and HTTP methods
- JSON data format
- Environment variables
- Database concepts

---

## System Requirements

### Minimum Hardware

- **CPU**: 2 cores (4 cores recommended)
- **RAM**: 4 GB (8 GB recommended)
- **Disk Space**: 10 GB free (20 GB recommended for test files)
- **Network**: Stable internet connection for external service integrations

### Operating Systems

This guide covers setup for:
- **Linux** (Ubuntu 20.04+, Debian, etc.)
- **macOS** (10.15+)
- **Windows** (10/11 with WSL2 recommended)

### Ports Required

Ensure these ports are available:

| Port | Service | Purpose |
|------|---------|---------|
| 3000 | SafeguardMedia API | Main application server |
| 27017 | MongoDB | Database server |
| 6379 | Redis | Cache and queue server |
| 3001 | Frontend (optional) | If testing with frontend |

---

## Installation Steps

### Step 1: Clone the Repository

```bash
# Navigate to your projects directory
cd ~/projects

# Clone the repository (use your actual repository URL)
git clone <repository-url> safeguardmedia-server

# Navigate into the project directory
cd safeguardmedia-server

# Verify you're on the correct branch
git branch
```

**Expected Output**: Should show you're on the `main` branch (or appropriate testing branch)

### Step 2: Install Dependencies

```bash
# Install all npm dependencies using pnpm
pnpm install

# This may take 2-5 minutes depending on your internet connection
```

**Expected Output**:
```
Packages: +XXX
Progress: resolved XXX, reused XXX, downloaded XXX, added XXX
Done in XXs
```

**Troubleshooting**:
- If `pnpm` is not found: Run `npm install -g pnpm` first
- If you get permission errors: Don't use `sudo`; fix npm permissions instead
- If packages fail to install: Check your internet connection and npm registry

### Step 3: Verify Installation

```bash
# Check that TypeScript is installed
npx tsc --version

# Check the project structure
ls -la

# You should see directories: src/, dist/, node_modules/, etc.
```

---

## Service Configuration

### MongoDB Setup

#### Option A: Local MongoDB Installation

**For Ubuntu/Debian:**

```bash
# Import MongoDB public GPG key
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -

# Add MongoDB repository
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list

# Update package list
sudo apt-get update

# Install MongoDB
sudo apt-get install -y mongodb-org

# Start MongoDB service
sudo systemctl start mongod

# Enable MongoDB to start on boot
sudo systemctl enable mongod

# Verify MongoDB is running
sudo systemctl status mongod
```

**For macOS:**

```bash
# Install via Homebrew
brew tap mongodb/brew
brew install mongodb-community@6.0

# Start MongoDB service
brew services start mongodb-community@6.0

# Verify installation
mongo --version
```

**For Windows:**

1. Download MongoDB Community Server from [mongodb.com](https://www.mongodb.com/try/download/community)
2. Run the installer (`.msi` file)
3. Choose "Complete" installation
4. Install MongoDB Compass (GUI tool) when prompted
5. MongoDB will start as a Windows service automatically

#### Option B: MongoDB Atlas (Cloud)

If you prefer a cloud database:

1. Go to [mongodb.com/cloud/atlas](https://www.mongodb.com/cloud/atlas)
2. Create a free account
3. Create a new cluster (free tier available)
4. Create a database user
5. Whitelist your IP address (or use `0.0.0.0/0` for testing)
6. Get your connection string (looks like: `mongodb+srv://username:password@cluster.mongodb.net/dbname`)

**Verify MongoDB Connection:**

```bash
# Connect to local MongoDB
mongosh

# Or use connection string for Atlas
mongosh "mongodb+srv://username:password@cluster.mongodb.net/test"

# You should see MongoDB shell prompt
# Exit with: exit
```

### Redis Setup

#### Option A: Local Redis Installation

**For Ubuntu/Debian:**

```bash
# Install Redis
sudo apt-get update
sudo apt-get install redis-server -y

# Start Redis service
sudo systemctl start redis-server

# Enable Redis to start on boot
sudo systemctl enable redis-server

# Verify Redis is running
redis-cli ping
# Should respond: PONG
```

**For macOS:**

```bash
# Install via Homebrew
brew install redis

# Start Redis service
brew services start redis

# Verify installation
redis-cli ping
# Should respond: PONG
```

**For Windows:**

1. Download Redis from [github.com/microsoftarchive/redis/releases](https://github.com/microsoftarchive/redis/releases)
2. Extract the ZIP file
3. Run `redis-server.exe`
4. Or use WSL2 and follow Linux instructions

#### Option B: Redis Cloud

If you prefer a cloud Redis instance:

1. Go to [redis.com/try-free](https://redis.com/try-free/)
2. Create a free account
3. Create a new database
4. Get your connection string (format: `redis://username:password@host:port`)

**Verify Redis Connection:**

```bash
# Test connection
redis-cli

# Should see Redis prompt
# Test with a command
SET test "Hello"
GET test
# Should return: "Hello"

# Exit with: exit
```

---

## Environment Variables

### Step 1: Create Environment File

```bash
# In the project root directory
cp .env.example .env

# Or create a new .env file
touch .env
```

### Step 2: Configure Environment Variables

Open the `.env` file in your text editor and configure the following:

```bash
# ==============================================
# APPLICATION CONFIGURATION
# ==============================================
NODE_ENV=development
PORT=3000
HOST=localhost

# ==============================================
# DATABASE CONFIGURATION
# ==============================================

# MongoDB Connection String
# Local: mongodb://localhost:27017/safeguardmedia-test
# Atlas: mongodb+srv://username:password@cluster.mongodb.net/safeguardmedia-test
MONGODB_URI=mongodb://localhost:27017/safeguardmedia-test

# ==============================================
# AUTHENTICATION & SECURITY
# ==============================================

# JWT Secret for access tokens (minimum 32 characters)
# Generate with: node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
JWT_SECRET=your_super_secret_jwt_key_at_least_32_characters_long_replace_this

# JWT Refresh Secret (minimum 32 characters, different from JWT_SECRET)
JWT_REFRESH_SECRET=your_super_secret_refresh_key_at_least_32_characters_long_replace_this

# Token Expiration
JWT_EXPIRATION=15m
JWT_REFRESH_EXPIRATION=7d

# ==============================================
# REDIS CONFIGURATION
# ==============================================

# Redis Connection String
# Local: redis://localhost:6379
# Cloud: redis://username:password@host:port
REDIS_URL=redis://localhost:6379

# ==============================================
# AWS S3 CONFIGURATION
# ==============================================

# AWS Credentials (get from AWS IAM)
AWS_ACCESS_KEY_ID=your_aws_access_key_id
AWS_SECRET_ACCESS_KEY=your_aws_secret_access_key
AWS_REGION=us-east-1

# S3 Bucket Name (create a dedicated test bucket)
AWS_S3_BUCKET=safeguardmedia-test-bucket

# CloudFront Distribution (optional)
AWS_CLOUDFRONT_DOMAIN=

# ==============================================
# EMAIL SERVICE CONFIGURATION
# ==============================================

# SMTP Server Configuration
# For Gmail: smtp.gmail.com, port 587
# For Mailtrap: smtp.mailtrap.io, port 2525
SMTP_HOST=smtp.mailtrap.io
SMTP_PORT=2525
SMTP_USER=your_mailtrap_username
SMTP_PASSWORD=your_mailtrap_password
SMTP_FROM=noreply@safeguardmedia.test

# Frontend URL (for email links)
FRONTEND_URL=http://localhost:3001

# ==============================================
# EXTERNAL API SERVICES
# ==============================================

# Google Gemini AI
GEMINI_API_KEY=your_gemini_api_key

# Deepfake Detection Service
DEEPFAKE_API_KEY=your_deepfake_service_api_key
DEEPFAKE_API_URL=https://api.deepfakedetection.com

# Social Media APIs (optional for testing)
FACEBOOK_APP_ID=
FACEBOOK_APP_SECRET=
TWITTER_API_KEY=
TWITTER_API_SECRET=

# ==============================================
# FEATURE FLAGS & MONITORING
# ==============================================

# Enable Bull Board Queue Monitoring UI
BULL_BOARD_ENABLED=true

# Logging Level (error, warn, info, debug)
LOG_LEVEL=debug

# File Upload Limits
MAX_FILE_SIZE=104857600
MAX_FILES_PER_USER=1000

# User Storage Quota (in bytes, default 10GB)
USER_STORAGE_QUOTA=10737418240

# ==============================================
# RATE LIMITING
# ==============================================

# Rate limit: requests per window
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100

# ==============================================
# TESTING & DEVELOPMENT
# ==============================================

# Disable certain checks for testing
SKIP_EMAIL_VERIFICATION=false
DISABLE_RATE_LIMITING=false
```

### Step 3: Generate Secure Secrets

Generate secure JWT secrets using Node.js:

```bash
# Generate JWT_SECRET
node -e "console.log('JWT_SECRET=' + require('crypto').randomBytes(32).toString('hex'))"

# Generate JWT_REFRESH_SECRET
node -e "console.log('JWT_REFRESH_SECRET=' + require('crypto').randomBytes(32).toString('hex'))"
```

Copy the generated values into your `.env` file.

### Step 4: Verify Environment Variables

```bash
# Check that .env file exists
ls -la .env

# View the file (be careful with secrets!)
cat .env

# Verify Node.js can read environment variables
node -e "require('dotenv').config(); console.log('PORT:', process.env.PORT)"
```

---

## Database Setup

### Step 1: Create Test Database

```bash
# Connect to MongoDB
mongosh

# Switch to test database
use safeguardmedia-test

# Create a test collection
db.users.insertOne({test: true})

# Verify database exists
show dbs

# Exit MongoDB shell
exit
```

### Step 2: Run Database Migrations (if applicable)

```bash
# If your project has migration scripts
pnpm run migrate

# Or seed the database with initial data
pnpm run seed
```

**Note**: Check if your project has a `migrations/` directory or seed scripts.

### Step 3: Install MongoDB Compass (Optional)

MongoDB Compass provides a GUI for database management:

1. Download from [mongodb.com/products/compass](https://www.mongodb.com/products/compass)
2. Install and launch
3. Connect using connection string: `mongodb://localhost:27017`
4. Navigate to `safeguardmedia-test` database

---

## AWS S3 Configuration

### Step 1: Create AWS Account (if needed)

1. Go to [aws.amazon.com](https://aws.amazon.com)
2. Create an account (free tier available)
3. Complete verification

### Step 2: Create IAM User

1. Log into AWS Console
2. Navigate to **IAM** → **Users** → **Add User**
3. Username: `safeguardmedia-test`
4. Access type: **Programmatic access**
5. Click **Next: Permissions**

### Step 3: Attach S3 Permissions

1. Choose **Attach existing policies directly**
2. Search and select: `AmazonS3FullAccess` (or create custom policy)
3. Click **Next** through to **Create user**
4. **Important**: Copy `Access Key ID` and `Secret Access Key` immediately

### Step 4: Create S3 Bucket

```bash
# Using AWS CLI (install with: pip install awscli)
aws configure
# Enter your Access Key ID and Secret Access Key

# Create bucket (replace with unique name)
aws s3 mb s3://safeguardmedia-test-bucket-unique-name --region us-east-1

# Verify bucket exists
aws s3 ls
```

**Or via AWS Console**:

1. Navigate to **S3** → **Create bucket**
2. Bucket name: `safeguardmedia-test-bucket-unique-name`
3. Region: `us-east-1` (or your preferred region)
4. Block all public access: **Enabled** (recommended)
5. Click **Create bucket**

### Step 5: Configure CORS (if needed)

If testing with frontend:

1. Select your bucket
2. Go to **Permissions** → **CORS configuration**
3. Add configuration:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
    "AllowedOrigins": ["http://localhost:3000", "http://localhost:3001"],
    "ExposeHeaders": ["ETag"]
  }
]
```

### Step 6: Update Environment Variables

Add your AWS credentials to `.env`:

```bash
AWS_ACCESS_KEY_ID=your_access_key_id_here
AWS_SECRET_ACCESS_KEY=your_secret_access_key_here
AWS_REGION=us-east-1
AWS_S3_BUCKET=safeguardmedia-test-bucket-unique-name
```

---

## Email Service Setup

### Option A: Mailtrap (Recommended for Testing)

Mailtrap is a fake SMTP server perfect for testing:

1. Go to [mailtrap.io](https://mailtrap.io)
2. Create free account
3. Navigate to **Email Testing** → **Inboxes** → **My Inbox**
4. Click on **SMTP Settings**
5. Copy credentials

Update `.env`:

```bash
SMTP_HOST=smtp.mailtrap.io
SMTP_PORT=2525
SMTP_USER=your_mailtrap_username
SMTP_PASSWORD=your_mailtrap_password
SMTP_FROM=noreply@safeguardmedia.test
```

### Option B: Gmail (Use with Caution)

**Not recommended for automated testing due to rate limits**

1. Enable 2-factor authentication on your Google account
2. Generate an App Password:
   - Go to Google Account Settings → Security
   - Under "Signing in to Google", click "App passwords"
   - Generate password for "Mail" app

Update `.env`:

```bash
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password_here
SMTP_FROM=your_email@gmail.com
```

### Option C: Mailhog (Local)

Run a local SMTP server:

```bash
# Install Mailhog
# macOS
brew install mailhog

# Linux (download binary)
wget https://github.com/mailhog/MailHog/releases/download/v1.0.1/MailHog_linux_amd64
chmod +x MailHog_linux_amd64
sudo mv MailHog_linux_amd64 /usr/local/bin/mailhog

# Start Mailhog
mailhog
```

Access web UI at: `http://localhost:8025`

Update `.env`:

```bash
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_USER=
SMTP_PASSWORD=
SMTP_FROM=noreply@safeguardmedia.test
```

---

## Testing Tools Setup

### Postman Setup

1. **Download**: [postman.com/downloads](https://www.postman.com/downloads/)
2. **Install and Launch**
3. **Create Workspace**: "SafeguardMedia QA"
4. **Create Collection**: "SafeguardMedia API Tests"
5. **Set Base URL**:
   - Click on collection → Variables
   - Add variable: `baseUrl` = `http://localhost:3000`
6. **Create Environment**: "Local Testing"
   - Add `baseUrl` variable
   - Add `authToken` variable (will be set during login tests)

### Alternative: Insomnia

1. **Download**: [insomnia.rest/download](https://insomnia.rest/download)
2. **Install and Launch**
3. **Create Project**: "SafeguardMedia QA"
4. **Create Environment**:

```json
{
  "baseUrl": "http://localhost:3000",
  "authToken": ""
}
```

### cURL (Command Line)

No installation needed on most systems. Test with:

```bash
# Health check
curl http://localhost:3000/health

# Pretty print JSON
curl http://localhost:3000/health | json_pp
```

---

## Verification

### Step 1: Build the Application

```bash
# Build TypeScript to JavaScript
pnpm run build

# You should see a dist/ directory created
ls -la dist/
```

**Expected Output**: `dist/` directory with compiled `.js` files

### Step 2: Start the Application

```bash
# Start in development mode (with hot reload)
pnpm run dev
```

**Expected Output**:

```
[INFO] Server starting...
[INFO] MongoDB connected successfully
[INFO] Redis connected successfully
[INFO] Bull queues initialized
[INFO] Server is running on http://localhost:3000
[INFO] Environment: development
[INFO] Bull Board available at: http://localhost:3000/admin/queues
```

**Troubleshooting**:
- If MongoDB connection fails: Verify MongoDB is running and connection string is correct
- If Redis connection fails: Verify Redis is running
- If port 3000 is in use: Change PORT in `.env` or stop other process

### Step 3: Start the Worker (Separate Terminal)

```bash
# Open a new terminal window
cd safeguardmedia-server

# Start the worker process
pnpm run worker
```

**Expected Output**:

```
[INFO] Worker starting...
[INFO] Redis connected successfully
[INFO] Worker listening for jobs...
```

### Step 4: Verify Health Endpoint

```bash
# Test health check endpoint
curl http://localhost:3000/health

# Or open in browser: http://localhost:3000/health
```

**Expected Response**:

```json
{
  "success": true,
  "message": "SafeguardMedia API is healthy",
  "data": {
    "status": "healthy",
    "timestamp": "2025-12-02T...",
    "uptime": 12.345,
    "environment": "development",
    "services": {
      "database": "connected",
      "redis": "connected"
    }
  }
}
```

### Step 5: Verify Bull Board (Queue Monitoring)

Open in browser: `http://localhost:3000/admin/queues`

You should see the Bull Board dashboard with queue statistics.

### Step 6: Verify Database Connection

```bash
# Connect to MongoDB
mongosh

# Use your test database
use safeguardmedia-test

# Check collections
show collections

# Should show collections created by the application
```

### Step 7: Verify Redis Connection

```bash
# Connect to Redis
redis-cli

# Check for keys created by the application
KEYS *

# Should show session keys, cache keys, queue keys
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. MongoDB Connection Error

**Error**: `MongoNetworkError: failed to connect to server`

**Solutions**:
```bash
# Check if MongoDB is running
sudo systemctl status mongod

# Restart MongoDB
sudo systemctl restart mongod

# Check connection string in .env
# Ensure it matches your MongoDB setup
```

#### 2. Redis Connection Error

**Error**: `Error: Redis connection to localhost:6379 failed`

**Solutions**:
```bash
# Check if Redis is running
redis-cli ping

# Start Redis
sudo systemctl start redis-server

# Check Redis URL in .env
```

#### 3. Port Already in Use

**Error**: `Error: listen EADDRINUSE: address already in use :::3000`

**Solutions**:
```bash
# Find process using port 3000
lsof -i :3000

# Kill the process (replace PID)
kill -9 <PID>

# Or change PORT in .env
PORT=3001
```

#### 4. Missing Environment Variables

**Error**: `JWT_SECRET is required`

**Solutions**:
```bash
# Verify .env file exists
ls -la .env

# Check specific variable
grep JWT_SECRET .env

# Ensure no typos in variable names
```

#### 5. Permission Denied Errors

**Error**: `EACCES: permission denied`

**Solutions**:
```bash
# Don't use sudo with npm/pnpm
# Fix npm permissions: https://docs.npmjs.com/resolving-eacces-permissions-errors-when-installing-packages-globally

# Or use nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 18
nvm use 18
```

#### 6. Module Not Found Errors

**Error**: `Cannot find module 'xyz'`

**Solutions**:
```bash
# Reinstall dependencies
rm -rf node_modules
rm pnpm-lock.yaml
pnpm install

# Clear npm cache
pnpm store prune
```

#### 7. TypeScript Build Errors

**Error**: `error TS2304: Cannot find name...`

**Solutions**:
```bash
# Run type checking
pnpm run type-check

# Check TypeScript version
npx tsc --version

# Reinstall dependencies
pnpm install
```

---

## Next Steps

Congratulations! Your testing environment is now set up.

**What's Next?**

1. **[Create Test Data →](./TEST-DATA.md)** - Set up test accounts, files, and fixtures
2. **[Begin Testing →](./features/01-authentication.md)** - Start with authentication tests
3. **[API Reference →](./API-REFERENCE.md)** - Review complete API documentation

**Quick Checklist**:

- [ ] Node.js and pnpm installed
- [ ] MongoDB running and connected
- [ ] Redis running and connected
- [ ] Application starts successfully (`pnpm run dev`)
- [ ] Worker starts successfully (`pnpm run worker`)
- [ ] Health endpoint returns success
- [ ] Bull Board accessible
- [ ] AWS S3 bucket created (if testing uploads)
- [ ] Email service configured
- [ ] API testing tool (Postman/Insomnia) configured
- [ ] .env file configured with all required variables

**Need Help?**

- Check [Troubleshooting](#troubleshooting) section above
- Review application logs in terminal
- Contact development team
- Review CLAUDE.md for development commands

---

**[← Back to Test Plan](./TEST-PLAN.md)** | **[Next: Test Data →](./TEST-DATA.md)**

---

*Last Updated: December 2, 2025*
