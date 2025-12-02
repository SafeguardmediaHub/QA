# Feature 01: Authentication & Authorization
## Comprehensive Testing Documentation

---

**[← Back to Home](../README.md)** | **[Next: Media Upload →](./02-media-upload.md)**

---

## Table of Contents

1. [Feature Overview](#feature-overview)
2. [Test Scope](#test-scope)
3. [API Endpoints](#api-endpoints)
4. [Test Scenarios](#test-scenarios)
5. [Detailed Test Cases](#detailed-test-cases)
6. [Edge Cases & Boundary Testing](#edge-cases--boundary-testing)
7. [Security Testing](#security-testing)
8. [Performance Considerations](#performance-considerations)
9. [Test Results](#test-results)
10. [Issues Found](#issues-found)
11. [Recommendations](#recommendations)

---

## Feature Overview

### Description

The Authentication & Authorization system provides secure user registration, login, session management, and account security features. It implements JWT-based authentication with refresh token rotation, role-based access control, email verification, password reset, and account lockout protection.

### Importance

Authentication is the **foundation of system security**. All other features depend on proper user identification and authorization. Vulnerabilities in authentication can lead to:
- Unauthorized access to user data
- Account takeovers
- Data breaches
- Privacy violations
- Compliance issues

### Key Features

- ✅ User registration with email verification
- ✅ Secure login with JWT tokens
- ✅ Refresh token rotation for extended sessions
- ✅ Password strength validation
- ✅ Account lockout after failed login attempts
- ✅ Password reset flow with secure tokens
- ✅ Email verification system
- ✅ Secure logout with token blacklisting
- ✅ Role-based access control (User, Moderator, Admin, Super Admin)
- ✅ Rate limiting on sensitive endpoints

---

## Test Scope

### In Scope ✅

- User registration with all required fields
- Email uniqueness validation
- Password strength requirements
- Email verification flow (token generation, verification, expiration)
- Login with valid credentials
- Login with invalid credentials
- Account lockout mechanism
- Password reset request
- Password reset token validation
- Token refresh mechanism
- Logout functionality
- Token expiration
- Rate limiting on auth endpoints
- Authorization checks for protected routes
- Input validation and sanitization
- Error message consistency

### Out of Scope ❌

- Frontend UI/UX testing
- Two-factor authentication (if not implemented)
- OAuth/Social login (if not implemented)
- Password manager integration
- Browser autofill behavior
- Email delivery infrastructure (SMTP server reliability)

### Dependencies

- ✅ MongoDB - User data persistence
- ✅ Redis - Session storage, rate limiting, token blacklist
- ✅ Email Service - Verification and password reset emails
- ✅ Background Jobs - Email queue processing

---

## API Endpoints

### Base URL

```
http://localhost:3000/api/auth
```

### Endpoint Summary

| Endpoint | Method | Auth Required | Rate Limit | Description |
|----------|--------|---------------|------------|-------------|
| `/register` | POST | No | 5 req/15min | Register new user |
| `/login` | POST | No | 10 req/15min | User login |
| `/logout` | POST | Yes | None | User logout |
| `/refresh` | POST | No | None | Refresh access token |
| `/forgot-password` | POST | No | 3 req/15min | Request password reset |
| `/reset-password` | POST | No | 3 req/15min | Reset password with token |
| `/verify-email` | POST | No | 5 req/15min | Verify email address |
| `/resend-verification` | POST | No | 5 req/15min | Resend verification email |

---

## Detailed Endpoint Documentation

### 1. User Registration

#### Endpoint

```
POST /api/auth/register
```

#### Description

Registers a new user account and sends an email verification link. User account status is set to `PENDING_VERIFICATION` until email is verified.

#### Request Headers

```
Content-Type: application/json
```

#### Request Body

```json
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "firstName": "John",
  "lastName": "Doe",
  "agreedToTerms": true
}
```

**Field Specifications**:

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `email` | string | Yes | Valid email format, unique, lowercase | User's email address |
| `password` | string | Yes | Min 8 chars, 1 uppercase, 1 lowercase, 1 number, 1 special char | User's password |
| `firstName` | string | Yes | 2-50 chars, letters only | User's first name |
| `lastName` | string | Yes | 2-50 chars, letters only | User's last name |
| `agreedToTerms` | boolean | Yes | Must be `true` | Terms of service agreement |

**Password Requirements**:
- Minimum 8 characters
- At least one uppercase letter (A-Z)
- At least one lowercase letter (a-z)
- At least one number (0-9)
- At least one special character (!@#$%^&*)

#### Success Response (201 Created)

```json
{
  "success": true,
  "message": "User registered successfully. Please check your email for verification.",
  "data": {
    "user": {
      "id": "507f1f77bcf86cd799439011",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "accountStatus": "PENDING_VERIFICATION",
      "emailVerified": false
    }
  },
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Error Responses

**409 Conflict - Email Already Exists**

```json
{
  "success": false,
  "message": "User with this email already exists",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**400 Bad Request - Weak Password**

```json
{
  "success": false,
  "message": "Password does not meet these requirements: Must contain at least one uppercase letter. Must contain at least one special character.",
  "data": {
    "errors": [
      "Must contain at least one uppercase letter",
      "Must contain at least one special character"
    ]
  },
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**400 Bad Request - Missing Required Fields**

```json
{
  "success": false,
  "message": "Validation error",
  "data": {
    "errors": [
      {
        "field": "email",
        "message": "Email is required"
      },
      {
        "field": "agreedToTerms",
        "message": "You must agree to the terms of service"
      }
    ]
  },
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**429 Too Many Requests - Rate Limit Exceeded**

```json
{
  "success": false,
  "message": "Too many registration attempts. Please try again later.",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Side Effects

1. User document created in MongoDB with status `PENDING_VERIFICATION`
2. Email verification token generated and stored (hashed)
3. Verification email queued for delivery
4. Welcome notification created
5. Default media preferences set

---

### 2. User Login

#### Endpoint

```
POST /api/auth/login
```

#### Description

Authenticates user credentials and returns JWT access token and refresh token. Tokens are also set as HTTP-only cookies.

#### Request Headers

```
Content-Type: application/json
```

#### Request Body

```json
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "rememberMe": false
}
```

**Field Specifications**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | User's email address |
| `password` | string | Yes | User's password |
| `rememberMe` | boolean | No | If true, extends refresh token expiration to 30 days |

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "user": {
      "id": "507f1f77bcf86cd799439011",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "displayName": "John Doe",
      "role": "user",
      "accountStatus": "ACTIVE",
      "emailVerified": true,
      "profilePicture": null,
      "lastLoginAt": "2025-12-02T10:30:00.000Z"
    },
    "tokens": {
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    }
  },
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**Response Cookies**:

```
Set-Cookie: accessToken=eyJhbGc...; HttpOnly; Secure; SameSite=Strict; Max-Age=900
Set-Cookie: refreshToken=eyJhbGc...; HttpOnly; Secure; SameSite=Strict; Max-Age=604800
```

**Token Details**:
- `accessToken`: Short-lived (15 minutes), used for API requests
- `refreshToken`: Long-lived (7 days, 30 days if `rememberMe`), used to obtain new access tokens

#### Error Responses

**401 Unauthorized - Invalid Credentials**

```json
{
  "success": false,
  "message": "Invalid credentials",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**423 Locked - Account Locked**

```json
{
  "success": false,
  "message": "Account is temporarily locked due to too many failed login attempts",
  "data": {
    "lockoutRemainingMinutes": 25
  },
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**403 Forbidden - Account Banned**

```json
{
  "success": false,
  "message": "Account has been banned",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**403 Forbidden - Account Suspended**

```json
{
  "success": false,
  "message": "Account has been suspended",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**429 Too Many Requests - Rate Limit Exceeded**

```json
{
  "success": false,
  "message": "Too many login attempts. Please try again later.",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Side Effects

1. Failed login counter incremented on wrong password
2. Account locked after multiple failed attempts
3. Failed login counter reset on successful login
4. `lastLoginAt` and `lastActiveAt` timestamps updated
5. Refresh token stored in user document
6. Cookies set in response

#### Account Lockout Policy

- **Threshold**: 5 failed login attempts
- **Lockout Duration**: 30 minutes
- **Reset**: Counter resets on successful login

---

### 3. User Logout

#### Endpoint

```
POST /api/auth/logout
```

#### Description

Invalidates the current session by blacklisting the access token and removing the refresh token from the user's active tokens list.

#### Request Headers

```
Content-Type: application/json
Authorization: Bearer <access_token>
```

**Or via Cookie**:
```
Cookie: accessToken=<token>; refreshToken=<token>
```

#### Request Body

None required.

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "Logout successful",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**Response Headers**:

```
Set-Cookie: accessToken=; HttpOnly; Secure; SameSite=Strict; Max-Age=0
Set-Cookie: refreshToken=; HttpOnly; Secure; SameSite=Strict; Max-Age=0
```

#### Error Responses

**401 Unauthorized - No Token Provided**

```json
{
  "success": false,
  "message": "Authentication required",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**401 Unauthorized - Invalid Token**

```json
{
  "success": false,
  "message": "Invalid or expired token",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Side Effects

1. Access token added to Redis blacklist (expires after token TTL)
2. Refresh token removed from user's `refreshTokens` array
3. Cookies cleared in response

---

### 4. Refresh Token

#### Endpoint

```
POST /api/auth/refresh
```

#### Description

Exchanges a valid refresh token for a new access token and refresh token pair. Implements token rotation for enhanced security.

#### Request Headers

```
Content-Type: application/json
Cookie: refreshToken=<refresh_token>
```

**Or via Request Body**:

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "Token refreshed successfully",
  "data": {
    "tokens": {
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    }
  },
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**Response Cookies**:

```
Set-Cookie: accessToken=<new_token>; HttpOnly; Secure; SameSite=Strict; Max-Age=900
Set-Cookie: refreshToken=<new_token>; HttpOnly; Secure; SameSite=Strict; Max-Age=604800
```

#### Error Responses

**401 Unauthorized - No Refresh Token**

```json
{
  "success": false,
  "message": "Refresh token required",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**401 Unauthorized - Invalid Refresh Token**

```json
{
  "success": false,
  "message": "Invalid refresh token",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**403 Forbidden - Account Not Active**

```json
{
  "success": false,
  "message": "Account is not active",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Side Effects

1. Old refresh token removed from user's `refreshTokens` array
2. New refresh token added to user's `refreshTokens` array
3. New cookies set in response

---

### 5. Forgot Password

#### Endpoint

```
POST /api/auth/forgot-password
```

#### Description

Initiates password reset process by sending a password reset email with a secure token. Returns success message regardless of whether email exists (security best practice).

#### Request Headers

```
Content-Type: application/json
```

#### Request Body

```json
{
  "email": "user@example.com"
}
```

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "If an account with that email exists, a password reset link has been sent",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**Note**: Same response returned whether email exists or not (prevents email enumeration).

#### Error Responses

**429 Too Many Requests - Token Still Valid**

```json
{
  "success": false,
  "message": "Password reset already requested. Please wait 15 minutes before requesting again",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**429 Too Many Requests - Rate Limit**

```json
{
  "success": false,
  "message": "Too many password reset attempts. Please try again later.",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**500 Internal Server Error - Email Send Failed**

```json
{
  "success": false,
  "message": "Failed to send password reset email. Please try again later",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Side Effects

1. Password reset token generated and stored (hashed) with 1-hour expiration
2. Password reset email sent to user
3. Rate limiting prevents rapid-fire requests

---

### 6. Reset Password

#### Endpoint

```
POST /api/auth/reset-password
```

#### Description

Resets user password using the token received via email. Invalidates all existing refresh tokens (forces logout on all devices).

#### Request Headers

```
Content-Type: application/json
```

#### Request Body

```json
{
  "token": "a1b2c3d4e5f6...",
  "password": "NewSecurePass123!"
}
```

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "Password reset successful. Please login with your new password",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Error Responses

**400 Bad Request - Invalid/Expired Token**

```json
{
  "success": false,
  "message": "Invalid or expired password reset token",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**400 Bad Request - Weak Password**

```json
{
  "success": false,
  "message": "Password does not meet requirements",
  "data": {
    "errors": [
      "Password must be at least 8 characters",
      "Must contain at least one number"
    ]
  },
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Side Effects

1. Password updated (hashed with bcrypt)
2. Password reset token and expiration cleared
3. All refresh tokens cleared (logout all sessions)
4. `security.lastPasswordReset` timestamp updated

---

### 7. Verify Email

#### Endpoint

```
POST /api/auth/verify-email
```

#### Description

Verifies user's email address using the token sent during registration. Changes account status from `PENDING_VERIFICATION` to `ACTIVE`.

#### Request Headers

```
Content-Type: application/json
```

#### Request Body

```json
{
  "token": "a1b2c3d4e5f6..."
}
```

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "Email verified successfully",
  "data": {
    "user": {
      "id": "507f1f77bcf86cd799439011",
      "email": "user@example.com",
      "emailVerified": true,
      "accountStatus": "ACTIVE"
    }
  },
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Error Responses

**400 Bad Request - Invalid/Expired Token**

```json
{
  "success": false,
  "message": "Invalid or expired email verification token",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Side Effects

1. `emailVerified` set to `true`
2. Email verification token and expiration cleared
3. Account status changed to `ACTIVE` (if was `PENDING_VERIFICATION`)

---

### 8. Resend Email Verification

#### Endpoint

```
POST /api/auth/resend-verification
```

#### Description

Resends email verification token to user. Only works if email is not already verified.

#### Request Headers

```
Content-Type: application/json
```

#### Request Body

```json
{
  "email": "user@example.com"
}
```

#### Success Response (200 OK)

```json
{
  "success": true,
  "message": "If an account with that email exists and is not verified, a verification email has been sent",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Error Responses

**400 Bad Request - Already Verified**

```json
{
  "success": false,
  "message": "Email is already verified",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**429 Too Many Requests - Token Still Valid**

```json
{
  "success": false,
  "message": "Verification email already sent. Please wait 15 minutes before requesting again",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

**500 Internal Server Error - Email Send Failed**

```json
{
  "success": false,
  "message": "Failed to send verification email. Please try again later",
  "timestamp": "2025-12-02T10:30:00.000Z"
}
```

#### Side Effects

1. New email verification token generated
2. Verification email sent to user

---

## Test Scenarios

### TS-01: Complete Registration Flow

**Objective**: Verify end-to-end user registration process

**Steps**:
1. Register new user with valid data
2. Check email inbox for verification email
3. Extract verification token from email
4. Verify email using token
5. Login with new credentials
6. Verify user profile and account status

**Expected Outcome**: User successfully registered, verified, and can login

---

### TS-02: Login Success Flow

**Objective**: Verify successful login with valid credentials

**Steps**:
1. Login with valid email and password
2. Verify tokens returned
3. Verify cookies set
4. Use access token to make authenticated request
5. Verify user data returned

**Expected Outcome**: Login successful, tokens work for authenticated requests

---

### TS-03: Failed Login and Account Lockout

**Objective**: Verify account lockout mechanism after failed attempts

**Steps**:
1. Attempt login with wrong password (5 times)
2. Verify lockout message received
3. Attempt login with correct password (should still fail)
4. Wait for lockout period to expire
5. Login with correct password

**Expected Outcome**: Account locked after 5 attempts, unlocked after timeout

---

### TS-04: Token Refresh Flow

**Objective**: Verify token refresh mechanism

**Steps**:
1. Login successfully
2. Wait for access token to expire (15 minutes)
3. Use refresh token to obtain new tokens
4. Verify new tokens work
5. Verify old refresh token no longer works (rotation)

**Expected Outcome**: Tokens refreshed successfully, old tokens invalidated

---

### TS-05: Password Reset Flow

**Objective**: Verify complete password reset process

**Steps**:
1. Request password reset
2. Check email for reset link
3. Extract reset token
4. Reset password with valid new password
5. Verify old password no longer works
6. Login with new password

**Expected Outcome**: Password reset successfully, new password works

---

### TS-06: Logout and Token Invalidation

**Objective**: Verify logout properly invalidates tokens

**Steps**:
1. Login successfully
2. Save access and refresh tokens
3. Logout
4. Attempt to use old access token (should fail)
5. Attempt to use old refresh token (should fail)

**Expected Outcome**: All tokens invalidated after logout

---

## Detailed Test Cases

### Registration Tests

#### TC-AUTH-001: Successful User Registration

**Objective**: Verify user can register with valid data

**Priority**: P0 - Critical

**Prerequisites**:
- API server running
- Email service configured
- Database accessible

**Test Data**:
```json
{
  "email": "testuser001@example.com",
  "password": "SecurePass123!",
  "firstName": "Test",
  "lastName": "User",
  "agreedToTerms": true
}
```

**Test Steps**:
1. Send POST request to `/api/auth/register` with test data
2. Verify response status is `201 Created`
3. Verify response contains success message
4. Verify user object returned with correct data
5. Verify `accountStatus` is `PENDING_VERIFICATION`
6. Verify `emailVerified` is `false`
7. Check database for new user record
8. Check email queue for verification email job

**Expected Result**:
- Status: `201 Created`
- Response: Success message with user data
- Database: New user document created
- Email: Verification email queued

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

**Screenshot**: *[Add screenshot here]*

**Notes**: *Any observations*

---

#### TC-AUTH-002: Registration with Duplicate Email

**Objective**: Verify system prevents duplicate email registration

**Priority**: P0 - Critical

**Prerequisites**:
- User already exists with email `duplicate@example.com`

**Test Data**:
```json
{
  "email": "duplicate@example.com",
  "password": "SecurePass123!",
  "firstName": "Test",
  "lastName": "User",
  "agreedToTerms": true
}
```

**Test Steps**:
1. Send POST request to `/api/auth/register` with duplicate email
2. Verify response status is `409 Conflict`
3. Verify error message indicates email already exists
4. Check database to confirm no duplicate user created

**Expected Result**:
- Status: `409 Conflict`
- Message: "User with this email already exists"
- No duplicate user in database

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

**Screenshot**: *[Add screenshot here]*

---

#### TC-AUTH-003: Registration with Weak Password

**Objective**: Verify password strength validation

**Priority**: P0 - Critical

**Prerequisites**: None

**Test Data** (Multiple Cases):

**Case 1: Too Short**
```json
{
  "email": "test@example.com",
  "password": "Short1!",
  "firstName": "Test",
  "lastName": "User",
  "agreedToTerms": true
}
```

**Case 2: No Uppercase**
```json
{
  "password": "nocapital123!"
}
```

**Case 3: No Lowercase**
```json
{
  "password": "NOLOWERCASE123!"
}
```

**Case 4: No Number**
```json
{
  "password": "NoNumbers!"
}
```

**Case 5: No Special Character**
```json
{
  "password": "NoSpecial123"
}
```

**Test Steps**:
1. For each test case, send POST request to `/api/auth/register`
2. Verify response status is `400 Bad Request`
3. Verify error message lists specific password requirements not met
4. Verify no user created in database

**Expected Result**:
- Status: `400 Bad Request`
- Message: Lists specific validation errors
- `data.errors` array contains requirement descriptions

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

**Screenshot**: *[Add screenshot here]*

---

#### TC-AUTH-004: Registration with Missing Required Fields

**Objective**: Verify all required fields are validated

**Priority**: P1 - High

**Prerequisites**: None

**Test Data** (Multiple Cases):

**Case 1: Missing Email**
```json
{
  "password": "SecurePass123!",
  "firstName": "Test",
  "lastName": "User",
  "agreedToTerms": true
}
```

**Case 2: Missing Password**
```json
{
  "email": "test@example.com",
  "firstName": "Test",
  "lastName": "User",
  "agreedToTerms": true
}
```

**Case 3: Missing Terms Agreement**
```json
{
  "email": "test@example.com",
  "password": "SecurePass123!",
  "firstName": "Test",
  "lastName": "User"
}
```

**Case 4: Terms Not True**
```json
{
  "email": "test@example.com",
  "password": "SecurePass123!",
  "firstName": "Test",
  "lastName": "User",
  "agreedToTerms": false
}
```

**Test Steps**:
1. For each case, send POST request with missing field
2. Verify response status is `400 Bad Request`
3. Verify error message indicates which field is missing/invalid
4. Verify no user created

**Expected Result**:
- Status: `400 Bad Request`
- Error details specify missing/invalid field

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-005: Registration with Invalid Email Format

**Objective**: Verify email format validation

**Priority**: P1 - High

**Test Data** (Invalid Emails):
- `notanemail`
- `missing@domain`
- `@nodomain.com`
- `spaces in@email.com`
- `double@@domain.com`

**Test Steps**:
1. For each invalid email, attempt registration
2. Verify `400 Bad Request` response
3. Verify error indicates invalid email format

**Expected Result**:
- Status: `400 Bad Request`
- Clear validation error for email field

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-006: Registration Rate Limiting

**Objective**: Verify rate limiting prevents abuse

**Priority**: P1 - High

**Prerequisites**: Rate limit set to 5 requests per 15 minutes

**Test Steps**:
1. Send 5 valid registration requests (different emails)
2. Send 6th registration request
3. Verify 6th request returns `429 Too Many Requests`
4. Wait 15 minutes
5. Send another registration request
6. Verify it succeeds

**Expected Result**:
- First 5 requests succeed
- 6th request gets rate limited
- After timeout, requests work again

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

### Email Verification Tests

#### TC-AUTH-007: Successful Email Verification

**Objective**: Verify email verification process

**Priority**: P0 - Critical

**Prerequisites**:
- User registered with unverified email

**Test Steps**:
1. Register new user
2. Extract verification token from database or email
3. Send POST request to `/api/auth/verify-email` with token
4. Verify response is `200 OK`
5. Verify `emailVerified` is now `true`
6. Verify `accountStatus` changed to `ACTIVE`
7. Check database to confirm changes

**Expected Result**:
- Status: `200 OK`
- `emailVerified: true`
- `accountStatus: "ACTIVE"`

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-008: Email Verification with Invalid Token

**Objective**: Verify invalid tokens are rejected

**Priority**: P0 - Critical

**Test Data**:
```json
{
  "token": "invalid_token_12345"
}
```

**Test Steps**:
1. Send POST request with invalid token
2. Verify response is `400 Bad Request`
3. Verify error message indicates invalid token
4. Check user in database remains unverified

**Expected Result**:
- Status: `400 Bad Request`
- Message: "Invalid or expired email verification token"
- User remains unverified

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-009: Email Verification with Expired Token

**Objective**: Verify expired tokens are rejected

**Priority**: P0 - Critical

**Prerequisites**:
- User with expired verification token (>24 hours old)

**Test Steps**:
1. Create user with old verification token
2. Attempt verification with expired token
3. Verify `400 Bad Request` response
4. Verify user remains unverified

**Expected Result**:
- Status: `400 Bad Request`
- User remains unverified

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-010: Resend Email Verification

**Objective**: Verify verification email can be resent

**Priority**: P1 - High

**Prerequisites**:
- Unverified user account

**Test Data**:
```json
{
  "email": "unverified@example.com"
}
```

**Test Steps**:
1. Send POST request to `/api/auth/resend-verification`
2. Verify `200 OK` response
3. Check email queue for new verification email
4. Verify new token generated in database
5. Use new token to verify email successfully

**Expected Result**:
- Status: `200 OK`
- New verification email sent
- New token works for verification

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-011: Resend Verification for Already Verified Email

**Objective**: Verify cannot resend for verified email

**Priority**: P2 - Medium

**Prerequisites**:
- User with already verified email

**Test Data**:
```json
{
  "email": "verified@example.com"
}
```

**Test Steps**:
1. Send POST request to `/api/auth/resend-verification`
2. Verify response is `400 Bad Request`
3. Verify error message indicates email already verified

**Expected Result**:
- Status: `400 Bad Request`
- Message: "Email is already verified"

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

### Login Tests

#### TC-AUTH-012: Successful Login

**Objective**: Verify successful login with valid credentials

**Priority**: P0 - Critical

**Prerequisites**:
- Active user account with verified email

**Test Data**:
```json
{
  "email": "activeuser@example.com",
  "password": "SecurePass123!",
  "rememberMe": false
}
```

**Test Steps**:
1. Send POST request to `/api/auth/login`
2. Verify response is `200 OK`
3. Verify response contains user object
4. Verify response contains `accessToken` and `refreshToken`
5. Verify cookies are set in response headers
6. Verify `lastLoginAt` updated in database
7. Verify refresh token stored in user's `refreshTokens` array
8. Use access token to make authenticated request

**Expected Result**:
- Status: `200 OK`
- Valid JWT tokens returned
- Cookies set with HttpOnly flag
- Database updated with login timestamp
- Token works for authenticated requests

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

**Screenshot**: *[Add screenshot here]*

---

#### TC-AUTH-013: Login with Invalid Email

**Objective**: Verify login fails with non-existent email

**Priority**: P0 - Critical

**Test Data**:
```json
{
  "email": "nonexistent@example.com",
  "password": "AnyPassword123!"
}
```

**Test Steps**:
1. Send POST request with non-existent email
2. Verify response is `401 Unauthorized`
3. Verify error message is generic (doesn't reveal user doesn't exist)
4. Verify no tokens returned

**Expected Result**:
- Status: `401 Unauthorized`
- Message: "Invalid credentials" (generic)
- No user data or tokens returned

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-014: Login with Wrong Password

**Objective**: Verify login fails with incorrect password

**Priority**: P0 - Critical

**Prerequisites**:
- User exists with known password

**Test Data**:
```json
{
  "email": "testuser@example.com",
  "password": "WrongPassword123!"
}
```

**Test Steps**:
1. Send POST request with wrong password
2. Verify response is `401 Unauthorized`
3. Verify error message is generic
4. Check database: `security.failedLoginAttempts` incremented
5. Verify no tokens returned

**Expected Result**:
- Status: `401 Unauthorized`
- Message: "Invalid credentials"
- Failed login counter incremented
- No tokens

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-015: Account Lockout After Failed Attempts

**Objective**: Verify account locks after 5 failed login attempts

**Priority**: P0 - Critical

**Prerequisites**:
- Fresh user account with 0 failed attempts

**Test Steps**:
1. Attempt login with wrong password (1st attempt)
2. Verify `401 Unauthorized` response
3. Repeat steps 1-2 four more times (total 5 attempts)
4. Attempt 6th login (even with correct password)
5. Verify response is `423 Locked`
6. Verify error message indicates account is locked
7. Verify `lockoutRemainingMinutes` provided in response
8. Check database: `security.lockoutUntil` set to 30 minutes from now

**Expected Result**:
- First 5 attempts: `401 Unauthorized`
- 6th attempt: `423 Locked`
- Lockout message with remaining time
- Database reflects lockout state

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

**Screenshot**: *[Add screenshot here]*

---

#### TC-AUTH-016: Login After Lockout Period Expires

**Objective**: Verify can login after lockout expires

**Priority**: P0 - Critical

**Prerequisites**:
- Account in locked state (or simulate by setting `lockoutUntil` in past)

**Test Steps**:
1. Create locked account or wait for lockout to expire
2. Attempt login with correct credentials
3. Verify login succeeds
4. Verify `200 OK` response with tokens
5. Verify `security.failedLoginAttempts` reset to 0
6. Verify `security.lockoutUntil` cleared

**Expected Result**:
- Login successful after lockout expires
- Failed attempts counter reset
- Lockout cleared

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-017: Login with Banned Account

**Objective**: Verify banned accounts cannot login

**Priority**: P0 - Critical

**Prerequisites**:
- User account with `accountStatus: "BANNED"`

**Test Data**:
```json
{
  "email": "banned@example.com",
  "password": "CorrectPassword123!"
}
```

**Test Steps**:
1. Attempt login with banned account
2. Verify response is `403 Forbidden`
3. Verify error message indicates account banned
4. Verify no tokens returned

**Expected Result**:
- Status: `403 Forbidden`
- Message: "Account has been banned"
- No tokens

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-018: Login with Suspended Account

**Objective**: Verify suspended accounts cannot login

**Priority**: P0 - Critical

**Prerequisites**:
- User account with `accountStatus: "SUSPENDED"`

**Test Steps**:
1. Attempt login with suspended account
2. Verify response is `403 Forbidden`
3. Verify error message indicates suspension

**Expected Result**:
- Status: `403 Forbidden`
- Message: "Account has been suspended"

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-019: Login with Remember Me

**Objective**: Verify "Remember Me" extends refresh token expiration

**Priority**: P2 - Medium

**Test Data**:
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "rememberMe": true
}
```

**Test Steps**:
1. Login with `rememberMe: true`
2. Extract refresh token from response
3. Decode JWT to check expiration
4. Verify refresh token expiration is 30 days (instead of default 7 days)
5. Verify refresh token cookie `Max-Age` is 30 days

**Expected Result**:
- Refresh token expires in 30 days
- Cookie `Max-Age` is 30 days (2592000 seconds)

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-020: Login Rate Limiting

**Objective**: Verify login rate limiting prevents brute force

**Priority**: P1 - High

**Prerequisites**:
- Rate limit set to 10 requests per 15 minutes

**Test Steps**:
1. Send 10 login requests rapidly
2. Send 11th login request
3. Verify 11th request returns `429 Too Many Requests`
4. Wait for rate limit window to expire
5. Verify login works again

**Expected Result**:
- First 10 requests processed (may succeed or fail)
- 11th request rate limited
- After timeout, requests work again

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

### Token Management Tests

#### TC-AUTH-021: Access Token Authentication

**Objective**: Verify access token works for authenticated requests

**Priority**: P0 - Critical

**Prerequisites**:
- Valid access token from login

**Test Steps**:
1. Login and obtain access token
2. Make request to protected endpoint with token in header: `Authorization: Bearer <token>`
3. Verify request succeeds with `200 OK`
4. Make same request without token
5. Verify request fails with `401 Unauthorized`

**Expected Result**:
- With token: Request succeeds
- Without token: `401 Unauthorized`

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-022: Expired Access Token

**Objective**: Verify expired access tokens are rejected

**Priority**: P0 - Critical

**Prerequisites**:
- Access token that has expired (wait 15+ minutes after login)

**Test Steps**:
1. Login and obtain access token
2. Wait for access token to expire (15 minutes)
3. Attempt to use expired token for authenticated request
4. Verify request fails with `401 Unauthorized`
5. Verify error message indicates token expired

**Expected Result**:
- Status: `401 Unauthorized`
- Message indicates token expired

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-023: Refresh Token to Get New Access Token

**Objective**: Verify refresh token mechanism works

**Priority**: P0 - Critical

**Prerequisites**:
- Valid refresh token

**Test Steps**:
1. Login and save both tokens
2. Wait for access token to expire
3. Send POST request to `/api/auth/refresh` with refresh token
4. Verify response is `200 OK`
5. Verify new access token and refresh token returned
6. Verify new access token works for authenticated requests
7. Verify old refresh token no longer works (rotation)

**Expected Result**:
- New tokens generated successfully
- Old refresh token invalidated (rotation)
- New access token works

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-024: Refresh Token with Invalid Token

**Objective**: Verify invalid refresh tokens are rejected

**Priority**: P0 - Critical

**Test Data**:
```json
{
  "refreshToken": "invalid.jwt.token"
}
```

**Test Steps**:
1. Send POST request to `/api/auth/refresh` with invalid token
2. Verify response is `401 Unauthorized`
3. Verify error message indicates invalid token

**Expected Result**:
- Status: `401 Unauthorized`
- Message: "Invalid refresh token"

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-025: Refresh Token After Logout

**Objective**: Verify refresh token doesn't work after logout

**Priority**: P0 - Critical

**Test Steps**:
1. Login and save refresh token
2. Logout
3. Attempt to use refresh token from step 1
4. Verify request fails with `401 Unauthorized`

**Expected Result**:
- Refresh token invalidated after logout
- Status: `401 Unauthorized`

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

### Logout Tests

#### TC-AUTH-026: Successful Logout

**Objective**: Verify logout invalidates session

**Priority**: P0 - Critical

**Prerequisites**:
- Logged in user with active session

**Test Steps**:
1. Login and save access and refresh tokens
2. Send POST request to `/api/auth/logout` with access token in header
3. Verify response is `200 OK`
4. Verify cookies cleared in response
5. Attempt to use old access token for authenticated request
6. Verify request fails with `401 Unauthorized`
7. Attempt to use old refresh token
8. Verify refresh fails with `401 Unauthorized`

**Expected Result**:
- Logout successful
- All tokens invalidated
- Cookies cleared

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-027: Logout Without Token

**Objective**: Verify logout requires authentication

**Priority**: P1 - High

**Test Steps**:
1. Send POST request to `/api/auth/logout` without auth token
2. Verify response is `401 Unauthorized`

**Expected Result**:
- Status: `401 Unauthorized`
- Error message indicates authentication required

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

### Password Reset Tests

#### TC-AUTH-028: Successful Password Reset Flow

**Objective**: Verify complete password reset process

**Priority**: P0 - Critical

**Prerequisites**:
- Active user account

**Test Steps**:
1. Send POST request to `/api/auth/forgot-password` with user email
2. Verify `200 OK` response
3. Check email inbox for password reset email
4. Extract reset token from email
5. Send POST request to `/api/auth/reset-password` with token and new password
6. Verify `200 OK` response
7. Attempt login with old password (should fail)
8. Login with new password
9. Verify login succeeds

**Expected Result**:
- Password reset email received
- Password successfully reset
- Old password no longer works
- New password works for login

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

**Screenshot**: *[Add screenshot here]*

---

#### TC-AUTH-029: Forgot Password with Non-Existent Email

**Objective**: Verify forgot password doesn't reveal if email exists

**Priority**: P0 - Critical (Security)

**Test Data**:
```json
{
  "email": "nonexistent@example.com"
}
```

**Test Steps**:
1. Send POST request with non-existent email
2. Verify response is `200 OK`
3. Verify message is generic (same as for existing email)
4. Verify no email sent (check email queue)

**Expected Result**:
- Status: `200 OK`
- Generic success message (prevents email enumeration)
- No email actually sent

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-030: Reset Password with Invalid Token

**Objective**: Verify invalid reset tokens are rejected

**Priority**: P0 - Critical

**Test Data**:
```json
{
  "token": "invalid_token",
  "password": "NewPassword123!"
}
```

**Test Steps**:
1. Send POST request with invalid token
2. Verify response is `400 Bad Request`
3. Verify error message indicates invalid token

**Expected Result**:
- Status: `400 Bad Request`
- Message: "Invalid or expired password reset token"

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-031: Reset Password with Expired Token

**Objective**: Verify expired reset tokens don't work

**Priority**: P0 - Critical

**Prerequisites**:
- Password reset token older than 1 hour

**Test Steps**:
1. Request password reset
2. Wait for token to expire (>1 hour) or manually set expiration in database
3. Attempt password reset with expired token
4. Verify `400 Bad Request` response

**Expected Result**:
- Status: `400 Bad Request`
- Password not changed

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-032: Reset Password with Weak Password

**Objective**: Verify password strength validation on reset

**Priority**: P1 - High

**Test Data**:
```json
{
  "token": "<valid_token>",
  "password": "weak"
}
```

**Test Steps**:
1. Get valid reset token
2. Attempt password reset with weak password
3. Verify `400 Bad Request` response
4. Verify error lists validation requirements not met

**Expected Result**:
- Status: `400 Bad Request`
- Error details specify password requirements

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-033: Password Reset Invalidates All Sessions

**Objective**: Verify password reset logs out all devices

**Priority**: P1 - High

**Prerequisites**:
- User logged in on multiple sessions (multiple refresh tokens)

**Test Steps**:
1. Login twice from "different devices" (save both refresh tokens)
2. Request and complete password reset
3. Attempt to use both old refresh tokens
4. Verify both fail with `401 Unauthorized`
5. Login with new password
6. Verify login succeeds

**Expected Result**:
- All refresh tokens invalidated after password reset
- Must login again with new password

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-034: Forgot Password Rate Limiting

**Objective**: Verify forgot password has rate limiting

**Priority**: P1 - High

**Prerequisites**:
- Rate limit: 3 requests per 15 minutes

**Test Steps**:
1. Send 3 forgot password requests for same email
2. Send 4th request immediately
3. Verify 4th request returns `429 Too Many Requests`
4. Wait 15 minutes
5. Verify request works again

**Expected Result**:
- First 3 requests succeed
- 4th request rate limited

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

#### TC-AUTH-035: Forgot Password Prevents Token Reuse

**Objective**: Verify requesting new reset token invalidates old one

**Priority**: P2 - Medium

**Test Steps**:
1. Request password reset (save token 1)
2. Immediately request password reset again (save token 2)
3. Attempt to use token 1 for reset
4. Verify token 1 fails
5. Use token 2 for reset
6. Verify token 2 succeeds

**Expected Result**:
- Only most recent token works
- Previous tokens invalidated

**Actual Result**: ⏳ *To be filled during testing*

**Status**: ⏳ Pending

---

## Edge Cases & Boundary Testing

### Edge Case Tests

#### TC-AUTH-036: Very Long Email Address

**Objective**: Verify system handles extremely long email addresses

**Test Data**:
```json
{
  "email": "verylongemailaddressthatgoesonyeskeepgoingitgoesonforeverandever@extremelylongdomainnamethatisridiculous.com",
  "password": "SecurePass123!",
  "firstName": "Test",
  "lastName": "User",
  "agreedToTerms": true
}
```

**Expected Result**: Either succeeds (if within limits) or fails gracefully with validation error

**Status**: ⏳ Pending

---

#### TC-AUTH-037: Special Characters in Name Fields

**Objective**: Verify handling of special characters in names

**Test Data**:
- Names with accents: "José", "François", "Müller"
- Names with hyphens: "Mary-Jane"
- Names with apostrophes: "O'Connor"
- Names with numbers: "Test123" (should fail)
- Names with symbols: "Test@User" (should fail)

**Expected Result**: Valid international characters accepted, invalid characters rejected

**Status**: ⏳ Pending

---

#### TC-AUTH-038: Email Case Sensitivity

**Objective**: Verify emails are case-insensitive

**Test Steps**:
1. Register user with email: `TestUser@Example.COM`
2. Verify email stored as lowercase: `testuser@example.com`
3. Attempt login with: `testuser@example.com`
4. Verify login succeeds
5. Attempt to register again with `TESTUSER@EXAMPLE.COM`
6. Verify duplicate email error

**Expected Result**: Emails treated as case-insensitive throughout system

**Status**: ⏳ Pending

---

#### TC-AUTH-039: Maximum Password Length

**Objective**: Verify system handles very long passwords

**Test Data**: Password with 1000 characters

**Expected Result**: System either accepts (if no max limit) or returns clear validation error

**Status**: ⏳ Pending

---

#### TC-AUTH-040: Unicode Characters in Password

**Objective**: Verify handling of Unicode characters in passwords

**Test Data**:
```json
{
  "password": "Pássw0rd!测试🔒"
}
```

**Expected Result**: Unicode characters handled correctly (either accepted or rejected with clear message)

**Status**: ⏳ Pending

---

#### TC-AUTH-041: Concurrent Login Attempts

**Objective**: Verify system handles simultaneous login requests

**Test Steps**:
1. Send 10 login requests simultaneously for same user
2. Verify all requests processed
3. Verify all return valid tokens (or appropriate error)
4. Verify failed login counter accurate
5. Check for race conditions in lockout mechanism

**Expected Result**: No race conditions, consistent behavior across concurrent requests

**Status**: ⏳ Pending

---

#### TC-AUTH-042: Token Refresh During Expiration Window

**Objective**: Verify token refresh works when access token is about to expire

**Test Steps**:
1. Login and get tokens
2. Wait until access token is about to expire (14 minutes 50 seconds)
3. Refresh token
4. Verify new tokens received
5. Use new access token immediately

**Expected Result**: Refresh works smoothly during expiration window

**Status**: ⏳ Pending

---

#### TC-AUTH-043: Multiple Active Refresh Tokens

**Objective**: Verify user can have multiple active sessions

**Test Steps**:
1. Login from "Device 1" (save tokens)
2. Login from "Device 2" (save tokens)
3. Login from "Device 3" (save tokens)
4. Verify all refresh tokens work
5. Logout from "Device 1"
6. Verify "Device 2" and "Device 3" tokens still work
7. Verify "Device 1" token no longer works

**Expected Result**: Multiple sessions supported, logout only affects specific session

**Status**: ⏳ Pending

---

#### TC-AUTH-044: Zero-Length and Whitespace Fields

**Objective**: Verify validation rejects empty/whitespace-only fields

**Test Data**:
```json
{
  "email": "   ",
  "password": "",
  "firstName": "\t\n",
  "lastName": "     "
}
```

**Expected Result**: Validation errors for all fields

**Status**: ⏳ Pending

---

#### TC-AUTH-045: SQL/NoSQL Injection in Email Field

**Objective**: Verify input sanitization prevents injection

**Test Data**:
```json
{
  "email": "admin@test.com'; DROP TABLE users;--",
  "password": "Test123!"
}
```

**Expected Result**: Input treated as literal string, no database manipulation

**Status**: ⏳ Pending

---

## Security Testing

### Security Test Cases

#### TC-AUTH-SEC-001: JWT Token Signature Validation

**Objective**: Verify tokens with invalid signatures are rejected

**Priority**: P0 - Critical

**Test Steps**:
1. Login and obtain valid JWT token
2. Decode token and modify payload (change user ID)
3. Re-encode without proper signature
4. Attempt to use modified token
5. Verify request fails with `401 Unauthorized`

**Expected Result**: Modified tokens rejected

**Status**: ⏳ Pending

---

#### TC-AUTH-SEC-002: Password Hashing Verification

**Objective**: Verify passwords are stored as hashes, not plain text

**Priority**: P0 - Critical

**Test Steps**:
1. Register new user with known password
2. Query database directly for user record
3. Examine `password` field
4. Verify it's a bcrypt hash (starts with `$2b$` or `$2a$`)
5. Verify hash is different from plain text password

**Expected Result**: Passwords stored as bcrypt hashes

**Status**: ⏳ Pending

---

#### TC-AUTH-SEC-003: Token Blacklist Verification

**Objective**: Verify logged-out tokens are blacklisted

**Priority**: P0 - Critical

**Test Steps**:
1. Login and save access token
2. Logout
3. Check Redis for blacklisted token
4. Attempt to use blacklisted token
5. Verify request fails

**Expected Result**: Tokens blacklisted in Redis after logout

**Status**: ⏳ Pending

---

#### TC-AUTH-SEC-004: HTTPS Requirement for Cookies

**Objective**: Verify authentication cookies have Secure flag

**Priority**: P0 - Critical (Production)

**Test Steps**:
1. Login in production/staging with HTTPS
2. Examine response headers
3. Verify cookies have `Secure` flag
4. Verify cookies have `HttpOnly` flag
5. Verify cookies have `SameSite=Strict` or `SameSite=Lax`

**Expected Result**: Cookies properly secured in production

**Status**: ⏳ Pending

---

#### TC-AUTH-SEC-005: Timing Attack Resistance

**Objective**: Verify login response times don't leak information

**Priority**: P2 - Medium

**Test Steps**:
1. Measure response time for login with non-existent email
2. Measure response time for login with wrong password
3. Compare response times
4. Verify no significant time difference (<100ms variance)

**Expected Result**: Response times consistent regardless of failure reason

**Status**: ⏳ Pending

---

#### TC-AUTH-SEC-006: Cross-Site Request Forgery (CSRF) Protection

**Objective**: Verify CSRF protection on state-changing endpoints

**Priority**: P1 - High

**Test Steps**:
1. Attempt to make POST request to `/api/auth/login` from different origin
2. Check CORS headers
3. Verify SameSite cookie attribute
4. Attempt CSRF attack simulation

**Expected Result**: CSRF protection mechanisms in place

**Status**: ⏳ Pending

---

#### TC-AUTH-SEC-007: Email Enumeration Prevention

**Objective**: Verify system doesn't reveal which emails exist

**Priority**: P1 - High

**Test Steps**:
1. Request password reset for existing email
2. Request password reset for non-existent email
3. Compare responses
4. Verify both return identical success messages
5. Attempt registration with existing email
6. Verify error is specific (different from password reset)

**Expected Result**: Password reset and verification endpoints don't leak email existence

**Status**: ⏳ Pending

---

#### TC-AUTH-SEC-008: Token Replay Attack Prevention

**Objective**: Verify tokens can't be replayed after logout

**Priority**: P0 - Critical

**Test Steps**:
1. Login and save tokens
2. Make successful authenticated request
3. Logout
4. Replay exact same authenticated request
5. Verify request fails

**Expected Result**: Replayed tokens don't work after logout

**Status**: ⏳ Pending

---

#### TC-AUTH-SEC-009: Refresh Token Rotation

**Objective**: Verify refresh tokens are rotated (one-time use)

**Priority**: P0 - Critical

**Test Steps**:
1. Login and save refresh token (Token A)
2. Use Token A to refresh (get Token B)
3. Attempt to use Token A again
4. Verify Token A no longer works
5. Use Token B to refresh
6. Verify Token B works once

**Expected Result**: Refresh tokens are single-use, rotated on each refresh

**Status**: ⏳ Pending

---

#### TC-AUTH-SEC-010: XSS Prevention in Error Messages

**Objective**: Verify error messages don't execute scripts

**Priority**: P1 - High

**Test Data**:
```json
{
  "email": "<script>alert('XSS')</script>@test.com",
  "password": "Test123!"
}
```

**Test Steps**:
1. Send registration/login with script in email
2. Verify error message doesn't include unescaped script
3. Verify response is JSON (Content-Type check)
4. If error includes user input, verify it's HTML-escaped

**Expected Result**: No script execution, proper output encoding

**Status**: ⏳ Pending

---

## Performance Considerations

### Performance Benchmarks

These are reference benchmarks, not pass/fail tests:

| Operation | Target Response Time | Notes |
|-----------|---------------------|-------|
| Registration | < 500ms | Excludes email sending (queued) |
| Login | < 200ms | Includes database query and token generation |
| Logout | < 100ms | Quick operation |
| Token Refresh | < 100ms | Lightweight operation |
| Password Reset Request | < 300ms | Includes email queueing |
| Email Verification | < 200ms | Database update |

### Load Testing Considerations

- **Concurrent Logins**: System should handle 100+ concurrent login requests
- **Rate Limiting**: Should gracefully handle burst traffic beyond limits
- **Database Queries**: Login queries should use indexes on email/username
- **Token Generation**: bcrypt hashing may be CPU-intensive under load

---

## Test Results

### Summary

| Category | Total Tests | Passed | Failed | Blocked | Pass Rate |
|----------|-------------|--------|--------|---------|-----------|
| Registration | 6 | 0 | 0 | 0 | 0% |
| Email Verification | 5 | 0 | 0 | 0 | 0% |
| Login | 9 | 0 | 0 | 0 | 0% |
| Token Management | 5 | 0 | 0 | 0 | 0% |
| Logout | 2 | 0 | 0 | 0 | 0% |
| Password Reset | 8 | 0 | 0 | 0 | 0% |
| Edge Cases | 10 | 0 | 0 | 0 | 0% |
| Security | 10 | 0 | 0 | 0 | 0% |
| **TOTAL** | **55** | **0** | **0** | **0** | **0%** |

*Table will be updated as testing progresses*

### Test Execution Log

| Date | Tester | Tests Run | Pass | Fail | Notes |
|------|--------|-----------|------|------|-------|
| TBD | TBD | 0 | 0 | 0 | Testing not started |

---

## Issues Found

### Critical Issues (P0)

No issues found yet.

### High Priority Issues (P1)

No issues found yet.

### Medium Priority Issues (P2)

No issues found yet.

### Low Priority Issues (P3)

No issues found yet.

---

## Recommendations

### Security Recommendations

1. **Two-Factor Authentication**: Consider implementing 2FA for enhanced security
2. **Password History**: Prevent users from reusing recent passwords
3. **Suspicious Login Detection**: Alert users to logins from new devices/locations
4. **Session Management UI**: Allow users to view and revoke active sessions

### Usability Recommendations

1. **Password Strength Indicator**: Show real-time password strength feedback
2. **Email Verification Reminder**: Prompt unverified users to verify email
3. **Login Activity Log**: Show recent login history to users

### Performance Recommendations

1. **Redis Caching**: Cache user profiles to reduce database queries
2. **Token Rotation Optimization**: Consider rotating refresh tokens less frequently
3. **Rate Limiting Granularity**: Different limits for different user roles

### Testing Recommendations

1. **Automated Tests**: Implement automated regression tests for authentication
2. **Load Testing**: Conduct comprehensive load testing before production
3. **Security Audit**: Regular third-party security audits
4. **Penetration Testing**: Engage security professionals for pen testing

---

## Appendix

### Test Data Reference

**Test User Accounts** (create before testing):

| Email | Password | Role | Status | Verified | Purpose |
|-------|----------|------|--------|----------|---------|
| admin@test.local | Admin123! | admin | ACTIVE | Yes | Admin testing |
| user@test.local | User123! | user | ACTIVE | Yes | Normal user testing |
| unverified@test.local | Unverified123! | user | PENDING_VERIFICATION | No | Email verification testing |
| suspended@test.local | Suspended123! | user | SUSPENDED | Yes | Suspended account testing |
| banned@test.local | Banned123! | user | BANNED | Yes | Banned account testing |
| locked@test.local | Locked123! | user | ACTIVE | Yes | Account lockout testing (set failedLoginAttempts: 5) |

### API Client Configuration

**Postman Collection Variables**:
```json
{
  "baseUrl": "http://localhost:3000",
  "apiPrefix": "/api/auth",
  "accessToken": "",
  "refreshToken": ""
}
```

**Example Request** (Postman):
```
POST {{baseUrl}}{{apiPrefix}}/login
Content-Type: application/json

{
  "email": "{{testEmail}}",
  "password": "{{testPassword}}"
}
```

### Related Documentation

- [User Model Schema](../../src/models/User.ts)
- [Auth Controller](../../src/controllers/auth.controller.ts)
- [Auth Routes](../../src/routes/auth.ts)
- [Auth Middleware](../../src/middleware/auth.ts)
- [Test Plan](../TEST-PLAN.md)

---

**[← Back to Home](../README.md)** | **[Next: Media Upload →](./02-media-upload.md)**

---

*Last Updated: December 2, 2025*
