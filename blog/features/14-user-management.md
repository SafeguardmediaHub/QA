# Feature 14: User Management

## Overview

The **User Management** feature provides authenticated users with complete control over their account profiles, preferences, security settings, and media quota tracking. This self-service system allows users to manage their personal information, customize their experience, secure their accounts, and monitor their resource usage without administrator intervention.

### What is User Management?

User Management encompasses all user-facing account operations:

1. **Profile Management**: Update personal information (name, bio, website, location, profile picture)
2. **Preference Management**: Customize notifications, privacy settings, theme, language, and timezone
3. **Password Management**: Change password with strength validation and security checks
4. **Account Deletion**: Soft-delete account with password confirmation and explicit consent
5. **Quota Monitoring**: Track media storage usage, file counts, and upload capacity

This feature empowers users to:

- **Maintain Current Information**: Keep profiles up-to-date with accurate contact and biographical details
- **Personalize Experience**: Configure UI preferences, notification settings, and privacy controls
- **Enhance Security**: Regularly update passwords with strong password requirements
- **Control Data**: Delete accounts while preserving data integrity through soft-deletion
- **Monitor Usage**: Track storage consumption and understand upload limits

### Why is User Management Important?

- **User Autonomy**: Self-service reduces dependency on support teams
- **Data Accuracy**: Users maintain their own information, ensuring freshness
- **Privacy Compliance**: GDPR/CCPA-compliant account deletion and data control
- **Security Best Practices**: Password change functionality encourages regular rotation
- **Resource Management**: Quota visibility prevents unexpected upload failures
- **User Satisfaction**: Customization options improve user experience and retention

---

## Architecture

The User Management system follows a RESTful architecture with authentication-protected endpoints:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        USER MANAGEMENT SYSTEM                           │
└─────────────────────────────────────────────────────────────────────────┘

Authentication Layer
┌──────────────────────────────────────────────────────────────────────┐
│  All requests require JWT authentication                             │
│  Token extracted from Authorization: Bearer <token>                  │
│  Middleware: authenticate (verifies token, attaches req.user)        │
└──────────────────────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      USER PROFILE OPERATIONS                         │
│                                                                      │
│  GET /api/users/me                                                  │
│  ┌────────────────────────────────────────────────┐                │
│  │ Retrieve Current User Profile                  │                │
│  │ - Returns complete user object                 │                │
│  │ - Includes: profile, role, preferences,        │                │
│  │   mediaQuota, accountStatus, timestamps        │                │
│  │ - Excludes: password, tokens (sensitive data)  │                │
│  └────────────────────────────────────────────────┘                │
│                          ▼                                           │
│  PATCH /api/users/me                                                │
│  ┌────────────────────────────────────────────────┐                │
│  │ Update Profile Fields                          │                │
│  │ - Allowed: firstName, lastName, displayName,   │                │
│  │   bio, website, location, profilePicture       │                │
│  │ - Validates website URL format                 │                │
│  │ - Rate limited (generalLimiter)                │                │
│  │ - Logs update with changed fields              │                │
│  └────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     PREFERENCES MANAGEMENT                           │
│                                                                      │
│  PATCH /api/users/me/preferences                                    │
│  ┌────────────────────────────────────────────────┐                │
│  │ Update User Preferences                        │                │
│  │                                                 │                │
│  │ Notifications:                                 │                │
│  │   - email: boolean                             │                │
│  │   - push: boolean                              │                │
│  │   - marketing: boolean                         │                │
│  │                                                 │                │
│  │ Privacy:                                       │                │
│  │   - profileVisibility: public|private          │                │
│  │   - showEmail: boolean                         │                │
│  │   - showLastActive: boolean                    │                │
│  │                                                 │                │
│  │ Theme: light | dark | auto                     │                │
│  │ Language: en, es, fr, etc.                     │                │
│  │ Timezone: America/New_York, UTC, etc.          │                │
│  │                                                 │                │
│  │ Media Preferences:                             │                │
│  │   - autoOptimize: boolean                      │                │
│  │   - defaultVisibility: public|private|unlisted │                │
│  │   - allowDownloads: boolean                    │                │
│  │   - watermarkEnabled: boolean                  │                │
│  └────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      PASSWORD MANAGEMENT                             │
│                                                                      │
│  PATCH /api/users/me/password                                       │
│  ┌────────────────────────────────────────────────┐                │
│  │ Change Password Flow                           │                │
│  │                                                 │                │
│  │ 1. Verify Current Password                     │                │
│  │    - bcrypt.compare(currentPassword, hash)     │                │
│  │    - Fail: 400 Bad Request                     │                │
│  │                                                 │                │
│  │ 2. Validate New Password Strength              │                │
│  │    - Minimum 8 characters                      │                │
│  │    - Contains uppercase letter                 │                │
│  │    - Contains lowercase letter                 │                │
│  │    - Contains number                           │                │
│  │    - Contains special character                │                │
│  │    - Fail: 400 with detailed errors            │                │
│  │                                                 │                │
│  │ 3. Check New ≠ Current                         │                │
│  │    - Prevent password reuse                    │                │
│  │    - Fail: 400 Bad Request                     │                │
│  │                                                 │                │
│  │ 4. Update Password + Security Metadata         │                │
│  │    - Hash new password (bcrypt)                │                │
│  │    - Set security.lastPasswordReset            │                │
│  │    - Revoke all refresh tokens except current  │                │
│  │    - Log password change event                 │                │
│  │                                                 │                │
│  │ 5. Maintain Active Session                     │                │
│  │    - Keep current refresh token valid          │                │
│  │    - User remains logged in                    │                │
│  └────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      ACCOUNT DELETION                                │
│                                                                      │
│  DELETE /api/users/me                                               │
│  ┌────────────────────────────────────────────────┐                │
│  │ Soft Delete Account                            │                │
│  │                                                 │                │
│  │ 1. Password Verification                       │                │
│  │    - Confirm user identity                     │                │
│  │    - Fail: 400 Bad Request                     │                │
│  │                                                 │                │
│  │ 2. Explicit Confirmation Check                 │                │
│  │    - Must send: confirmDeletion =              │                │
│  │      "DELETE_MY_ACCOUNT"                       │                │
│  │    - Prevents accidental deletion              │                │
│  │    - Fail: 400 Bad Request                     │                │
│  │                                                 │                │
│  │ 3. Soft Delete Execution                       │                │
│  │    - Set isDeleted = true                      │                │
│  │    - Set deletedAt = now()                     │                │
│  │    - Preserve data for auditing                │                │
│  │    - Clear all refresh tokens                  │                │
│  │                                                 │                │
│  │ 4. Session Termination                         │                │
│  │    - Clear accessToken cookie                  │                │
│  │    - Clear refreshToken cookie                 │                │
│  │    - Log account deletion event                │                │
│  │                                                 │                │
│  │ Note: Hard deletion handled by scheduled job   │                │
│  │ after retention period (e.g., 30-90 days)      │                │
│  └────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      MEDIA QUOTA TRACKING                            │
│                                                                      │
│  GET /api/users/me/media-quota                                      │
│  ┌────────────────────────────────────────────────┐                │
│  │ Retrieve Quota Information                     │                │
│  │                                                 │                │
│  │ Quota Data:                                    │                │
│  │   - used: bytes consumed                       │                │
│  │   - limit: total bytes allowed                 │                │
│  │   - fileCount: number of files                 │                │
│  │   - maxFiles: maximum files allowed            │                │
│  │                                                 │                │
│  │ Computed Metrics:                              │                │
│  │   - usedPercentage = (used/limit) × 100        │                │
│  │   - fileCountPercentage =                      │                │
│  │     (fileCount/maxFiles) × 100                 │                │
│  │   - canUpload = user.canUploadMedia()          │                │
│  │                                                 │                │
│  │ Includes Media Preferences                     │                │
│  │                                                 │                │
│  │ Requires: emailVerified = true                 │                │
│  │ (Email verification middleware)                │                │
│  └────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
```

## Real-World Test Scenarios

### Scenario 1: New User Completes Profile ✅

**Context**: Recently registered user updates profile with complete information.

**Test Flow**:

1. User logs in (JWT obtained)
2. GET /users/me → Sees default profile (minimal data)
3. PATCH /users/me with:
   - firstName: "Alice"
   - lastName: "Johnson"
   - bio: "Journalist at Tech Daily"
   - website: "https://techd daily.com/alice"
   - location: "New York, NY"
   - profilePicture: S3 URL
4. System validates website URL
5. System updates allowed fields only
6. Returns updated profile
7. User verifies changes by calling GET /users/me again

**Expected Result**:

- Profile updated successfully
- Website URL validated (must start with http/https)
- Restricted fields (email, role) unchanged
- Audit log records update with changed fields

---

### Scenario 2: User Configures Privacy Settings ✅

**Context**: Privacy-conscious user restricts profile visibility and disables marketing emails.

**Test Flow**:

1. User navigates to preferences page
2. PATCH /users/me/preferences with:
   ```json
   {
     "privacy": {
       "profileVisibility": "private",
       "showEmail": false,
       "showLastActive": false
     },
     "notifications": {
       "email": true,
       "push": true,
       "marketing": false
     }
   }
   ```
3. System applies partial update (only privacy + notifications)
4. Theme, language, timezone remain unchanged
5. Returns updated preferences

**Expected Result**:

- Privacy settings applied immediately
- Profile hidden from public search
- Email/lastActive hidden on profile page
- Marketing emails cease (unsubscribed)
- Other preferences (theme, language) unaffected

---

### Scenario 3: Scheduled Password Rotation ✅

**Context**: Security-conscious user changes password every 90 days per company policy.

**Test Flow**:

1. User submits PATCH /users/me/password:
   - currentPassword: "OldPass123!"
   - newPassword: "Weak"
2. System validates new password strength
3. Returns 400 error with detailed validation failures
4. User submits again with strong password: "NewSecure456#"
5. System verifies current password (bcrypt)
6. System validates new password strength (passes)
7. System checks new ≠ current (passes)
8. System updates password, sets lastPasswordReset timestamp
9. System revokes all refresh tokens except current session
10. User remains logged in, other sessions invalidated

**Expected Result**:

- Weak password rejected with specific errors
- Strong password accepted
- All other devices/sessions logged out
- Current session maintained
- security.lastPasswordReset updated
- Audit log records password change

---

### Scenario 4: User Exceeds Media Quota ✅

**Context**: Free-tier user approaches storage limit and checks quota before upload.

**Test Flow**:

1. User has used 950 MB of 1 GB limit
2. GET /users/me/media-quota returns:
   ```json
   {
     "quota": {
       "used": 994050048,
       "limit": 1073741824,
       "usedPercentage": 92.58,
       "canUpload": true
     }
   }
   ```
3. Frontend shows warning: "92% quota used"
4. User attempts to upload 200 MB file
5. Upload fails (would exceed limit)
6. User checks quota again:
   ```json
   {
     "quota": {
       "used": 1067450368,
       "limit": 1073741824,
       "usedPercentage": 99.41,
       "canUpload": false
     }
   }
   ```
7. Frontend displays: "Storage full. Upgrade or delete files."

**Expected Result**:

- Accurate quota tracking
- Percentage calculated correctly
- canUpload flag reflects actual capacity
- Upload prevented when quota exceeded
- User prompted to upgrade subscription

---

### Scenario 5: Account Deletion with GDPR Compliance ✅

**Context**: User exercises "Right to Erasure" under GDPR.

**Test Flow**:

1. User submits DELETE /users/me:
   - password: "correct password"
   - confirmDeletion: "DELETE_MY_ACCOUNT"
2. System verifies password (bcrypt)
3. System checks confirmation text (exact match)
4. System executes soft delete:
   - Sets isDeleted = true
   - Sets deletedAt = 2025-12-03T10:40:00Z
   - Clears all refresh tokens
   - Clears cookies (accessToken, refreshToken)
5. System logs deletion event
6. Returns 200 OK
7. User cannot log in (authentication rejects deleted accounts)
8. Data retained for 30 days (retention period)
9. After 30 days, scheduled job performs hard deletion:
   - Deletes user record
   - Deletes associated media files
   - Removes from all indexes

**Expected Result**:

- Account immediately inaccessible
- Data preserved temporarily for audit/legal requirements
- Hard deletion scheduled automatically
- GDPR/CCPA compliance maintained
- User can request data export before deletion (separate endpoint)

---

### Scenario 6: Theme Preference Synchronization ✅

**Context**: User switches between devices and expects consistent theme preference.

**Test Flow**:

1. User on desktop sets theme via PATCH /users/me/preferences:
   ```json
   { "theme": "dark" }
   ```
2. System updates preferences.theme = "dark"
3. User opens mobile app
4. Mobile app calls GET /users/me
5. Receives theme: "dark"
6. Mobile app applies dark theme
7. User changes to "auto" on mobile
8. Mobile submits PATCH /users/me/preferences:
   ```json
   { "theme": "auto" }
   ```
9. Desktop browser refreshes
10. Fetches GET /users/me → theme: "auto"
11. Desktop applies auto theme (system-based)

**Expected Result**:

- Theme preference synced across all devices
- Changes reflect immediately in database
- All clients retrieve latest preference
- Auto mode respects system theme (light/dark)

---

## Test Cases

### Profile Retrieval Tests (GET /users/me)

#### TC-UM-001: Retrieve Complete User Profile ✅

**Preconditions**: User authenticated, profile complete
**Steps**:

1. Send GET /api/users/me with valid JWT token

**Expected Result**:

- 200 OK
- Response includes all profile fields:
  - id, email, username, firstName, lastName, displayName, fullName
  - bio, website, location, profilePicture
  - role, accountStatus, emailVerified
  - lastLoginAt, lastActiveAt, createdAt
  - mediaQuota (used, limit, fileCount, maxFiles)
  - mediaPreferences, preferences
- Excludes sensitive fields: password, refreshTokens, tokens
- Response time: < 200ms

---

#### TC-UM-002: Retrieve Profile with Minimal Data ✅

**Preconditions**: New user with only required fields populated
**Steps**:

1. GET /users/me

**Expected Result**:

- 200 OK
- Required fields present: id, email, username, firstName, lastName, role
- Optional fields null or default: bio (null), displayName (null), website (null)
- Default preferences applied (e.g., theme: "light", language: "en")

---

#### TC-UM-003: Unauthenticated Profile Request ✅

**Preconditions**: No JWT token provided
**Steps**:

1. GET /users/me without Authorization header

**Expected Result**:

- 401 Unauthorized
- Error: "Authentication required"

---

#### TC-UM-004: Expired Token ✅

**Preconditions**: User has expired JWT token
**Steps**:

1. GET /users/me with expired token

**Expected Result**:

- 401 Unauthorized
- Error: "Token expired"
- Client should attempt token refresh

---

#### TC-UM-005: Deleted Account Access ✅

**Preconditions**: User account soft-deleted (isDeleted: true)
**Steps**:

1. GET /users/me with deleted user's token

**Expected Result**:

- 401 Unauthorized OR 404 Not Found
- Error: "Account not found" or "Account has been deleted"

---

### Profile Update Tests (PATCH /users/me)

#### TC-UM-006: Update Single Profile Field ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me with:
   ```json
   { "firstName": "UpdatedName" }
   ```

**Expected Result**:

- 200 OK
- firstName updated to "UpdatedName"
- Other fields unchanged
- Response includes updated user object
- Audit log records change

---

#### TC-UM-007: Update Multiple Profile Fields ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me with:
   ```json
   {
     "firstName": "Jane",
     "lastName": "Smith",
     "bio": "Product Designer",
     "location": "Austin, TX"
   }
   ```

**Expected Result**:

- 200 OK
- All 4 fields updated
- Response shows updated values
- Single database write operation

---

#### TC-UM-008: Update with Valid Website URL ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me with:
   ```json
   { "website": "https://example.com" }
   ```

**Expected Result**:

- 200 OK
- Website updated
- URL validation passed

---

#### TC-UM-009: Update with Invalid Website URL ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me with:
   ```json
   { "website": "not-a-valid-url" }
   ```

**Expected Result**:

- 400 Bad Request
- Error: "Website must be a valid URL"
- Website field unchanged in database

---

#### TC-UM-010: Attempt to Update Restricted Field (Email) ✅

**Preconditions**: User authenticated with email: "old@example.com"
**Steps**:

1. PATCH /users/me with:
   ```json
   { "email": "new@example.com" }
   ```

**Expected Result**:

- 200 OK (request succeeds but email ignored)
- email field unchanged (still "old@example.com")
- Only allowed fields updated
- Note: Email changes require separate verification flow

---

#### TC-UM-011: Attempt to Update Restricted Field (Role) ✅

**Preconditions**: Regular user (role: "user")
**Steps**:

1. PATCH /users/me with:
   ```json
   { "role": "admin" }
   ```

**Expected Result**:

- 200 OK (request succeeds but role ignored)
- role remains "user"
- Role changes require admin action (see Admin Features)

---

#### TC-UM-012: Clear Optional Fields ✅

**Preconditions**: User has bio: "Old bio"
**Steps**:

1. PATCH /users/me with:
   ```json
   { "bio": "" }
   ```

**Expected Result**:

- 200 OK
- bio set to empty string or null
- Field cleared successfully

---

#### TC-UM-013: Rate Limit on Profile Updates ✅

**Preconditions**: User authenticated
**Steps**:

1. Make 100 PATCH /users/me requests in 60 seconds

**Expected Result**:

- First N requests: 200 OK (N = rate limit threshold)
- Subsequent requests: 429 Too Many Requests
- Error: "Too many requests"
- Retry-After header present

---

#### TC-UM-014: Long String Validation ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me with bio exceeding 500 characters

**Expected Result**:

- 400 Bad Request
- Validation error: "Bio cannot exceed 500 characters"

---

#### TC-UM-015: Special Characters in Profile Fields ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me with:
   ```json
   {
     "displayName": "João_Müller-Ñ",
     "bio": "Specialist in AI/ML & data science 🤖"
   }
   ```

**Expected Result**:

- 200 OK
- Unicode characters handled correctly
- Emoji preserved
- No encoding issues

---

### Preferences Update Tests (PATCH /users/me/preferences)

#### TC-UM-016: Update Notification Preferences ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/preferences with:
   ```json
   {
     "notifications": {
       "email": false,
       "push": true,
       "marketing": false
     }
   }
   ```

**Expected Result**:

- 200 OK
- preferences.notifications updated
- Other preferences unchanged
- User unsubscribed from email notifications

---

#### TC-UM-017: Update Privacy Settings ✅

**Preconditions**: User authenticated with public profile
**Steps**:

1. PATCH /users/me/preferences with:
   ```json
   {
     "privacy": {
       "profileVisibility": "private",
       "showEmail": false,
       "showLastActive": false
     }
   }
   ```

**Expected Result**:

- 200 OK
- Profile visibility set to private
- Profile hidden from public search/directory
- Email and lastActive hidden

---

#### TC-UM-018: Change Theme ✅

**Preconditions**: User has theme: "light"
**Steps**:

1. PATCH /users/me/preferences with:
   ```json
   { "theme": "dark" }
   ```

**Expected Result**:

- 200 OK
- preferences.theme = "dark"
- Response includes updated preferences

---

#### TC-UM-019: Invalid Theme Value ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/preferences with:
   ```json
   { "theme": "blue" }
   ```

**Expected Result**:

- 400 Bad Request
- Error: "Theme must be one of: light, dark, auto"
- theme unchanged

---

#### TC-UM-020: Update Language and Timezone ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/preferences with:
   ```json
   {
     "language": "fr",
     "timezone": "Europe/Paris"
   }
   ```

**Expected Result**:

- 200 OK
- Language set to French
- Timezone set to Europe/Paris
- Future timestamps formatted accordingly

---

#### TC-UM-021: Update Media Preferences ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/preferences with:
   ```json
   {
     "media": {
       "autoOptimize": true,
       "defaultVisibility": "unlisted",
       "watermarkEnabled": true
     }
   }
   ```

**Expected Result**:

- 200 OK
- mediaPreferences updated
- Future uploads use new defaults
- Existing uploads unchanged

---

#### TC-UM-022: Partial Preference Update ✅

**Preconditions**: User has multiple preferences configured
**Steps**:

1. PATCH /users/me/preferences with only:
   ```json
   { "theme": "auto" }
   ```

**Expected Result**:

- 200 OK
- Only theme updated
- notifications, privacy, language, timezone unchanged
- Partial updates supported

---

#### TC-UM-023: Empty Preferences Request ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/preferences with empty body: `{}`

**Expected Result**:

- 200 OK (no changes)
- All preferences unchanged
- Response includes current preferences

---

### Password Change Tests (PATCH /users/me/password)

#### TC-UM-024: Successful Password Change ✅

**Preconditions**: User authenticated, current password known
**Steps**:

1. PATCH /users/me/password with:
   ```json
   {
     "currentPassword": "OldSecurePass123!",
     "newPassword": "NewStrongPass456#"
   }
   ```

**Expected Result**:

- 200 OK
- Message: "Password changed successfully"
- Password updated in database (bcrypt hash)
- security.lastPasswordReset timestamp updated
- All refresh tokens revoked except current
- User remains logged in (current session maintained)
- Audit log records password change

---

#### TC-UM-025: Incorrect Current Password ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/password with wrong currentPassword

**Expected Result**:

- 400 Bad Request
- Error: "Current password is incorrect"
- Password unchanged
- Failed attempt potentially logged (security monitoring)

---

#### TC-UM-026: Weak New Password ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/password with:
   ```json
   {
     "currentPassword": "correct",
     "newPassword": "weak"
   }
   ```

**Expected Result**:

- 400 Bad Request
- Error: "New password does not meet requirements"
- data.errors array contains:
  - "Password must be at least 8 characters"
  - "Password must contain uppercase letter"
  - "Password must contain number"
  - "Password must contain special character"
- Password unchanged

---

#### TC-UM-027: New Password Same as Current ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/password with newPassword identical to currentPassword

**Expected Result**:

- 400 Bad Request
- Error: "New password must be different from current password"
- Password unchanged

---

#### TC-UM-028: Password with All Character Types ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/password with:
   ```json
   {
     "currentPassword": "correct",
     "newPassword": "Abc123!@#Def456$%^"
   }
   ```

**Expected Result**:

- 200 OK
- Password accepted (meets all requirements)
- Contains: uppercase, lowercase, numbers, special chars

---

#### TC-UM-029: Minimum Length Password ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/password with 8-character password: "Abc123!@"

**Expected Result**:

- 200 OK (if meets all other requirements)
- Minimum length satisfied

---

#### TC-UM-030: Password Change Invalidates Other Sessions ✅

**Preconditions**: User logged in on 3 devices (3 refresh tokens)
**Steps**:

1. Device 1: PATCH /users/me/password (successful)
2. Device 2: Attempt to refresh access token
3. Device 3: Attempt to use existing access token

**Expected Result**:

- Device 1: Password change successful, session maintained
- Device 2: Refresh token invalid (401 Unauthorized), must re-login
- Device 3: Existing access tokens valid until expiry, then refresh fails
- refreshTokens array contains only Device 1's token

---

#### TC-UM-031: Rate Limiting on Password Change ✅

**Preconditions**: User authenticated
**Steps**:

1. Make 10 PATCH /users/me/password requests in 60 seconds

**Expected Result**:

- First few requests processed normally
- Subsequent requests: 429 Too Many Requests
- Prevents brute-force password attempts

---

### Account Deletion Tests (DELETE /users/me)

#### TC-UM-032: Successful Account Deletion ✅

**Preconditions**: User authenticated
**Steps**:

1. DELETE /users/me with:
   ```json
   {
     "password": "UserCorrectPassword123!",
     "confirmDeletion": "DELETE_MY_ACCOUNT"
   }
   ```

**Expected Result**:

- 200 OK
- Message: "Account deleted successfully"
- user.isDeleted = true
- user.deletedAt = current timestamp
- All refresh tokens cleared
- accessToken and refreshToken cookies cleared
- User cannot log in subsequently
- Audit log records deletion event
- Data retained for retention period (30-90 days)

---

#### TC-UM-033: Deletion with Incorrect Password ✅

**Preconditions**: User authenticated
**Steps**:

1. DELETE /users/me with wrong password

**Expected Result**:

- 400 Bad Request
- Error: "Password is incorrect"
- Account not deleted
- isDeleted remains false

---

#### TC-UM-034: Deletion Without Confirmation Text ✅

**Preconditions**: User authenticated
**Steps**:

1. DELETE /users/me with:
   ```json
   {
     "password": "correct",
     "confirmDeletion": "YES"
   }
   ```

**Expected Result**:

- 400 Bad Request
- Error: "Account deletion not confirmed"
- data.message: "Please provide confirmDeletion: \"DELETE_MY_ACCOUNT\" to confirm account deletion"
- Account not deleted

---

#### TC-UM-035: Deletion with Case-Sensitive Confirmation ✅

**Preconditions**: User authenticated
**Steps**:

1. DELETE /users/me with:
   ```json
   {
     "password": "correct",
     "confirmDeletion": "delete_my_account"
   }
   ```

**Expected Result**:

- 400 Bad Request
- Confirmation is case-sensitive
- "delete_my_account" ≠ "DELETE_MY_ACCOUNT"
- Account not deleted

---

#### TC-UM-036: Deleted Account Login Attempt ✅

**Preconditions**: Account deleted (isDeleted: true)
**Steps**:

1. POST /api/auth/login with deleted user's credentials

**Expected Result**:

- 401 Unauthorized OR 404 Not Found
- Error: "Account not found" or "Invalid credentials"
- Login rejected

---

#### TC-UM-037: Deleted Account Token Usage ✅

**Preconditions**: User deleted account but still has valid JWT token
**Steps**:

1. GET /users/me with pre-deletion token

**Expected Result**:

- 401 Unauthorized
- Authentication middleware checks isDeleted flag
- Rejects deleted accounts

---

#### TC-UM-038: Data Retention After Deletion ✅

**Preconditions**: Account deleted
**Steps**:

1. Check database for user record

**Expected Result**:

- User record exists in database
- isDeleted: true
- deletedAt: timestamp
- All user data preserved (for audit/legal)
- Record excluded from normal queries (filter: isDeleted ≠ true)

---

### Media Quota Tests (GET /users/me/media-quota)

#### TC-UM-039: Retrieve Quota with Available Space ✅

**Preconditions**: User authenticated, emailVerified: true, used < limit
**Steps**:

1. GET /users/me/media-quota

**Expected Result**:

- 200 OK
- quota object includes:
  - used: actual bytes consumed
  - limit: total bytes allowed
  - fileCount: number of files
  - maxFiles: file limit
  - usedPercentage: (used/limit) × 100
  - fileCountPercentage: (fileCount/maxFiles) × 100
  - canUpload: true
- preferences included

---

#### TC-UM-040: Quota at 100% Capacity ✅

**Preconditions**: User has used = limit (quota full)
**Steps**:

1. GET /users/me/media-quota

**Expected Result**:

- 200 OK
- usedPercentage: 100.00
- canUpload: false
- User cannot upload additional files

---

#### TC-UM-041: Quota Below Threshold ✅

**Preconditions**: User has used = 5% of limit
**Steps**:

1. GET /users/me/media-quota

**Expected Result**:

- 200 OK
- usedPercentage: 5.00
- canUpload: true
- Ample space available

---

#### TC-UM-042: File Count Limit Reached ✅

**Preconditions**: fileCount = maxFiles, but storage space available
**Steps**:

1. GET /users/me/media-quota

**Expected Result**:

- 200 OK
- fileCountPercentage: 100.00
- canUpload: false (file count limit reached)
- Even if storage space available, file limit prevents uploads

---

#### TC-UM-043: Unverified Email Access ✅

**Preconditions**: User authenticated but emailVerified: false
**Steps**:

1. GET /users/me/media-quota

**Expected Result**:

- 403 Forbidden
- Error: "Email verification required"
- requireEmailVerification middleware blocks request

---

#### TC-UM-044: Quota Percentage Calculation Accuracy ✅

**Preconditions**: User has specific quota values
**Steps**:

1. User has: used = 536870912 (512 MB), limit = 1073741824 (1 GB)
2. GET /users/me/media-quota

**Expected Result**:

- usedPercentage: 50.00
- Calculation: (536870912 / 1073741824) × 100 = 50.00
- Rounded to 2 decimal places

---

#### TC-UM-045: Quota After File Upload ✅

**Preconditions**: User uploads 100 MB file
**Steps**:

1. GET /users/me/media-quota (before upload)
2. Upload file (100 MB)
3. GET /users/me/media-quota (after upload)

**Expected Result**:

- Before: used = X, fileCount = Y
- After: used = X + 100MB, fileCount = Y + 1
- usedPercentage increased
- Quota updates reflect upload immediately

---

#### TC-UM-046: Quota After File Deletion ✅

**Preconditions**: User deletes 50 MB file
**Steps**:

1. GET /users/me/media-quota (before deletion)
2. Delete file (50 MB)
3. GET /users/me/media-quota (after deletion)

**Expected Result**:

- After deletion: used decreased by 50 MB
- fileCount decreased by 1
- Quota freed up
- canUpload may change from false → true

---

### Edge Cases and Error Handling

#### TC-UM-047: Concurrent Profile Updates ✅

**Preconditions**: User authenticated on 2 devices
**Steps**:

1. Device 1: PATCH /users/me with firstName: "Alice"
2. Device 2 (simultaneously): PATCH /users/me with firstName: "Alicia"

**Expected Result**:

- Both requests processed
- Last write wins (Alicia or Alice depending on timing)
- No data corruption
- Optimistic locking or versioning may be used

---

#### TC-UM-048: Empty String vs Null for Optional Fields ✅

**Preconditions**: User has bio: "Old bio"
**Steps**:

1. PATCH /users/me with:
   ```json
   { "bio": null }
   ```

**Expected Result**:

- bio set to null or empty string (depending on schema)
- Field cleared successfully
- Consistent behavior across all optional fields

---

#### TC-UM-049: SQL Injection in Profile Fields ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me with:
   ```json
   { "bio": "'; DROP TABLE users; --" }
   ```

**Expected Result**:

- 200 OK
- bio set to literal string "'; DROP TABLE users; --"
- No SQL execution (MongoDB uses BSON, not SQL)
- Input sanitized/escaped

---

#### TC-UM-050: XSS in Profile Fields ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me with:
   ```json
   { "bio": "<script>alert('XSS')</script>" }
   ```

**Expected Result**:

- 200 OK
- bio stored as plain text
- When rendered in frontend, HTML escaped
- No script execution

---

#### TC-UM-051: Extremely Long Website URL ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me with website URL exceeding 2048 characters

**Expected Result**:

- 400 Bad Request OR
- URL truncated to maximum length
- Validation error if exceeds limit

---

#### TC-UM-052: Invalid Timezone Identifier ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/preferences with:
   ```json
   { "timezone": "Invalid/Timezone" }
   ```

**Expected Result**:

- 400 Bad Request
- Error: "Invalid timezone identifier"
- timezone unchanged
- (Or accepted if no validation, potentially causing issues)

---

#### TC-UM-053: Password Change with Special Unicode ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me/password with:
   ```json
   {
     "currentPassword": "correct",
     "newPassword": "Pãsswörd123!日本"
   }
   ```

**Expected Result**:

- 200 OK if meets length/strength requirements
- Unicode characters supported
- bcrypt handles UTF-8 properly

---

#### TC-UM-054: Account Deletion During Active Upload ✅

**Preconditions**: User uploading large file (in progress)
**Steps**:

1. Start 500 MB file upload
2. During upload: DELETE /users/me

**Expected Result**:

- Account deleted successfully
- Upload either:
  - Completes (orphaned file, cleanup job handles)
  - Cancelled (upload service checks user status)
- No data corruption

---

#### TC-UM-055: Quota Query for Admin/Super Admin ✅

**Preconditions**: Admin user authenticated
**Steps**:

1. GET /users/me/media-quota

**Expected Result**:

- 200 OK
- Returns admin's personal quota (not system-wide)
- Admins may have different quota limits (higher tier)

---

#### TC-UM-056: Profile Update with Null Values ✅

**Preconditions**: User authenticated
**Steps**:

1. PATCH /users/me with:
   ```json
   {
     "displayName": null,
     "bio": null,
     "website": null
   }
   ```

**Expected Result**:

- 200 OK
- Optional fields cleared/set to null
- Required fields (firstName, lastName) cannot be nulled

---

#### TC-UM-057: Multiple Rapid Password Changes ✅

**Preconditions**: User authenticated
**Steps**:

1. Change password to "NewPass1#"
2. Immediately change to "NewPass2#"
3. Immediately change to "NewPass3#"

**Expected Result**:

- All changes succeed (if within rate limit)
- Final password: "NewPass3#"
- Each change revokes previous refresh tokens
- security.lastPasswordReset updated with latest timestamp

---

### Security Tests

#### TC-UM-SEC-001: JWT Token Tampering ✅

**Steps**:

1. Obtain valid JWT token
2. Modify payload (change user ID)
3. GET /users/me with tampered token

**Expected Result**:

- 401 Unauthorized
- Error: "Invalid token signature"
- Authentication middleware detects tampering

---

#### TC-UM-SEC-002: Expired Token Rejection ✅

**Steps**:

1. Use JWT token expired >1 hour ago
2. GET /users/me

**Expected Result**:

- 401 Unauthorized
- Error: "Token expired"
- Client must refresh token

---

#### TC-UM-SEC-003: Password Brute Force Protection ✅

**Steps**:

1. Make 20 password change requests with wrong currentPassword

**Expected Result**:

- First few attempts: 400 Bad Request
- After threshold: 429 Too Many Requests
- Account potentially locked (security.lockoutUntil set)
- Rate limiting prevents brute force

---

#### TC-UM-SEC-004: Password Complexity Enforcement ✅

**Steps**:

1. Attempt password change with variations:
   - "12345678" (numbers only)
   - "abcdefgh" (lowercase only)
   - "ABCDEFGH" (uppercase only)
   - "Abcdefgh" (no number/special)

**Expected Result**:

- All rejected with 400 Bad Request
- Specific error for each missing requirement

---

#### TC-UM-SEC-005: Session Fixation Prevention ✅

**Steps**:

1. Obtain session token
2. Change password
3. Attempt to use old access token (issued before password change)

**Expected Result**:

- Old access tokens remain valid until expiry (short TTL)
- Old refresh tokens invalidated
- New sessions required after password change

---

#### TC-UM-SEC-006: Account Deletion Audit Trail ✅

**Steps**:

1. Delete account
2. Check audit logs

**Expected Result**:

- Deletion event logged with:
  - User ID
  - Email
  - Timestamp
  - IP address
  - User agent
- Immutable audit record

---

#### TC-UM-SEC-007: CSRF Protection on State-Changing Endpoints ✅

**Steps**:

1. Attempt PATCH /users/me without CSRF token (if implemented)

**Expected Result**:

- 403 Forbidden (if CSRF protection enabled)
- OR request succeeds (if using JWT-only auth)

---

#### TC-UM-SEC-008: HTTP vs HTTPS Enforcement ✅

**Steps**:

1. Attempt HTTP request to http://api.safeguardmedia.com/api/users/me

**Expected Result**:

- 301 Redirect to HTTPS OR
- Connection refused (HTTP disabled)
- Sensitive data only transmitted over HTTPS

---

### Performance Tests

#### TC-UM-PERF-001: Profile Retrieval Response Time ✅

**Steps**:

1. GET /users/me
2. Measure response time

**Expected Result**:

- Response time: < 200ms (95th percentile)
- Average: 50-100ms
- Single database query

---

#### TC-UM-PERF-002: Profile Update Response Time ✅

**Steps**:

1. PATCH /users/me with 3 field updates
2. Measure response time

**Expected Result**:

- Response time: < 300ms
- Single database write
- No unnecessary queries

---

#### TC-UM-PERF-003: Password Change Response Time ✅

**Steps**:

1. PATCH /users/me/password
2. Measure response time

**Expected Result**:

- Response time: < 500ms
- bcrypt hashing may take 200-300ms (intentional slowness)
- Acceptable trade-off for security

---

#### TC-UM-PERF-004: Concurrent User Requests ✅

**Steps**:

1. Simulate 100 concurrent GET /users/me requests from different users
2. Measure throughput and response times

**Expected Result**:

- All requests complete successfully
- Average response time: < 300ms
- No timeouts
- Server handles concurrent load

---

#### TC-UM-PERF-005: Quota Calculation Performance ✅

**Steps**:

1. GET /users/me/media-quota
2. Measure calculation time for percentages

**Expected Result**:

- Response time: < 200ms
- Percentage calculations in-memory (not database aggregations)
- Efficient computation

---

#### TC-UM-PERF-006: Large Preferences Object ✅

**Steps**:

1. User has complex preferences with many custom settings
2. PATCH /users/me/preferences with large update

**Expected Result**:

- Response time: < 400ms
- Handles large JSON objects efficiently
- No performance degradation

---

#### TC-UM-PERF-007: Database Index Utilization ✅

**Steps**:

1. Execute GET /users/me
2. Check database query plan

**Expected Result**:

- Query uses index on user ID
- No full collection scan
- Indexed lookups for fast retrieval

---

## Conclusion

This test plan covers **70 comprehensive test cases** for the User Management feature:

- **Profile Retrieval**: 5 tests
- **Profile Update**: 10 tests
- **Preferences Update**: 8 tests
- **Password Change**: 8 tests
- **Account Deletion**: 7 tests
- **Media Quota**: 8 tests
- **Edge Cases**: 11 tests
- **Security**: 8 tests
- **Performance**: 7 tests
