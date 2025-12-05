# Feature 15: Admin Features

## Overview

The **Admin Features** provide administrators with comprehensive user management, system monitoring, and administrative control capabilities. This powerful toolset enables admins to manage user accounts, monitor platform health, enforce policies, and maintain system integrity through role-based access control.

### What are Admin Features?

Admin Features encompass all administrative operations for platform governance:

1. **User Administration**: Search, filter, and view all user accounts with detailed information
2. **User Statistics**: Monitor user growth, activity, account statuses, and role distribution
3. **Account Status Management**: Activate, suspend, or ban user accounts with reason tracking
4. **Role Management**: Assign and modify user roles (USER, MODERATOR, ADMIN, SUPER_ADMIN)
5. **Security Controls**: Enforce hierarchical permissions with SUPER_ADMIN protection

This feature empowers administrators to:

- **Monitor User Base**: Track total users, active users, growth trends, and demographics
- **Enforce Policies**: Suspend or ban users violating terms of service
- **Manage Permissions**: Grant or revoke elevated privileges (moderator, admin roles)
- **Maintain Security**: Protect critical accounts (SUPER_ADMIN) from unauthorized modification
- **Audit Activity**: Track administrative actions and status changes
- **Search and Filter**: Quickly locate users by email, name, role, or account status

### Why are Admin Features Important?

- **Platform Governance**: Centralized control over user accounts and permissions
- **Policy Enforcement**: Tools to handle violations, abuse, and terms of service breaches
- **Security Management**: Role-based access control prevents privilege escalation
- **Operational Efficiency**: Bulk user management and filtering reduces administrative overhead
- **Compliance**: Audit trails for administrative actions meet regulatory requirements
- **Data Visibility**: Statistics provide insights for business decisions and growth tracking
- **Incident Response**: Quick account suspension/ban capabilities for security incidents

---

## Real-World Test Scenarios

### Scenario 1: Admin Searches for Suspended Users ✅

**Context**: Admin needs to review all suspended accounts for monthly audit.

**Test Flow**:

1. Admin logs in (JWT obtained)
2. GET /admin/users?status=suspended&page=1&limit=50
3. System filters users with accountStatus = SUSPENDED
4. Returns paginated list of 50 suspended users
5. Admin reviews each case:
   - Checks suspension reasons in audit logs
   - Determines if suspension should be lifted
   - Updates status back to "active" for resolved cases

**Expected Result**:

- All suspended users returned
- Pagination works correctly (50 per page)
- Each user shows accountStatus: "suspended"
- Admin can view details via GET /admin/users/:id

---

### Scenario 2: Promote User to Moderator Role ✅

**Context**: Active community member promoted to moderator for content moderation duties.

**Test Flow**:

1. Admin identifies user: "user-abc123" (currently role: "user")
2. GET /admin/users/user-abc123 to verify current role
3. PATCH /admin/users/user-abc123/role with:
   ```json
   { "role": "moderator" }
   ```
4. System validates:
   - Admin has permission (ADMIN role)
   - Target is not SUPER_ADMIN
   - New role is valid enum value
5. System updates user.role = "moderator"
6. Audit log records promotion
7. User now has access to moderation endpoints
8. User receives email notification of role change

**Expected Result**:

- Role updated from "user" to "moderator"
- User gains moderator permissions immediately
- Audit trail records: adminId, targetUserId, oldRole, newRole
- No session disruption (user remains logged in)

---

### Scenario 3: Admin Suspends Spammer Account ✅

**Context**: User reported for spamming other users, admin investigates and suspends account.

**Test Flow**:

1. Admin receives abuse report for "user-spam456"
2. GET /admin/users/user-spam456 to review activity
3. Examines:
   - Recent uploads (media spam)
   - Account age (created 2 days ago)
   - Email verified (false)
   - lastActiveAt (frequent activity)
4. Decision: Suspend account
5. PATCH /admin/users/user-spam456/status:
   ```json
   {
     "status": "suspended",
     "reason": "Spamming other users with unsolicited messages"
   }
   ```
6. System executes:
   - user.accountStatus = "suspended"
   - user.refreshTokens = [] (clears all tokens)
   - Audit log records suspension with reason
7. User's active sessions invalidated
8. User cannot log in (authentication rejects suspended accounts)

**Expected Result**:

- Account suspended immediately
- User logged out of all devices
- Audit log shows suspension reason
- User receives email notification of suspension
- Admin can reverse suspension later if needed

---

### Scenario 4: Regular Admin Attempts to Modify SUPER_ADMIN ✅

**Context**: Regular admin tries to change super admin's status (should be blocked).

**Test Flow**:

1. Admin (role: "admin") logged in
2. Identifies SUPER_ADMIN user: "super-admin-001"
3. Attempts PATCH /admin/users/super-admin-001/status:
   ```json
   { "status": "suspended" }
   ```
4. System checks protection rule:
   - target.role = "super_admin"
   - admin.role = "admin" (not super_admin)
   - Condition met: REJECT
5. Returns 403 Forbidden:
   ```json
   {
     "success": false,
     "message": "Cannot modify super admin accounts"
   }
   ```
6. No changes made to SUPER_ADMIN account
7. Attempt logged in security audit trail

**Expected Result**:

- Request rejected with 403 Forbidden
- SUPER_ADMIN account unchanged
- Security alert potentially triggered
- Demonstrates protection hierarchy working correctly

---

### Scenario 5: Generate User Statistics for Monthly Report ✅

**Context**: Admin generates monthly user growth and status report.

**Test Flow**:

1. Admin navigates to dashboard
2. GET /admin/users/stats
3. System executes parallel queries:
   - Count total users (1547)
   - Count active users (1402)
   - Count pending verification (87)
   - Count suspended (23)
   - Count banned (15)
   - Count verified emails (1420)
   - Count recent signups (last 30 days: 142)
   - Aggregate users by role:
     - user: 1503
     - moderator: 32
     - admin: 10
     - super_admin: 2
4. Dashboard displays:
   - "User Growth: +142 this month (+10.1%)"
   - "Active Users: 90.6% (1402/1547)"
   - "Email Verified: 91.8% (1420/1547)"
   - "Moderation Needed: 38 accounts (suspended/banned)"
   - Role distribution pie chart

**Expected Result**:

- All statistics accurate
- Percentages calculated correctly
- Provides actionable insights:
  - High pending verification rate (87) → email delivery issue?
  - Suspended/banned accounts (38) → review monthly
  - Strong growth (+142 users)

---

### Scenario 6: Bulk User Search by Email Domain ✅

**Context**: Admin needs to find all users from specific company domain for enterprise inquiry.

**Test Flow**:

1. Admin searches for users from "techcorp.com"
2. GET /admin/users?search=techcorp.com&limit=100
3. System performs case-insensitive regex search:
   - Searches email field: /techcorp.com/i
4. Returns all users with @techcorp.com emails:
   - alice@techcorp.com
   - bob@techcorp.com
   - charlie@techcorp.com
   - (15 total results)
5. Admin reviews list:
   - All active accounts
   - Most verified
   - Created over last 6 months
6. Admin contacts enterprise sales for potential B2B deal

**Expected Result**:

- Search finds all matching email addresses
- Case-insensitive matching works
- Results paginated if > 100 users
- Enables business development opportunities

---

## Test Cases

### User Listing Tests (GET /admin/users)

#### TC-ADMIN-001: List All Users with Default Pagination ✅

**Preconditions**: Admin authenticated, database has >20 users
**Steps**:

1. GET /api/admin/users (no query parameters)

**Expected Result**:

- 200 OK
- Returns first 20 users (default limit)
- Sorted by createdAt descending (newest first)
- pagination object shows:
  - page: 1
  - limit: 20
  - total: actual count
  - totalPages: ceil(total / 20)
  - hasNextPage: true (if total > 20)
  - hasPrevPage: false
- Excludes sensitive fields (password, tokens)

---

#### TC-ADMIN-002: Paginate Through Users ✅

**Preconditions**: Database has 100 users
**Steps**:

1. GET /admin/users?page=1&limit=25 (page 1)
2. GET /admin/users?page=2&limit=25 (page 2)
3. GET /admin/users?page=3&limit=25 (page 3)
4. GET /admin/users?page=4&limit=25 (page 4)

**Expected Result**:

- Page 1: Users 1-25, hasNextPage: true, hasPrevPage: false
- Page 2: Users 26-50, hasNextPage: true, hasPrevPage: true
- Page 3: Users 51-75, hasNextPage: true, hasPrevPage: true
- Page 4: Users 76-100, hasNextPage: false, hasPrevPage: true
- Total pages: 4
- No duplicate users across pages

---

#### TC-ADMIN-003: Filter Users by Role ✅

**Preconditions**: Database has users with various roles
**Steps**:

1. GET /admin/users?role=moderator

**Expected Result**:

- Returns only users with role = "moderator"
- All other roles excluded
- Pagination works normally
- Count matches actual moderator users

---

#### TC-ADMIN-004: Filter Users by Account Status ✅

**Preconditions**: Database has users with various statuses
**Steps**:

1. GET /admin/users?status=suspended

**Expected Result**:

- Returns only users with accountStatus = "suspended"
- Active, banned, pending users excluded
- Useful for reviewing suspended accounts

---

#### TC-ADMIN-005: Search Users by Name ✅

**Preconditions**: Database has user "Alice Johnson"
**Steps**:

1. GET /admin/users?search=alice

**Expected Result**:

- Returns users matching "alice" in:
  - firstName (Alice)
  - lastName
  - email (alice@...)
  - username (alice\_...)
- Case-insensitive search
- Partial matching works

---

#### TC-ADMIN-006: Search Users by Email ✅

**Preconditions**: User exists with email "john@example.com"
**Steps**:

1. GET /admin/users?search=john@example.com

**Expected Result**:

- Returns user with exact email match
- Also returns users with "john" in firstName/lastName/username
- Regex search: /john@example.com/i

---

#### TC-ADMIN-007: Sort Users by Email ✅

**Preconditions**: Multiple users exist
**Steps**:

1. GET /admin/users?sort=email&order=asc

**Expected Result**:

- Users sorted alphabetically by email A→Z
- alice@... comes before zack@...

---

#### TC-ADMIN-008: Sort Users by Last Login ✅

**Preconditions**: Users with various lastLoginAt timestamps
**Steps**:

1. GET /admin/users?sort=lastLoginAt&order=desc

**Expected Result**:

- Users sorted by most recent login first
- Users who never logged in (lastLoginAt: null) appear last

---

#### TC-ADMIN-009: Combine Multiple Filters ✅

**Preconditions**: Database with diverse user base
**Steps**:

1. GET /admin/users?role=user&status=active&emailVerified=true&limit=50

**Expected Result**:

- Returns users matching ALL criteria:
  - role = "user"
  - accountStatus = "active"
  - emailVerified = true
- Limit: 50 results per page
- AND logic applied to filters

---

#### TC-ADMIN-010: Exceed Maximum Limit ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /admin/users?limit=500

**Expected Result**:

- 400 Bad Request OR
- Limit capped at 100 (maximum allowed)
- Error: "Limit cannot exceed 100"

---

#### TC-ADMIN-011: Invalid Page Number ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /admin/users?page=0
2. GET /admin/users?page=-1

**Expected Result**:

- 400 Bad Request
- Error: "Page must be greater than 0"

---

#### TC-ADMIN-012: Non-Admin Access Attempt ✅

**Preconditions**: Regular user (role: "user") authenticated
**Steps**:

1. GET /admin/users

**Expected Result**:

- 403 Forbidden
- Error: "Insufficient permissions" or "Admin access required"
- requireRole middleware blocks request

---

#### TC-ADMIN-013: Moderator Access Attempt ✅

**Preconditions**: Moderator (role: "moderator") authenticated
**Steps**:

1. GET /admin/users

**Expected Result**:

- 403 Forbidden
- Moderators do not have admin endpoint access
- Only ADMIN and SUPER_ADMIN roles allowed

---

#### TC-ADMIN-014: Empty Search Results ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /admin/users?search=nonexistent_user_xyz

**Expected Result**:

- 200 OK
- users: [] (empty array)
- pagination.total: 0
- Message: "Users retrieved successfully" (no error, just no results)

---

#### TC-ADMIN-015: Filter by Unverified Emails ✅

**Preconditions**: Database has unverified users
**Steps**:

1. GET /admin/users?emailVerified=false

**Expected Result**:

- Returns only users with emailVerified = false
- Useful for identifying accounts needing email verification
- Admin can resend verification emails

---

### User Statistics Tests (GET /admin/users/stats)

#### TC-ADMIN-016: Retrieve User Statistics ✅

**Preconditions**: Admin authenticated, diverse user base
**Steps**:

1. GET /api/admin/users/stats

**Expected Result**:

- 200 OK
- stats object includes:
  - total: all non-deleted users
  - active: accountStatus = ACTIVE count
  - pending: PENDING_VERIFICATION count
  - suspended: SUSPENDED count
  - banned: BANNED count
  - verified: emailVerified = true count
  - recent: created in last 30 days count
  - inactive: total - active
  - roles: {user: X, moderator: Y, admin: Z, super_admin: W}
- All counts accurate
- Response time: < 500ms

---

#### TC-ADMIN-017: Statistics Exclude Deleted Users ✅

**Preconditions**: Database has soft-deleted users (isDeleted: true)
**Steps**:

1. GET /admin/users/stats

**Expected Result**:

- All statistics exclude isDeleted = true users
- total count only includes active database records
- Deleted users do not inflate counts

---

#### TC-ADMIN-018: Role Distribution Accuracy ✅

**Preconditions**: Known role distribution (e.g., 100 users, 10 moderators, 2 admins)
**Steps**:

1. GET /admin/users/stats
2. Verify stats.roles matches expected distribution

**Expected Result**:

- roles.user: 100
- roles.moderator: 10
- roles.admin: 2
- roles.super_admin: 1
- Sum of all roles = total users

---

#### TC-ADMIN-019: Recent Users Calculation ✅

**Preconditions**: 5 users created in last 30 days, 10 created 35 days ago
**Steps**:

1. GET /admin/users/stats

**Expected Result**:

- stats.recent: 5
- Only users where: createdAt >= (now - 30 days)
- 35-day-old users excluded

---

#### TC-ADMIN-020: Inactive Calculation ✅

**Preconditions**: total = 100, active = 85
**Steps**:

1. GET /admin/users/stats

**Expected Result**:

- stats.inactive: 15 (100 - 85)
- Includes: pending, suspended, banned, inactive statuses

---

### Single User Retrieval Tests (GET /admin/users/:id)

#### TC-ADMIN-021: Retrieve User by ID ✅

**Preconditions**: Admin authenticated, user "user-abc123" exists
**Steps**:

1. GET /api/admin/users/user-abc123

**Expected Result**:

- 200 OK
- Complete user object returned
- Includes: profile, role, accountStatus, mediaQuota, subscription, preferences, security (non-sensitive)
- Excludes: password, refreshTokens, emailVerificationToken, passwordResetToken

---

#### TC-ADMIN-022: Non-Existent User ID ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /admin/users/nonexistent-user-id-999

**Expected Result**:

- 404 Not Found
- Error: "User not found"

---

#### TC-ADMIN-023: Invalid User ID Format ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /admin/users/invalid-id-format

**Expected Result**:

- 400 Bad Request
- Validation error: "Invalid user ID format"

---

#### TC-ADMIN-024: Retrieve Deleted User ✅

**Preconditions**: User soft-deleted (isDeleted: true)
**Steps**:

1. GET /admin/users/:deletedUserId

**Expected Result**:

- 200 OK (user still exists in database) OR
- 404 Not Found (if query filters exclude deleted users)
- If returned, isDeleted: true visible to admin

---

### Account Status Management Tests (PATCH /admin/users/:id/status)

#### TC-ADMIN-025: Suspend User Account ✅

**Preconditions**: Admin authenticated, target user active
**Steps**:

1. PATCH /admin/users/user-abc123/status:
   ```json
   {
     "status": "suspended",
     "reason": "Terms of Service violation"
   }
   ```

**Expected Result**:

- 200 OK
- user.accountStatus = "suspended"
- user.refreshTokens = [] (all tokens revoked)
- Audit log records: adminId, targetUserId, oldStatus, newStatus, reason, timestamp
- User logged out of all devices
- User cannot log in until reactivated

---

#### TC-ADMIN-026: Ban User Account ✅

**Preconditions**: Admin authenticated, target user active
**Steps**:

1. PATCH /admin/users/user-abc123/status:
   ```json
   { "status": "banned" }
   ```

**Expected Result**:

- 200 OK
- user.accountStatus = "banned"
- All refresh tokens revoked
- Permanent restriction (unless admin reverses)

---

#### TC-ADMIN-027: Reactivate Suspended User ✅

**Preconditions**: User currently suspended
**Steps**:

1. PATCH /admin/users/user-abc123/status:
   ```json
   {
     "status": "active",
     "reason": "Appeal approved, suspension lifted"
   }
   ```

**Expected Result**:

- 200 OK
- user.accountStatus = "active"
- User can log in again
- User must obtain new refresh token (old ones were revoked)

---

#### TC-ADMIN-028: Invalid Status Value ✅

**Preconditions**: Admin authenticated
**Steps**:

1. PATCH /admin/users/user-abc123/status:
   ```json
   { "status": "unknown_status" }
   ```

**Expected Result**:

- 400 Bad Request
- Error: "Invalid account status"
- user.accountStatus unchanged

---

#### TC-ADMIN-029: Admin Attempts to Modify SUPER_ADMIN Status ✅

**Preconditions**: Regular admin (not super_admin), target is super_admin
**Steps**:

1. PATCH /admin/users/super-admin-001/status:
   ```json
   { "status": "suspended" }
   ```

**Expected Result**:

- 403 Forbidden
- Error: "Cannot modify super admin accounts"
- SUPER_ADMIN status unchanged
- Protection rule enforced

---

#### TC-ADMIN-030: SUPER_ADMIN Modifies Another SUPER_ADMIN ✅

**Preconditions**: SUPER_ADMIN authenticated, target is another SUPER_ADMIN
**Steps**:

1. PATCH /admin/users/super-admin-002/status:
   ```json
   { "status": "suspended" }
   ```

**Expected Result**:

- 200 OK
- Status updated successfully
- SUPER_ADMIN has permission to modify other SUPER_ADMINs

---

#### TC-ADMIN-031: Status Change Audit Log ✅

**Preconditions**: Admin suspends user
**Steps**:

1. PATCH /admin/users/user-abc123/status (status: "suspended")
2. Check application logs

**Expected Result**:

- Audit log entry created:
  - adminId: ID of admin who made change
  - targetUserId: ID of affected user
  - oldStatus: "active"
  - newStatus: "suspended"
  - reason: (if provided)
  - timestamp: ISO 8601 timestamp
- Immutable log record

---

#### TC-ADMIN-032: Suspend User with Active Upload ✅

**Preconditions**: User uploading file, admin suspends during upload
**Steps**:

1. User starts 500 MB file upload
2. Admin: PATCH /admin/users/:id/status (suspended)
3. Upload continues

**Expected Result**:

- Account suspended immediately
- Upload either:
  - Completes (file uploaded)
  - Cancelled (depending on implementation)
- Refresh tokens revoked
- User cannot start new uploads

---

### Role Management Tests (PATCH /admin/users/:id/role)

#### TC-ADMIN-033: Promote User to Moderator ✅

**Preconditions**: Admin authenticated, target role: "user"
**Steps**:

1. PATCH /admin/users/user-abc123/role:
   ```json
   { "role": "moderator" }
   ```

**Expected Result**:

- 200 OK
- user.role = "moderator"
- User gains moderator permissions immediately
- Audit log records: adminId, targetUserId, oldRole, newRole
- User remains logged in (no session disruption)

---

#### TC-ADMIN-034: Promote Moderator to Admin ✅

**Preconditions**: Admin authenticated, target role: "moderator"
**Steps**:

1. PATCH /admin/users/user-abc123/role:
   ```json
   { "role": "admin" }
   ```

**Expected Result**:

- 200 OK
- user.role = "admin"
- User gains admin permissions

---

#### TC-ADMIN-035: Demote Admin to Moderator ✅

**Preconditions**: Admin authenticated, target role: "admin"
**Steps**:

1. PATCH /admin/users/user-abc123/role:
   ```json
   { "role": "moderator" }
   ```

**Expected Result**:

- 200 OK
- user.role = "moderator"
- User loses admin permissions immediately
- Access to /admin/\* routes revoked

---

#### TC-ADMIN-036: Invalid Role Value ✅

**Preconditions**: Admin authenticated
**Steps**:

1. PATCH /admin/users/user-abc123/role:
   ```json
   { "role": "invalid_role" }
   ```

**Expected Result**:

- 400 Bad Request
- Error: "Invalid user role"
- user.role unchanged

---

#### TC-ADMIN-037: Regular Admin Attempts to Grant SUPER_ADMIN ✅

**Preconditions**: Regular admin (not super_admin)
**Steps**:

1. PATCH /admin/users/user-abc123/role:
   ```json
   { "role": "super_admin" }
   ```

**Expected Result**:

- 403 Forbidden
- Error: "Only super admins can assign super admin role"
- user.role unchanged (remains "user")
- Protection rule enforced

---

#### TC-ADMIN-038: SUPER_ADMIN Grants SUPER_ADMIN Role ✅

**Preconditions**: SUPER_ADMIN authenticated
**Steps**:

1. PATCH /admin/users/user-abc123/role:
   ```json
   { "role": "super_admin" }
   ```

**Expected Result**:

- 200 OK
- user.role = "super_admin"
- User gains full system access
- Only SUPER_ADMIN can perform this operation

---

#### TC-ADMIN-039: Regular Admin Attempts to Modify SUPER_ADMIN Role ✅

**Preconditions**: Regular admin, target is super_admin
**Steps**:

1. PATCH /admin/users/super-admin-001/role:
   ```json
   { "role": "admin" }
   ```

**Expected Result**:

- 403 Forbidden
- Error: "Cannot modify super admin accounts"
- SUPER_ADMIN role unchanged

---

#### TC-ADMIN-040: Demote Self (Admin Reduces Own Role) ✅

**Preconditions**: Admin authenticated
**Steps**:

1. Admin: PATCH /admin/users/{own_user_id}/role:
   ```json
   { "role": "user" }
   ```

**Expected Result**:

- 200 OK (if no self-modification prevention)
- user.role = "user"
- Admin immediately loses admin access
- Cannot access /admin/\* routes
- Warning: Potentially dangerous, may want to prevent

---

### Edge Cases and Error Handling

#### TC-ADMIN-041: Concurrent Status Updates ✅

**Preconditions**: 2 admins update same user's status simultaneously
**Steps**:

1. Admin A: PATCH /admin/users/:id/status (status: "suspended")
2. Admin B (simultaneously): PATCH /admin/users/:id/status (status: "active")

**Expected Result**:

- Both requests processed
- Last write wins (either suspended or active depending on timing)
- Both changes logged in audit trail
- No data corruption

---

#### TC-ADMIN-042: Extremely Long Search Query ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /admin/users?search={5000-character string}

**Expected Result**:

- 400 Bad Request OR
- Search string truncated
- Server remains stable

---

#### TC-ADMIN-043: SQL Injection in Search ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /admin/users?search='; DROP TABLE users; --

**Expected Result**:

- 200 OK
- Search treats input as literal string
- No SQL execution (MongoDB uses BSON)
- No security impact

---

#### TC-ADMIN-044: NoSQL Injection in Filters ✅

**Preconditions**: Admin authenticated
**Steps**:

1. GET /admin/users?role[$ne]=null

**Expected Result**:

- 400 Bad Request
- Validation rejects non-string role values
- Query parameter sanitization prevents injection

---

#### TC-ADMIN-045: Pagination Beyond Total Pages ✅

**Preconditions**: Database has 50 users (2 pages at limit=25)
**Steps**:

1. GET /admin/users?page=10&limit=25

**Expected Result**:

- 200 OK
- users: [] (empty array)
- pagination.hasNextPage: false
- pagination.hasPrevPage: true
- No error, just no results

---

#### TC-ADMIN-046: Status Change on Deleted User ✅

**Preconditions**: User soft-deleted (isDeleted: true)
**Steps**:

1. PATCH /admin/users/:deletedUserId/status:
   ```json
   { "status": "active" }
   ```

**Expected Result**:

- 404 Not Found (if query filters exclude deleted) OR
- 200 OK with status updated (admin can reactivate deleted accounts)
- Depends on business logic

---

#### TC-ADMIN-047: Role Change Without Role Field ✅

**Preconditions**: Admin authenticated
**Steps**:

1. PATCH /admin/users/user-abc123/role:
   ```json
   {}
   ```

**Expected Result**:

- 400 Bad Request
- Validation error: "role field is required"

---

#### TC-ADMIN-048: Status Change Without Status Field ✅

**Preconditions**: Admin authenticated
**Steps**:

1. PATCH /admin/users/user-abc123/status:
   ```json
   {}
   ```

**Expected Result**:

- 400 Bad Request
- Validation error: "status field is required"

---

#### TC-ADMIN-049: Search with Special Characters ✅

**Preconditions**: User has name "O'Brien"
**Steps**:

1. GET /admin/users?search=O'Brien

**Expected Result**:

- 200 OK
- Returns user with name "O'Brien"
- Special characters (apostrophe) handled correctly
- Regex escaping applied

---

#### TC-ADMIN-050: Retrieve User with No Quota Data ✅

**Preconditions**: New user with uninitialized mediaQuota
**Steps**:

1. GET /admin/users/new-user-id

**Expected Result**:

- 200 OK
- mediaQuota shows defaults:
  - used: 0
  - limit: default tier limit
  - fileCount: 0
  - maxFiles: default tier max

---

### Security Tests

#### TC-ADMIN-SEC-001: Non-Admin User Access ✅

**Steps**:

1. Regular user (role: "user") attempts GET /admin/users

**Expected Result**:

- 403 Forbidden
- Error: "Admin access required" or "Insufficient permissions"
- requireRole(UserRole.ADMIN) middleware blocks request

---

#### TC-ADMIN-SEC-002: Moderator Access to Admin Endpoints ✅

**Steps**:

1. Moderator (role: "moderator") attempts GET /admin/users

**Expected Result**:

- 403 Forbidden
- Moderators cannot access admin endpoints
- Only ADMIN and SUPER_ADMIN allowed

---

#### TC-ADMIN-SEC-003: Unauthenticated Access ✅

**Steps**:

1. GET /admin/users without JWT token

**Expected Result**:

- 401 Unauthorized
- authenticate middleware blocks before requireRole check

---

#### TC-ADMIN-SEC-004: Expired Admin Token ✅

**Steps**:

1. GET /admin/users with expired JWT token

**Expected Result**:

- 401 Unauthorized
- Error: "Token expired"
- Client must refresh token

---

#### TC-ADMIN-SEC-005: Privilege Escalation Attempt ✅

**Steps**:

1. Regular admin attempts to grant self super_admin role
2. PATCH /admin/users/{own_id}/role:
   ```json
   { "role": "super_admin" }
   ```

**Expected Result**:

- 403 Forbidden
- Error: "Only super admins can assign super admin role"
- Privilege escalation blocked

---

#### TC-ADMIN-SEC-006: Password Field Exposure ✅

**Steps**:

1. Admin: GET /admin/users/:id
2. Inspect response for password field

**Expected Result**:

- password field NOT present in response
- .select('-password') excludes field
- Password hash never exposed via API

---

#### TC-ADMIN-SEC-007: Refresh Token Exposure ✅

**Steps**:

1. Admin: GET /admin/users/:id
2. Inspect response for refreshTokens field

**Expected Result**:

- refreshTokens field NOT present
- Security tokens never exposed
- .select('-refreshTokens') applied

---

#### TC-ADMIN-SEC-008: Audit Log Immutability ✅

**Steps**:

1. Admin updates user status
2. Attempt to modify audit log entry in database

**Expected Result**:

- Audit logs stored in append-only manner
- No API endpoint to modify/delete audit logs
- Tamper-evident logging

---

#### TC-ADMIN-SEC-009: Rate Limiting on Admin Endpoints ✅

**Steps**:

1. Make 1000 requests to /admin/users in 60 seconds

**Expected Result**:

- First N requests: 200 OK
- Subsequent requests: 429 Too Many Requests
- Rate limiting prevents abuse even for admins

---

#### TC-ADMIN-SEC-010: SUPER_ADMIN Protection Bypass Attempt ✅

**Steps**:

1. Regular admin modifies request to bypass protection:
   - Tamper with JWT payload
   - Modify user_id in request body
2. Attempt to modify SUPER_ADMIN

**Expected Result**:

- Protection still enforced server-side
- JWT signature validation prevents tampering
- Authorization checks against verified token
- Attack fails

---

### Performance Tests

#### TC-ADMIN-PERF-001: User Listing Response Time ✅

**Steps**:

1. GET /admin/users?limit=100
2. Measure response time

**Expected Result**:

- Response time: < 500ms (95th percentile)
- Database query indexed on commonly filtered fields
- Efficient pagination

---

#### TC-ADMIN-PERF-002: User Search Performance ✅

**Steps**:

1. GET /admin/users?search=john
2. Measure response time

**Expected Result**:

- Response time: < 800ms
- Regex search may be slower than indexed lookups
- Acceptable for admin operations

---

#### TC-ADMIN-PERF-003: Statistics Aggregation Performance ✅

**Steps**:

1. GET /admin/users/stats (database with 10,000 users)
2. Measure response time

**Expected Result**:

- Response time: < 1000ms
- Parallel query execution (Promise.all)
- Indexed queries for counts
- Aggregation pipeline efficient

---

#### TC-ADMIN-PERF-004: Single User Retrieval Performance ✅

**Steps**:

1. GET /admin/users/:id
2. Measure response time

**Expected Result**:

- Response time: < 200ms
- Indexed on \_id (MongoDB default)
- Single document lookup

---

#### TC-ADMIN-PERF-005: Concurrent Admin Requests ✅

**Steps**:

1. Simulate 50 concurrent GET /admin/users requests
2. Measure throughput and response times

**Expected Result**:

- All requests complete successfully
- Average response time: < 600ms
- Server handles concurrent admin load
- No database connection pool exhaustion

---

#### TC-ADMIN-PERF-006: Large Result Set Pagination ✅

**Steps**:

1. GET /admin/users?limit=100 (maximum allowed)
2. Measure response time and payload size

**Expected Result**:

- Response time: < 800ms
- Payload size: reasonable (< 1 MB)
- Large limits may impact performance
- Pagination prevents loading all users at once

---

#### TC-ADMIN-PERF-007: Status Update Response Time ✅

**Steps**:

1. PATCH /admin/users/:id/status
2. Measure response time

**Expected Result**:

- Response time: < 300ms
- Single document update
- Token revocation adds minimal overhead
- Audit logging asynchronous (non-blocking)

---

## Conclusion

This test plan covers **70 comprehensive test cases** for the Admin Features:

- **User Listing**: 15 tests
- **User Statistics**: 5 tests
- **Single User Retrieval**: 4 tests
- **Account Status Management**: 8 tests
- **Role Management**: 8 tests
- **Edge Cases**: 10 tests
- **Security**: 10 tests
- **Performance**: 7 tests

**Home**: [Home →](../README.md)

**Previous Feature**: [← Feature 11: User Management ](./14-user-management.md)
