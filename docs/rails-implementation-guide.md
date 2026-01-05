# Homeschool Master - Rails API Implementation Guide

Technical implementation specification for the Homeschool Master Rails API backend.

---

## Table of Contents

- [Phase 0: Project Foundation](#phase-0-project-foundation)
- [Phase 1: Authentication System](#phase-1-authentication-system)
- [Phase 2: Student Management](#phase-2-student-management)
- [Phase 3: Calendar & Events](#phase-3-calendar--events)
- [Phase 4: Assignments & Tasks](#phase-4-assignments--tasks)
- [Phase 5: Report Cards & Progress](#phase-5-report-cards--progress)
- [Phase 6: Expense Tracking](#phase-6-expense-tracking)
- [Phase 7: Lesson Planning](#phase-7-lesson-planning)
- [Phase 8: Production Preparation](#phase-8-production-preparation)

---

## Phase 0: Project Foundation

### Technical Requirements

**Rails Configuration:**
- API-only application
- PostgreSQL database
- UUID primary keys
- No test framework (using RSpec separately)

**Required Dependencies:**
- Password encryption
- JWT handling
- CORS middleware
- Background job processing
- Caching layer
- Cloud file storage client
- Pagination
- RSpec and factories
- Code linter

**PostgreSQL Setup:**
- Enable UUID extension (pgcrypto)
- Configure connection pooling
- Set up development and test databases

**CORS Configuration:**
- Allow requests from local development (localhost:8081)
- Allow requests from production domains
- Expose headers: Authorization, X-RateLimit-Limit, X-RateLimit-Remaining
- Allow methods: GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD

**Environment Variables:**
- Database connection strings
- JWT secret key
- JWT expiration times (access: 3600s, refresh: 2592000s)
- Redis connection
- AWS S3 credentials and bucket
- Application host URLs

---

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Rails API Server                    │
│                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │ Controllers  │  │   Services    │  │  Models   │ │
│  │  (routes)    │→ │   (logic)     │→ │ (database)│ │
│  └──────────────┘  └──────────────┘  └───────────┘ │
│                                                       │
│  ┌──────────────────────────────────────────────┐   │
│  │          PostgreSQL Database                  │   │
│  │  (UUID primary keys, indexes, constraints)   │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
         ↑                                    ↑
         │ JSON requests                      │ JSON responses
         │                                    │
    ┌────────────────────────────────────────────┐
    │        React Native Mobile App             │
    └────────────────────────────────────────────┘
```

---

### Base Controller Implementation

**Location:** `app/controllers/api/v1/base_controller.rb`

**Functionality:**
- Inherit from ApplicationController
- Run authentication before all actions
- Provide `current_teacher` accessor to child controllers
- Implement authentication logic:
  - Extract token from Authorization header
  - Decode and verify JWT
  - Load teacher from token payload
  - Verify teacher exists and is active
  - Return 401 for invalid/missing/expired tokens

**Response Helpers:**
- `render_success(data, status: :ok, meta: nil)` - Success responses with optional metadata
- `render_created(data)` - 201 responses
- `render_no_content` - 204 responses
- `render_error(message, code:, status:, details: nil)` - Error responses
- `render_unauthorized(message)` - 401 responses
- `render_forbidden(message)` - 403 responses
- `render_not_found(resource)` - 404 responses
- `render_validation_errors(model)` - 422 responses with model errors

**Pagination Helper:**
- `paginate(collection)` - Returns paginated data with metadata
- Metadata includes: page, limit, total, total_pages, has_next, has_prev
- Default limit: 20 items per page

**Authentication Skip:**
- Child controllers can skip authentication with `skip_before_action :authenticate_request`

---

### Routes Structure

**Namespace:** `/api/v1`

**Public Routes (no authentication):**
- POST `/auth/register`
- POST `/auth/login`
- POST `/auth/refresh`
- POST `/auth/logout`
- POST `/auth/password/reset-request`
- POST `/auth/password/reset`
- POST `/auth/email/verify`
- POST `/auth/email/resend-verification`

**Protected Routes (authentication required):**
- All other endpoints require valid JWT access token
- Token provided in Authorization header: `Bearer <token>`

**Health Check:**
- GET `/health` - No authentication required

---

### Testing Requirements

**Project Setup:**
- Rails server starts successfully
- PostgreSQL connection works
- UUID generation enabled
- All gems installed

**CORS:**
- Cross-origin requests succeed
- Correct headers returned
- Credentials handled properly

**Base Controller:**
- Rejects requests without tokens
- Rejects requests with invalid tokens
- Rejects requests with expired tokens
- Accepts requests with valid tokens
- Sets current_teacher correctly
- Response formatters return correct JSON structure
- Pagination returns correct metadata

---

### Validation Checklist

- [ ] Rails project initialized in API mode
- [ ] PostgreSQL configured
- [ ] UUID extension enabled
- [ ] All required gems in Gemfile
- [ ] CORS configured
- [ ] Environment variables template created
- [ ] Base controller implemented
- [ ] Routes namespace configured
- [ ] Server starts without errors
- [ ] Can make test request and receive JSON response
- [ ] Linter configured

---

## Phase 1: Authentication System

### Technical Requirements

**JWT Configuration:**
- Algorithm: HS256 (HMAC-SHA256)
- Access token expiry: 1 hour
- Refresh token expiry: 30 days
- Include expiration (exp) and issued-at (iat) claims
- Refresh tokens include unique identifier (jti)

**Password Security:**
- Bcrypt hashing
- Minimum 8 characters
- Validation on creation and update

**Token Storage:**
- Access tokens: Not stored (stateless)
- Refresh tokens: Stored in database with jti
- Enable token revocation via database

---

### Architecture

**Authentication Flow:**
```
┌──────┐                    ┌──────┐                   ┌──────────┐
│Client│                    │ API  │                   │ Database │
└──┬───┘                    └──┬───┘                   └────┬─────┘
   │                           │                            │
   │ POST /auth/login          │                            │
   │ {email, password}         │                            │
   │──────────────────────────>│                            │
   │                           │                            │
   │                           │ Find teacher by email      │
   │                           │───────────────────────────>│
   │                           │                            │
   │                           │ Teacher record             │
   │                           │<───────────────────────────│
   │                           │                            │
   │                           │ Verify password hash       │
   │                           │                            │
   │                           │ Generate access token      │
   │                           │ (short expiry, no DB)      │
   │                           │                            │
   │                           │ Generate refresh token     │
   │                           │ (long expiry, with jti)    │
   │                           │                            │
   │                           │ Store refresh token        │
   │                           │───────────────────────────>│
   │                           │                            │
   │ 200 OK                    │                            │
   │ {access_token,            │                            │
   │  refresh_token}           │                            │
   │<──────────────────────────│                            │
```

**Authenticated Request Flow:**
```
┌──────┐                    ┌──────┐                   ┌──────────┐
│Client│                    │ API  │                   │ Database │
└──┬───┘                    └──┬───┘                   └────┬─────┘
   │                           │                            │
   │ GET /students             │                            │
   │ Authorization:            │                            │
   │ Bearer {access_token}     │                            │
   │──────────────────────────>│                            │
   │                           │                            │
   │                           │ Extract token from header  │
   │                           │                            │
   │                           │ Decode and verify token    │
   │                           │                            │
   │                           │ Check expiration           │
   │                           │                            │
   │                           │ Find teacher by ID         │
   │                           │───────────────────────────>│
   │                           │                            │
   │                           │ Teacher record             │
   │                           │<───────────────────────────│
   │                           │                            │
   │                           │ Check teacher is active    │
   │                           │                            │
   │                           │ Fetch students             │
   │                           │───────────────────────────>│
   │                           │                            │
   │ 200 OK                    │                            │
   │ {students: [...]}         │                            │
   │<──────────────────────────│                            │
```

---

### Database Schema

**Teachers Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| first_name | string | Not null |
| last_name | string | Not null |
| nickname | string | Nullable |
| email | string | Not null, unique |
| phone | string | Nullable |
| newsletter_subscribed | boolean | Default: false |
| password_digest | string | Not null |
| profile_image_url | string | Nullable |
| email_verified_at | datetime | Nullable |
| email_verification_token | string | Unique, nullable |
| password_reset_token | string | Unique, nullable |
| password_reset_sent_at | datetime | Nullable |
| is_active | boolean | Default: true |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- email (unique)
- email_verification_token (unique)
- password_reset_token (unique)
- is_active

---

**Refresh Tokens Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| token | string | Not null, unique |
| jti | string | Not null, unique |
| expires_at | datetime | Not null |
| revoked_at | datetime | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- token (unique)
- jti (unique)

**Foreign Keys:**
- teacher_id references teachers(id)

---

### JWT Service Implementation

**Location:** `app/services/jwt_service.rb`

**Methods:**

`encode(payload, exp: nil)`
- Add expiration to payload (default from ENV or 3600 seconds)
- Add issued-at timestamp
- Sign with secret key from ENV
- Return JWT string

`encode_refresh_token(teacher_id)`
- Create payload with teacher_id, type: 'refresh', unique jti
- Set expiration from ENV (default 2592000 seconds)
- Add issued-at timestamp
- Sign with secret key
- Return JWT string

`decode(token)`
- Verify signature
- Check expiration
- Return payload hash
- Return nil on JWT::ExpiredSignature
- Return nil on JWT::DecodeError

`decode_refresh_token(token)`
- Decode token
- Verify type is 'refresh'
- Return payload or nil

**Private Methods:**

`secret_key`
- Fetch from ENV['JWT_SECRET_KEY']
- Fallback to Rails credentials secret_key_base

`access_token_expiry`
- Current time + ENV['JWT_ACCESS_TOKEN_EXPIRY'] seconds

`refresh_token_expiry`
- Current time + ENV['JWT_REFRESH_TOKEN_EXPIRY'] seconds

---

### Teacher Model

**Location:** `app/models/teacher.rb`

**Functionality:**
- Use has_secure_password for password handling
- Downcase email before save
- Generate email verification token before create

**Validations:**
- first_name: presence, max length 100
- last_name: presence, max length 100
- nickname: max length 100, optional
- email: presence, uniqueness (case insensitive), format (email regex)
- password: presence on create, minimum length 8
- phone: max length 20, optional

**Associations:**
- has_many :students, dependent: :destroy
- has_many :calendar_events, dependent: :destroy
- has_many :assignments, dependent: :destroy
- has_many :tasks, dependent: :destroy
- has_many :report_cards, dependent: :destroy
- has_many :expenses, dependent: :destroy
- has_many :subjects, dependent: :destroy
- has_many :lesson_plans, dependent: :destroy
- has_many :expense_categories, dependent: :destroy
- has_many :refresh_tokens, dependent: :destroy

**Scopes:**
- active: `where(is_active: true)`
- verified: `where.not(email_verified_at: nil)`

**Instance Methods:**

`full_name`
- Return "#{first_name} #{last_name}"

`display_name`
- Return nickname if present, otherwise full_name

`email_verified?`
- Return true if email_verified_at is present

`verify_email!`
- Set email_verified_at to current time
- Clear email_verification_token
- Save

`generate_password_reset_token!`
- Generate secure random token (32 bytes, URL-safe base64)
- Set password_reset_sent_at to current time
- Save

`password_reset_token_valid?`
- Return false if token or timestamp missing
- Return true if timestamp within last 2 hours

`clear_password_reset_token!`
- Clear password_reset_token and password_reset_sent_at
- Save

**Class Methods:**

`find_by_email(email)`
- Find by downcased email

**Private Methods:**

`downcase_email`
- Set email to lowercase

`generate_email_verification_token`
- Generate secure random token (32 bytes, URL-safe base64)

`password_required?`
- True if new record or password present

---

### Refresh Token Model

**Location:** `app/models/refresh_token.rb`

**Associations:**
- belongs_to :teacher

**Validations:**
- token: presence, uniqueness
- jti: presence, uniqueness
- expires_at: presence

**Scopes:**
- active: `where(revoked_at: nil).where('expires_at > ?', Time.current)`

**Instance Methods:**

`expired?`
- Return true if expires_at <= current time

`revoked?`
- Return true if revoked_at present

`valid_token?`
- Return true if not expired and not revoked

`revoke!`
- Set revoked_at to current time
- Save

**Class Methods:**

`find_valid_by_token(token)`
- Find in active scope by token

`revoke_all_for_teacher(teacher_id)`
- Update all tokens for teacher, set revoked_at

---

### Authentication Controller

**Location:** `app/controllers/api/v1/authentication_controller.rb`

**Configuration:**
- Skip authentication for: register, login, refresh

**Endpoints:**

`POST /api/v1/auth/register`
- Accept: first_name, last_name, email, password
- Validate all required fields
- Check email uniqueness
- Create teacher
- Generate email verification token
- Return 201 with teacher data (exclude password_digest)
- Return 422 with validation errors

`POST /api/v1/auth/login`
- Accept: email, password
- Find teacher by email
- Verify password
- Check account is active
- Generate access token
- Generate refresh token with jti
- Store refresh token in database
- Return 200 with access_token and refresh_token
- Return 401 for invalid credentials
- Return 401 for inactive account

`POST /api/v1/auth/refresh`
- Accept: refresh_token
- Decode refresh token
- Find refresh token in database by jti
- Verify not revoked
- Verify not expired
- Generate new access token
- Return 200 with new access_token
- Return 401 for invalid/expired/revoked token

`POST /api/v1/auth/logout`
- Accept: refresh_token
- Decode refresh token
- Find in database
- Revoke token
- Return 204
- Return 401 for invalid token

`POST /api/v1/auth/email/verify`
- Accept: token
- Find teacher by email_verification_token
- Verify email
- Return 200 with success message
- Return 400 for invalid token

`POST /api/v1/auth/email/resend-verification`
- Requires authentication
- Generate new verification token
- Save to current teacher
- Return 200 with success message

`POST /api/v1/auth/password/reset-request`
- Accept: email
- Find teacher by email
- Generate password reset token
- Return 200 (always, for security)

`POST /api/v1/auth/password/reset`
- Accept: token, password
- Find teacher by password_reset_token
- Verify token valid (not expired)
- Update password
- Clear reset token
- Return 200 with success message
- Return 400 for invalid/expired token

`POST /api/v1/auth/password/change`
- Requires authentication
- Accept: current_password, new_password
- Verify current password
- Update to new password
- Return 200 with success message
- Return 401 for incorrect current password

---

### Testing Requirements

**JWT Service:**
- Encodes payload into valid JWT
- Decoded token contains original payload
- Access tokens expire after configured time
- Refresh tokens expire after configured time
- Returns nil for expired tokens
- Returns nil for invalid signatures
- Returns nil for malformed tokens
- Refresh tokens include jti

**Teacher Model:**
- Cannot create without required fields
- Email must be unique (case insensitive)
- Email must be valid format
- Password must be at least 8 characters
- Password is hashed, not stored as plain text
- Can find by email (case insensitive)
- Email verification token generated on creation
- Can verify email
- Can generate password reset token
- Reset token expires after 2 hours
- full_name combines first and last
- display_name returns nickname or full_name

**Refresh Token Model:**
- Cannot create without required fields
- Token must be unique
- JTI must be unique
- Active scope excludes revoked tokens
- Active scope excludes expired tokens
- Can revoke token
- Can revoke all for teacher
- Validation methods work correctly

**Authentication Controller:**

*Register:*
- Creates teacher with valid data
- Returns 422 for duplicate email
- Returns 422 for invalid email
- Returns 422 for weak password
- Returns 422 for missing fields
- Generates verification token

*Login:*
- Returns tokens for valid credentials
- Returns 401 for wrong password
- Returns 401 for non-existent email
- Returns 401 for inactive account
- Stores refresh token in database

*Refresh:*
- Returns new access token with valid refresh token
- Returns 401 for invalid refresh token
- Returns 401 for expired refresh token
- Returns 401 for revoked refresh token

*Logout:*
- Revokes refresh token
- Returns 204 for valid token
- Returns 401 for invalid token

*Email Verification:*
- Verifies email with valid token
- Returns 400 for invalid token
- Can resend verification

*Password Reset:*
- Generates reset token
- Always returns 200 (security)
- Can reset with valid token
- Returns 400 for expired token
- Token expires after 2 hours

*Password Change:*
- Changes password with correct current password
- Returns 401 for wrong current password
- Requires authentication

---

### Validation Checklist

- [ ] JWT service implemented
- [ ] Token encoding/decoding works
- [ ] Access tokens expire correctly
- [ ] Refresh tokens expire correctly
- [ ] Teacher model created with password encryption
- [ ] Email validations work
- [ ] Email verification flow implemented
- [ ] Password reset flow implemented
- [ ] Refresh token model created
- [ ] Registration endpoint works
- [ ] Login endpoint returns tokens
- [ ] Refresh endpoint works
- [ ] Logout endpoint revokes tokens
- [ ] Cannot use revoked tokens
- [ ] Password reset works
- [ ] Password change works
- [ ] All authentication tests pass

---

## Phase 2: Student Management

### Technical Requirements

**Data Model:**
- Students belong to teachers (one-to-many)
- Soft deletion (archived_at timestamp)
- Foreign key constraints enforced

**Validation:**
- Model-level validations for user feedback
- Database-level constraints for data integrity

---

### Architecture

```
┌─────────────────────────────────────┐
│           teachers                   │
└──────────────┬──────────────────────┘
               │
               │ has_many
               │
               ▼
┌─────────────────────────────────────┐
│           students                   │
│─────────────────────────────────────│
│ id (uuid)                            │
│ teacher_id (foreign key) ────────┐  │
│ first_name                        │  │
│ last_name                         │  │
│ date_of_birth                     │  │
│ grade_level                       │  │
│ notes                             │  │
│ archived_at (soft delete)         │  │
│ created_at                        │  │
│ updated_at                        │  │
└───────────────────────────────────┘  │
                                       │
                                       │
          ┌────────────────────────────┘
          │
          ▼
     Ensures student
     belongs to valid teacher
```

---

### Database Schema

**Students Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| first_name | string | Not null |
| last_name | string | Not null |
| nickname | string | Nullable |
| date_of_birth | date | Not null |
| grade_level | string | Not null |
| notes | text | Nullable |
| archived_at | datetime | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- archived_at
- Composite: teacher_id + archived_at

**Foreign Keys:**
- teacher_id references teachers(id)

**Check Constraints:**
- date_of_birth must be in the past

---

### Student Model

**Location:** `app/models/student.rb`

**Associations:**
- belongs_to :teacher

**Validations:**
- first_name: presence, max length 100
- last_name: presence, max length 100
- nickname: max length 100, optional
- date_of_birth: presence, must be in past
- grade_level: presence
- teacher: must exist

**Scopes:**
- active: `where(archived_at: nil)`
- archived: `where.not(archived_at: nil)`
- by_grade: `where(grade_level: level)`
- alphabetical: `order(:last_name, :first_name)`

**Instance Methods:**

`full_name`
- Return "#{first_name} #{last_name}"

`display_name`
- Return nickname if present, otherwise full_name

`age`
- Calculate from date_of_birth to current date
- Return integer years

`archive!`
- Set archived_at to current time
- Save

`restore!`
- Clear archived_at
- Save

`archived?`
- Return true if archived_at present

---

### Students Controller

**Location:** `app/controllers/api/v1/students_controller.rb`

**Configuration:**
- All actions require authentication
- Verify student belongs to current_teacher

**Endpoints:**

`GET /api/v1/students`
- Filter by grade_level (optional)
- Search by name (optional)
- Return only active students by default
- Support pagination
- Return count and metadata

**Query Parameters:**
- grade: filter by grade level
- search: search in first_name, last_name, nickname
- page: page number
- limit: items per page

**Response:**
```json
{
  "data": [...],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "total_pages": 3,
    "has_next": true,
    "has_prev": false
  }
}
```

`GET /api/v1/students/:id`
- Find student by ID
- Verify belongs to current_teacher
- Return student details
- Return 404 if not found
- Return 403 if belongs to different teacher

`POST /api/v1/students`
- Accept: first_name, last_name, nickname, date_of_birth, grade_level, notes
- Link to current_teacher
- Validate all fields
- Return 201 with created student
- Return 422 with validation errors

`PATCH /api/v1/students/:id`
- Accept: first_name, last_name, nickname, date_of_birth, grade_level, notes
- Find student by ID
- Verify belongs to current_teacher
- Update allowed fields
- Validate changes
- Return 200 with updated student
- Return 404 if not found
- Return 403 if wrong teacher
- Return 422 with validation errors

`POST /api/v1/students/:id/archive`
- Find student by ID
- Verify belongs to current_teacher
- Set archived_at
- Return 200 with success message
- Return 404 if not found
- Return 403 if wrong teacher
- Return 400 if already archived

`POST /api/v1/students/:id/restore`
- Find student by ID (including archived)
- Verify belongs to current_teacher
- Clear archived_at
- Return 200 with success message
- Return 404 if not found
- Return 403 if wrong teacher
- Return 400 if not archived

---

### Testing Requirements

**Student Model:**
- Cannot create without teacher
- Cannot create without required fields
- Names have max length limits
- Date of birth must be in past
- Age calculation correct
- full_name combines correctly
- display_name uses nickname if available
- Can archive student
- Archived students not in active scope
- Can restore archived student
- Grade level scope filters correctly

**Students Controller:**

*List:*
- Returns only current_teacher's students
- Returns only active students by default
- Pagination works correctly
- Metadata accurate
- Can filter by grade level
- Can search by name
- Requires authentication

*Show:*
- Returns student details
- Returns 403 for other teacher's student
- Returns 404 for non-existent student
- Requires authentication

*Create:*
- Can create with valid data
- Student linked to current_teacher
- Returns 422 for missing fields
- Returns 422 for invalid date of birth
- Returns created student data
- Requires authentication

*Update:*
- Can update fields
- Returns 403 for other teacher's student
- Returns 422 for invalid data
- Cannot change teacher_id
- Requires authentication

*Archive:*
- Can archive student
- Student not in active list
- Can still retrieve by ID
- Returns 400 if already archived
- Requires authentication

*Restore:*
- Can restore archived student
- Student appears in active list
- Returns 400 if not archived
- Requires authentication

---

### Validation Checklist

- [ ] Student model created with all fields
- [ ] Student belongs_to teacher association works
- [ ] Validations prevent invalid data
- [ ] Soft deletion implemented
- [ ] Can create students via API
- [ ] Can list students with pagination
- [ ] Can update student details
- [ ] Can archive and restore students
- [ ] Students filtered by teacher
- [ ] Search functionality works
- [ ] All student tests pass

---

## Phase 3: Calendar & Events

### Technical Requirements

**Data Model:**
- Events belong to teachers
- Polymorphic association (can link to students, assignments, lesson plans)
- Support for all-day events
- Support for recurring events
- Timezone-aware timestamps

**Event Types:**
- lesson
- field_trip
- appointment
- holiday
- meeting
- other

---

### Architecture

```
┌─────────────────────┐
│      teachers       │
└──────────┬──────────┘
           │
           │ has_many
           │
           ▼
┌─────────────────────────────────┐
│      calendar_events            │
│─────────────────────────────────│
│ id                              │
│ teacher_id (foreign key)        │
│ title                           │
│ description                     │
│ start_time                      │
│ end_time                        │
│ all_day (boolean)               │
│ event_type (enum)               │
│ recurrence_rule (optional)      │
│ eventable_type (polymorphic)    │◄──┐
│ eventable_id (polymorphic)      │   │
└─────────────────────────────────┘   │
                                       │
         Can link to:                  │
         ┌─────────────────────────────┘
         │
         ├──► Student
         ├──► Assignment
         └──► Lesson Plan
```

---

### Database Schema

**Calendar Events Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| title | string | Not null |
| description | text | Nullable |
| start_time | datetime | Not null |
| end_time | datetime | Nullable |
| all_day | boolean | Default: false |
| event_type | string | Not null |
| recurrence_rule | text | Nullable |
| eventable_type | string | Nullable |
| eventable_id | uuid | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- start_time
- Composite: teacher_id + start_time
- Composite: eventable_type + eventable_id

**Foreign Keys:**
- teacher_id references teachers(id)

**Valid Event Types:**
- lesson
- field_trip
- appointment
- holiday
- meeting
- other

---

### Calendar Event Model

**Location:** `app/models/calendar_event.rb`

**Associations:**
- belongs_to :teacher
- belongs_to :eventable, polymorphic: true, optional: true

**Validations:**
- title: presence, max length 200
- start_time: presence
- end_time: must be after start_time (if present)
- event_type: presence, inclusion in valid types
- teacher: must exist

**Scopes:**
- between_dates: `where('start_time >= ? AND start_time <= ?', start_date, end_date)`
- by_type: `where(event_type: type)`
- all_day: `where(all_day: true)`
- recurring: `where.not(recurrence_rule: nil)`
- for_student: `where(eventable_type: 'Student', eventable_id: student_id)`

**Instance Methods:**

`happening_now?`
- Return true if current time between start_time and end_time

`all_day?`
- Return all_day value

`duration`
- Return difference between end_time and start_time (in seconds)
- Return nil if no end_time

`recurring?`
- Return true if recurrence_rule present

---

### Calendar Events Controller

**Location:** `app/controllers/api/v1/calendar_events_controller.rb`

**Configuration:**
- All actions require authentication
- Verify event belongs to current_teacher

**Endpoints:**

`GET /api/v1/calendar_events`
- Required: start_date, end_date (for date range)
- Optional: event_type, student_id
- Return events in date range
- Include associated records (eventable)
- Support pagination

**Query Parameters:**
- start_date: ISO 8601 date (required)
- end_date: ISO 8601 date (required)
- event_type: filter by type
- student_id: filter by linked student
- page: page number
- limit: items per page

`GET /api/v1/calendar_events/:id`
- Find event by ID
- Verify belongs to current_teacher
- Include eventable record
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/calendar_events`
- Accept: title, description, start_time, end_time, all_day, event_type, recurrence_rule, eventable_type, eventable_id
- Link to current_teacher
- Validate times
- Return 201 with created event
- Return 422 with validation errors

`PATCH /api/v1/calendar_events/:id`
- Accept: title, description, start_time, end_time, all_day, event_type, recurrence_rule, eventable_type, eventable_id
- Find event by ID
- Verify belongs to current_teacher
- Update fields
- Validate changes
- Return 200 with updated event
- Return 404 if not found
- Return 403 if wrong teacher
- Return 422 with validation errors

`DELETE /api/v1/calendar_events/:id`
- Find event by ID
- Verify belongs to current_teacher
- Delete event
- Return 204
- Return 404 if not found
- Return 403 if wrong teacher

---

### Testing Requirements

**Calendar Event Model:**
- Cannot create without teacher
- Cannot create without title
- Cannot create without start_time
- End time must be after start time
- All-day events work correctly
- Event type validation works
- Can link to student (polymorphic)
- Date range filtering works
- Recurring events identified correctly
- Duration calculation works

**Calendar Events Controller:**

*List:*
- Returns events in date range
- Returns only current_teacher's events
- Can filter by event type
- Can filter by student
- Pagination works
- Includes related records
- Requires date range parameters
- Requires authentication

*Show:*
- Returns event details
- Includes linked records
- Returns 403 for other teacher's event
- Requires authentication

*Create:*
- Can create with valid data
- Can create all-day event
- Can link to student
- Returns 422 for end before start
- Returns 422 for missing fields
- Requires authentication

*Update:*
- Can update fields
- Cannot change teacher
- Time validation still applies
- Requires authentication

*Delete:*
- Can delete event
- Event removed from list
- Returns 404 for non-existent
- Requires authentication

---

### Validation Checklist

- [ ] Calendar event model created
- [ ] Polymorphic associations work
- [ ] Date range queries efficient
- [ ] Can create events via API
- [ ] Can list events for date range
- [ ] Can filter events by type
- [ ] All-day events work
- [ ] Can link events to students
- [ ] Validation prevents invalid data
- [ ] All calendar tests pass

---

## Phase 4: Assignments & Tasks

### Technical Requirements

**Data Models:**
- Assignments: Given to students, tracked for completion
- Tasks: Teacher's personal to-do items
- Join table: Links students to assignments with status

**Assignment Tracking:**
- Status: not_started, in_progress, completed, graded
- Completion timestamp
- Grade/score
- Teacher notes

---

### Architecture

```
┌──────────────┐
│   teachers   │
└──┬────────┬──┘
   │        │
   │        │ has_many
   │        ├─────────────────┐
   │        │                 │
   │        ▼                 ▼
   │  ┌────────────┐    ┌──────────┐
   │  │assignments │    │  tasks   │
   │  └─────┬──────┘    └──────────┘
   │        │
   │        │ has_many
   │        │
   │        ▼
   │  ┌──────────────────┐
   │  │student_assignments│ (join table)
   │  │──────────────────│
   │  │ student_id       │
   │  │ assignment_id    │
   │  │ status           │
   │  │ completed_at     │
   │  │ grade            │
   │  │ notes            │
   │  └──────────────────┘
   │        │
   └────────┤
            │ belongs_to
            ▼
      ┌──────────┐
      │ students │
      └──────────┘
```

---

### Database Schema

**Assignments Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| title | string | Not null |
| description | text | Nullable |
| subject | string | Nullable |
| due_date | datetime | Not null |
| estimated_time | integer | Minutes, nullable |
| instructions | text | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- due_date
- subject

---

**Student Assignments Table (Join):**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| student_id | uuid | Foreign key, not null |
| assignment_id | uuid | Foreign key, not null |
| status | string | Not null, default: 'not_started' |
| completed_at | datetime | Nullable |
| grade | decimal(5,2) | Nullable |
| notes | text | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- student_id
- assignment_id
- Composite: student_id + assignment_id (unique)
- status

**Foreign Keys:**
- student_id references students(id)
- assignment_id references assignments(id)

**Valid Status Values:**
- not_started
- in_progress
- completed
- graded

---

**Tasks Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| title | string | Not null |
| description | text | Nullable |
| due_date | datetime | Nullable |
| priority | string | Not null, default: 'medium' |
| status | string | Not null, default: 'pending' |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- due_date
- priority
- status

**Valid Priority Values:**
- low
- medium
- high

**Valid Status Values:**
- pending
- in_progress
- completed

---

### Assignment Model

**Location:** `app/models/assignment.rb`

**Associations:**
- belongs_to :teacher
- has_many :student_assignments, dependent: :destroy
- has_many :students, through: :student_assignments

**Validations:**
- title: presence, max length 200
- due_date: presence
- teacher: must exist

**Scopes:**
- upcoming: `where('due_date > ?', Time.current).order(:due_date)`
- overdue: `where('due_date < ?', Time.current)`
- by_subject: `where(subject: subject)`
- due_between: `where('due_date >= ? AND due_date <= ?', start_date, end_date)`

**Instance Methods:**

`overdue?`
- Return true if due_date < current time

`students_count`
- Return count of assigned students

`completed_count`
- Return count of student_assignments with status 'completed' or 'graded'

`completion_percentage`
- Return (completed_count / students_count) * 100
- Return 0 if no students assigned

---

### Student Assignment Model

**Location:** `app/models/student_assignment.rb`

**Associations:**
- belongs_to :student
- belongs_to :assignment

**Validations:**
- student: must exist
- assignment: must exist
- status: presence, inclusion in valid values
- grade: numericality (0-100), optional
- Unique combination: student_id + assignment_id

**Scopes:**
- completed: `where(status: ['completed', 'graded'])`
- graded: `where(status: 'graded')`
- not_started: `where(status: 'not_started')`
- for_student: `where(student_id: student_id)`

**Instance Methods:**

`mark_completed!`
- Set status to 'completed'
- Set completed_at to current time
- Save

`mark_graded!(grade, notes: nil)`
- Set status to 'graded'
- Set grade
- Set notes if provided
- Save

`overdue?`
- Return true if assignment.due_date < current time and not completed

---

### Task Model

**Location:** `app/models/task.rb`

**Associations:**
- belongs_to :teacher

**Validations:**
- title: presence, max length 200
- priority: presence, inclusion in valid values
- status: presence, inclusion in valid values
- teacher: must exist

**Scopes:**
- incomplete: `where.not(status: 'completed')`
- high_priority: `where(priority: 'high')`
- due_today: `where('DATE(due_date) = ?', Date.current)`
- overdue: `where('due_date < ? AND status != ?', Time.current, 'completed')`

**Instance Methods:**

`mark_completed!`
- Set status to 'completed'
- Save

`overdue?`
- Return true if due_date present and < current time and not completed

---

### Assignments Controller

**Location:** `app/controllers/api/v1/assignments_controller.rb`

**Configuration:**
- All actions require authentication
- Verify assignment belongs to current_teacher

**Endpoints:**

`GET /api/v1/assignments`
- Optional: subject, start_date, end_date
- Return assignments with completion stats
- Support pagination

`GET /api/v1/assignments/:id`
- Find assignment by ID
- Verify belongs to current_teacher
- Include list of assigned students with status
- Include completion statistics
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/assignments`
- Accept: title, description, subject, due_date, estimated_time, instructions, student_ids
- Link to current_teacher
- Create student_assignments for each student_id
- Return 201 with assignment
- Return 422 with validation errors

`PATCH /api/v1/assignments/:id`
- Accept: title, description, subject, due_date, estimated_time, instructions
- Find assignment by ID
- Verify belongs to current_teacher
- Update fields
- Return 200 with updated assignment
- Return 404 if not found
- Return 403 if wrong teacher
- Return 422 with validation errors

`DELETE /api/v1/assignments/:id`
- Find assignment by ID
- Verify belongs to current_teacher
- Delete assignment (cascades to student_assignments)
- Return 204
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/assignments/:id/assign`
- Accept: student_ids (array)
- Find assignment by ID
- Verify belongs to current_teacher
- Create student_assignments for new students
- Skip existing assignments
- Return 200 with updated assignment
- Return 404 if not found
- Return 403 if wrong teacher

`DELETE /api/v1/assignments/:id/unassign`
- Accept: student_ids (array)
- Find assignment by ID
- Verify belongs to current_teacher
- Delete student_assignments for specified students
- Return 200 with updated assignment
- Return 404 if not found
- Return 403 if wrong teacher

---

### Student Assignments Controller

**Location:** `app/controllers/api/v1/student_assignments_controller.rb`

**Configuration:**
- All actions require authentication

**Endpoints:**

`GET /api/v1/students/:student_id/assignments`
- Verify student belongs to current_teacher
- Optional: status filter
- Return all assignments for student with status
- Support pagination
- Return 403 if wrong teacher

`PATCH /api/v1/student_assignments/:id`
- Accept: status, grade, notes
- Find student_assignment by ID
- Verify both student and assignment belong to current_teacher
- Update fields
- If status changed to 'completed', set completed_at
- Return 200 with updated record
- Return 404 if not found
- Return 403 if wrong teacher
- Return 422 with validation errors

`POST /api/v1/student_assignments/bulk_update`
- Accept: array of {id, status, grade, notes}
- Verify all records belong to current_teacher
- Update each record
- Return 200 with success count
- Return 422 with errors for failed updates

---

### Tasks Controller

**Location:** `app/controllers/api/v1/tasks_controller.rb`

**Configuration:**
- All actions require authentication
- Verify task belongs to current_teacher

**Endpoints:**

`GET /api/v1/tasks`
- Optional: status, priority filters
- Sort by due_date by default
- Support pagination

`GET /api/v1/tasks/:id`
- Find task by ID
- Verify belongs to current_teacher
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/tasks`
- Accept: title, description, due_date, priority, status
- Link to current_teacher
- Return 201 with created task
- Return 422 with validation errors

`PATCH /api/v1/tasks/:id`
- Accept: title, description, due_date, priority, status
- Find task by ID
- Verify belongs to current_teacher
- Update fields
- Return 200 with updated task
- Return 404 if not found
- Return 403 if wrong teacher
- Return 422 with validation errors

`DELETE /api/v1/tasks/:id`
- Find task by ID
- Verify belongs to current_teacher
- Delete task
- Return 204
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/tasks/:id/complete`
- Find task by ID
- Verify belongs to current_teacher
- Mark as completed
- Return 200 with updated task
- Return 404 if not found
- Return 403 if wrong teacher

---

### Testing Requirements

**Assignment Model:**
- Cannot create without teacher
- Cannot create without title or due_date
- Can assign to multiple students
- Completion percentage calculates correctly
- Identifies overdue assignments
- Scopes filter correctly

**Student Assignment Model:**
- Cannot create duplicate student + assignment
- Status transitions work correctly
- Can add grade
- Can add notes
- Completion timestamp set automatically
- Unique constraint enforced

**Task Model:**
- Cannot create without required fields
- Priority validation works
- Status validation works
- Can mark complete
- Overdue detection works

**Controllers:**
- Can create assignments
- Can assign to students
- Can update assignment status
- Can grade assignments
- Students only see their assignments
- Teachers only see their assignments/tasks
- Bulk operations work
- All require authentication

---

### Validation Checklist

- [ ] Assignment model with associations works
- [ ] Student assignment join table works
- [ ] Task model works
- [ ] Can create assignments and assign to students
- [ ] Can track assignment completion
- [ ] Can grade assignments
- [ ] Can create and manage tasks
- [ ] Completion statistics calculate correctly
- [ ] All assignment/task tests pass

---

## Phase 5: Report Cards & Progress

### Technical Requirements

**Data Model:**
- Report cards are snapshots in time
- Each report card has multiple subject grades
- Grades use letter (A-F) or numeric (0-100) scale
- Finalized report cards cannot be edited

**Grading Periods:**
- Q1, Q2, Q3, Q4 (quarters)
- S1, S2 (semesters)
- Year (annual)
- Custom

---

### Architecture

```
┌──────────────┐
│   teachers   │
└──────┬───────┘
       │ has_many
       │
       ▼
┌──────────────────┐
│   report_cards   │
│──────────────────│
│ teacher_id       │
│ student_id       │───────────┐
│ grading_period   │           │
│ start_date       │           │
│ end_date         │           │
│ overall_comments │           │
│ finalized_at     │           │
└──────┬───────────┘           │
       │                       │
       │ has_many              │
       │                       │
       ▼                       │
┌──────────────────┐           │
│  subject_grades  │           │
│──────────────────│           │
│ report_card_id   │           │
│ subject_name     │           │
│ letter_grade     │           │
│ numeric_grade    │           │
│ comments         │           │
│ teacher_notes    │           │
└──────────────────┘           │
                               │
                               │ belongs_to
                               │
                               ▼
                          ┌──────────┐
                          │ students │
                          └──────────┘
```

---

### Database Schema

**Report Cards Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| student_id | uuid | Foreign key, not null |
| grading_period | string | Not null |
| start_date | date | Not null |
| end_date | date | Not null |
| overall_comments | text | Nullable |
| finalized_at | datetime | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- student_id
- grading_period
- Composite: student_id + grading_period (unique)

**Foreign Keys:**
- teacher_id references teachers(id)
- student_id references students(id)

**Check Constraints:**
- end_date must be after start_date
- No overlapping periods for same student

---

**Subject Grades Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| report_card_id | uuid | Foreign key, not null |
| subject_name | string | Not null |
| letter_grade | string | Nullable |
| numeric_grade | decimal(5,2) | Nullable |
| comments | text | Nullable |
| teacher_notes | text | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- report_card_id
- Composite: report_card_id + subject_name (unique)

**Foreign Keys:**
- report_card_id references report_cards(id)

**Valid Letter Grades:**
- A+, A, A-, B+, B, B-, C+, C, C-, D+, D, D-, F

**Valid Numeric Grades:**
- 0 to 100

---

### Report Card Model

**Location:** `app/models/report_card.rb`

**Associations:**
- belongs_to :teacher
- belongs_to :student
- has_many :subject_grades, dependent: :destroy

**Validations:**
- teacher: must exist
- student: must exist
- grading_period: presence
- start_date: presence
- end_date: presence, must be after start_date
- No overlapping periods for same student (custom validation)

**Scopes:**
- finalized: `where.not(finalized_at: nil)`
- draft: `where(finalized_at: nil)`
- by_period: `where(grading_period: period)`

**Instance Methods:**

`gpa`
- Calculate GPA from subject_grades
- Use 4.0 scale (A=4.0, B=3.0, C=2.0, D=1.0, F=0.0)
- Return rounded to 2 decimals

`finalize!`
- Set finalized_at to current time
- Save
- Raise error if already finalized

`finalized?`
- Return true if finalized_at present

---

### Subject Grade Model

**Location:** `app/models/subject_grade.rb`

**Associations:**
- belongs_to :report_card

**Validations:**
- report_card: must exist
- subject_name: presence
- At least one of letter_grade or numeric_grade must be present
- letter_grade: inclusion in valid values (if present)
- numeric_grade: numericality between 0 and 100 (if present)
- Unique combination: report_card_id + subject_name

**Instance Methods:**

`grade_point`
- Convert letter_grade to point value for GPA
- A/A+ = 4.0, A- = 3.7, B+ = 3.3, B = 3.0, B- = 2.7, C+ = 2.3, C = 2.0, C- = 1.7, D+ = 1.3, D = 1.0, D- = 0.7, F = 0.0

`convert_to_letter(numeric)`
- Convert numeric grade to letter
- 97-100: A+, 93-96: A, 90-92: A-, 87-89: B+, etc.

`convert_to_numeric(letter)`
- Convert letter grade to numeric
- A+ = 98, A = 95, A- = 92, B+ = 88, etc.

---

### Report Cards Controller

**Location:** `app/controllers/api/v1/report_cards_controller.rb`

**Configuration:**
- All actions require authentication
- Verify report card belongs to current_teacher

**Endpoints:**

`GET /api/v1/students/:student_id/report_cards`
- Verify student belongs to current_teacher
- Optional: grading_period, status (draft/finalized) filters
- Return all report cards for student
- Include subject_grades
- Include GPA calculation
- Support pagination
- Return 403 if wrong teacher

`GET /api/v1/report_cards/:id`
- Find report card by ID
- Verify belongs to current_teacher
- Include all subject_grades
- Include student info
- Calculate GPA
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/students/:student_id/report_cards`
- Verify student belongs to current_teacher
- Accept: grading_period, start_date, end_date, overall_comments, subject_grades
- Create report card (draft status)
- Create subject_grades if provided
- Return 201 with created report card
- Return 422 with validation errors
- Return 403 if wrong teacher

`PATCH /api/v1/report_cards/:id`
- Find report card by ID
- Verify belongs to current_teacher
- Return 422 if finalized
- Accept: overall_comments, subject_grades
- Update fields
- Update or create subject_grades
- Return 200 with updated report card
- Return 404 if not found
- Return 403 if wrong teacher
- Return 422 if finalized or validation errors

`POST /api/v1/report_cards/:id/finalize`
- Find report card by ID
- Verify belongs to current_teacher
- Return 422 if already finalized
- Set finalized_at
- Return 200 with finalized report card
- Return 404 if not found
- Return 403 if wrong teacher

`DELETE /api/v1/report_cards/:id`
- Find report card by ID
- Verify belongs to current_teacher
- Return 422 if finalized
- Delete report card (cascades to subject_grades)
- Return 204
- Return 404 if not found
- Return 403 if wrong teacher

---

### Subject Grades Controller

**Location:** `app/controllers/api/v1/subject_grades_controller.rb`

**Configuration:**
- All actions require authentication
- Nested under report_cards

**Endpoints:**

`POST /api/v1/report_cards/:report_card_id/subject_grades`
- Find report card by ID
- Verify belongs to current_teacher
- Return 422 if report card finalized
- Accept: subject_name, letter_grade, numeric_grade, comments, teacher_notes
- Create subject_grade
- Return 201 with created grade
- Return 422 with validation errors
- Return 403 if wrong teacher

`PATCH /api/v1/subject_grades/:id`
- Find subject_grade by ID
- Verify report_card belongs to current_teacher
- Return 422 if report card finalized
- Accept: letter_grade, numeric_grade, comments, teacher_notes
- Update fields
- Return 200 with updated grade
- Return 404 if not found
- Return 403 if wrong teacher
- Return 422 if finalized or validation errors

`DELETE /api/v1/subject_grades/:id`
- Find subject_grade by ID
- Verify report_card belongs to current_teacher
- Return 422 if report card finalized
- Delete grade
- Return 204
- Return 404 if not found
- Return 403 if wrong teacher

---

### Testing Requirements

**Report Card Model:**
- Cannot create without teacher and student
- Cannot overlap grading periods for same student
- GPA calculates correctly from subject grades
- Cannot modify finalized report card
- Status transitions work correctly
- Validates date range

**Subject Grade Model:**
- Cannot create without report card
- Grade validation works (A-F, 0-100)
- Letter and numeric grades stay consistent
- Grade point calculation works
- Conversion methods work
- Unique constraint on subject per report card

**Report Cards Controller:**
- Can create draft report card
- Can add subject grades
- Can finalize report card
- Cannot edit finalized report card
- Can list report cards for student
- Can filter by status and period
- GPA appears in response
- Requires authentication and authorization

**Subject Grades Controller:**
- Can add grades to draft report card
- Cannot add grades to finalized report card
- Can update grades on draft
- Cannot update on finalized
- Grade validation works
- Requires authentication

---

### Validation Checklist

- [ ] Report card model created
- [ ] Subject grades model created
- [ ] GPA calculation works
- [ ] Can create report cards with grades
- [ ] Can finalize report cards
- [ ] Finalized cards cannot be edited
- [ ] Can view report history
- [ ] Period overlap validation works
- [ ] All report card tests pass

---

## Phase 6: Expense Tracking

### Technical Requirements

**Data Model:**
- Expenses belong to teachers
- Categories can be nested (parent-child)
- Expenses can be linked to specific students
- Support for recurring expenses
- Receipt file storage

**Recurring Expenses:**
- Frequency: monthly, quarterly, annually
- Used for subscriptions and memberships

---

### Architecture

```
┌──────────────┐
│   teachers   │
└──────┬───────┘
       │
       ├─── has_many ───┐
       │                │
       ▼                ▼
┌──────────────┐  ┌─────────────────┐
│  categories  │  │    expenses     │
└──────┬───────┘  │─────────────────│
       │          │ teacher_id      │
       │          │ category_id     │◄────┐
       │          │ student_id      │     │
       │          │ amount          │     │
       │          │ date            │     │
       └──────────│ description     │     │
  belongs_to     │ receipt_url     │     │
                 │ is_recurring    │     │
                 │ recurring_freq  │     │
                 └─────────────────┘     │
                                         │
                           Optional link to student
                                    (if expense is
                                    specific to one student)
```

---

### Database Schema

**Expense Categories Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| parent_id | uuid | Foreign key, nullable |
| name | string | Not null |
| description | text | Nullable |
| color | string | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- parent_id
- Composite: teacher_id + name (unique)

**Foreign Keys:**
- teacher_id references teachers(id)
- parent_id references expense_categories(id)

---

**Expenses Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| category_id | uuid | Foreign key, not null |
| student_id | uuid | Foreign key, nullable |
| amount | decimal(10,2) | Not null |
| date | date | Not null |
| description | text | Nullable |
| receipt_url | string | Nullable |
| is_recurring | boolean | Default: false |
| recurring_frequency | string | Nullable |
| vendor | string | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- category_id
- student_id
- date
- amount

**Foreign Keys:**
- teacher_id references teachers(id)
- category_id references expense_categories(id)
- student_id references students(id)

**Valid Recurring Frequencies:**
- monthly
- quarterly
- annually

---

### Expense Category Model

**Location:** `app/models/expense_category.rb`

**Associations:**
- belongs_to :teacher
- belongs_to :parent, class_name: 'ExpenseCategory', optional: true
- has_many :children, class_name: 'ExpenseCategory', foreign_key: 'parent_id', dependent: :destroy
- has_many :expenses, dependent: :restrict_with_error

**Validations:**
- teacher: must exist
- name: presence, uniqueness scoped to teacher_id
- color: format (hex color), optional

**Scopes:**
- top_level: `where(parent_id: nil)`
- active: `joins(:expenses).distinct`

**Instance Methods:**

`full_path`
- Return category path (e.g., "Curriculum > Math")
- Recursively build from parent

`total_expenses`
- Sum of all expenses in this category
- Include expenses from child categories

---

### Expense Model

**Location:** `app/models/expense.rb`

**Associations:**
- belongs_to :teacher
- belongs_to :category, class_name: 'ExpenseCategory'
- belongs_to :student, optional: true

**Validations:**
- teacher: must exist
- category: must exist
- amount: presence, numericality (greater than 0)
- date: presence
- recurring_frequency: inclusion in valid values (if is_recurring is true)

**Scopes:**
- between_dates: `where('date >= ? AND date <= ?', start_date, end_date)`
- by_category: `where(category_id: category_id)`
- for_student: `where(student_id: student_id)`
- recurring: `where(is_recurring: true)`
- one_time: `where(is_recurring: false)`

**Instance Methods:**

`recurring?`
- Return is_recurring value

`annual_cost`
- If not recurring, return amount
- If monthly, return amount * 12
- If quarterly, return amount * 4
- If annually, return amount

`category_path`
- Return full category path from category.full_path

---

### Expense Categories Controller

**Location:** `app/controllers/api/v1/expense_categories_controller.rb`

**Configuration:**
- All actions require authentication
- Verify category belongs to current_teacher

**Endpoints:**

`GET /api/v1/expense_categories`
- Return all categories for current_teacher
- Include hierarchy (children nested)
- Include expense totals per category
- No pagination (categories typically limited)

`GET /api/v1/expense_categories/:id`
- Find category by ID
- Verify belongs to current_teacher
- Include children
- Include expense total
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/expense_categories`
- Accept: name, description, color, parent_id
- Link to current_teacher
- Return 201 with created category
- Return 422 with validation errors

`PATCH /api/v1/expense_categories/:id`
- Find category by ID
- Verify belongs to current_teacher
- Accept: name, description, color, parent_id
- Update fields
- Return 200 with updated category
- Return 404 if not found
- Return 403 if wrong teacher
- Return 422 with validation errors

`DELETE /api/v1/expense_categories/:id`
- Find category by ID
- Verify belongs to current_teacher
- Return 422 if category has expenses
- Delete category
- Return 204
- Return 404 if not found
- Return 403 if wrong teacher

---

### Expenses Controller

**Location:** `app/controllers/api/v1/expenses_controller.rb`

**Configuration:**
- All actions require authentication
- Verify expense belongs to current_teacher

**Endpoints:**

`GET /api/v1/expenses`
- Optional: start_date, end_date, category_id, student_id, recurring filters
- Sort by date (desc) by default
- Include pagination
- Calculate totals
- Return expenses with category and student info

**Query Parameters:**
- start_date: ISO 8601 date
- end_date: ISO 8601 date
- category_id: filter by category
- student_id: filter by student
- recurring: true/false filter
- page: page number
- limit: items per page

`GET /api/v1/expenses/:id`
- Find expense by ID
- Verify belongs to current_teacher
- Include receipt URL
- Include category and student info
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/expenses`
- Accept: category_id, student_id, amount, date, description, is_recurring, recurring_frequency, vendor, receipt (file upload)
- Link to current_teacher
- Upload receipt to cloud storage if provided
- Return 201 with created expense
- Return 422 with validation errors

`PATCH /api/v1/expenses/:id`
- Find expense by ID
- Verify belongs to current_teacher
- Accept: category_id, student_id, amount, date, description, is_recurring, recurring_frequency, vendor, receipt (file upload)
- Update fields
- Handle receipt upload/replacement
- Return 200 with updated expense
- Return 404 if not found
- Return 403 if wrong teacher
- Return 422 with validation errors

`DELETE /api/v1/expenses/:id`
- Find expense by ID
- Verify belongs to current_teacher
- Delete receipt from cloud storage if exists
- Delete expense
- Return 204
- Return 404 if not found
- Return 403 if wrong teacher

`GET /api/v1/expenses/reports`
- Accept: start_date, end_date, group_by (category, month, student)
- Generate expense report
- Calculate totals and breakdowns
- Include recurring expense projections
- Return aggregated data

---

### Testing Requirements

**Expense Category Model:**
- Cannot create without name
- Name must be unique per teacher
- Can nest categories
- Full path calculation works
- Total expenses calculation works
- Cannot delete category with expenses

**Expense Model:**
- Cannot create without required fields
- Amount must be positive
- Date must be valid
- Annual cost calculation for recurring expenses works
- Category path lookup works

**Controllers:**
- Can create categories
- Can create expenses
- Can upload receipts
- Can filter expenses by multiple criteria
- Can generate expense reports
- Totals calculate correctly
- Recurring expense projection works
- Requires authentication
- Teachers only see own data

---

### Validation Checklist

- [ ] Can create expense categories
- [ ] Can nest categories
- [ ] Can record expenses
- [ ] Can upload receipts
- [ ] Can filter and search expenses
- [ ] Can generate expense reports
- [ ] Totals calculate correctly
- [ ] Recurring expense projection works
- [ ] Category hierarchy works
- [ ] All expense tests pass

---

## Phase 7: Lesson Planning

### Technical Requirements

**Data Model:**
- Subjects organize lesson plans
- Lesson plans contain objectives, materials, activities
- Support for file/link resources
- Track when lessons are taught
- Enable lesson duplication for reuse

---

### Architecture

```
┌──────────────┐
│   teachers   │
└──────┬───────┘
       │
       ├─── has_many ──────┐
       │                   │
       ▼                   ▼
┌──────────────┐     ┌──────────────┐
│  subjects    │     │lesson_plans  │
│──────────────│     │──────────────│
│ teacher_id   │     │ teacher_id   │
│ name         │◄────│ subject_id   │
│ description  │     │ title        │
│ color        │     │ objectives   │
└──────────────┘     │ materials    │
                     │ activities   │
                     │ assessment   │
                     │ resources    │
                     │ date_taught  │
                     │ notes        │
                     └──────────────┘
```

---

### Database Schema

**Subjects Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| name | string | Not null |
| description | text | Nullable |
| color | string | Nullable |
| grade_level | string | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- Composite: teacher_id + name (unique)

**Foreign Keys:**
- teacher_id references teachers(id)

---

**Lesson Plans Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| subject_id | uuid | Foreign key, not null |
| title | string | Not null |
| objectives | text | Not null |
| materials | text | Nullable |
| activities | text | Nullable |
| assessment | text | Nullable |
| resources | jsonb | Nullable |
| date_taught | date | Nullable |
| notes | text | Nullable |
| duration | integer | Minutes, nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- subject_id
- date_taught

**Foreign Keys:**
- teacher_id references teachers(id)
- subject_id references subjects(id)

**Resources JSON Structure:**
```json
[
  {
    "type": "file",
    "name": "Worksheet.pdf",
    "url": "https://..."
  },
  {
    "type": "link",
    "name": "Educational Video",
    "url": "https://youtube.com/..."
  }
]
```

---

### Subject Model

**Location:** `app/models/subject.rb`

**Associations:**
- belongs_to :teacher
- has_many :lesson_plans, dependent: :restrict_with_error

**Validations:**
- teacher: must exist
- name: presence, uniqueness scoped to teacher_id
- color: format (hex color), optional

**Scopes:**
- by_grade: `where(grade_level: level)`
- alphabetical: `order(:name)`

**Instance Methods:**

`lesson_count`
- Return count of lesson_plans

`completed_lesson_count`
- Return count of lesson_plans where date_taught is not null

---

### Lesson Plan Model

**Location:** `app/models/lesson_plan.rb`

**Associations:**
- belongs_to :teacher
- belongs_to :subject

**Validations:**
- teacher: must exist
- subject: must exist
- title: presence, max length 200
- objectives: presence

**Scopes:**
- upcoming: `where(date_taught: nil).order(:created_at)`
- completed: `where.not(date_taught: nil).order(date_taught: :desc)`
- by_subject: `where(subject_id: subject_id)`
- taught_between: `where('date_taught >= ? AND date_taught <= ?', start_date, end_date)`

**Instance Methods:**

`mark_taught!(date = Date.current)`
- Set date_taught to provided date
- Save

`completed?`
- Return true if date_taught present

`duplicate`
- Create copy of lesson plan
- Clear date_taught
- Clear notes
- Copy all other fields including resources
- Return new unsaved lesson plan

---

### Subjects Controller

**Location:** `app/controllers/api/v1/subjects_controller.rb`

**Configuration:**
- All actions require authentication
- Verify subject belongs to current_teacher

**Endpoints:**

`GET /api/v1/subjects`
- Return all subjects for current_teacher
- Include lesson counts
- Sort alphabetically by default
- Support pagination

`GET /api/v1/subjects/:id`
- Find subject by ID
- Verify belongs to current_teacher
- Include lesson count
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/subjects`
- Accept: name, description, color, grade_level
- Link to current_teacher
- Return 201 with created subject
- Return 422 with validation errors

`PATCH /api/v1/subjects/:id`
- Find subject by ID
- Verify belongs to current_teacher
- Accept: name, description, color, grade_level
- Update fields
- Return 200 with updated subject
- Return 404 if not found
- Return 403 if wrong teacher
- Return 422 with validation errors

`DELETE /api/v1/subjects/:id`
- Find subject by ID
- Verify belongs to current_teacher
- Return 422 if subject has lesson plans
- Delete subject
- Return 204
- Return 404 if not found
- Return 403 if wrong teacher

---

### Lesson Plans Controller

**Location:** `app/controllers/api/v1/lesson_plans_controller.rb`

**Configuration:**
- All actions require authentication
- Verify lesson plan belongs to current_teacher

**Endpoints:**

`GET /api/v1/lesson_plans`
- Optional: subject_id, taught (true/false), start_date, end_date filters
- Sort by date_taught (desc) then created_at (desc)
- Include subject info
- Support pagination

**Query Parameters:**
- subject_id: filter by subject
- taught: true/false filter on date_taught
- start_date: taught after this date
- end_date: taught before this date
- page: page number
- limit: items per page

`GET /api/v1/lesson_plans/:id`
- Find lesson plan by ID
- Verify belongs to current_teacher
- Include resources
- Include subject info
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/lesson_plans`
- Accept: subject_id, title, objectives, materials, activities, assessment, notes, duration, resources (array), date_taught
- Link to current_teacher
- Handle file uploads in resources
- Return 201 with created lesson plan
- Return 422 with validation errors

`PATCH /api/v1/lesson_plans/:id`
- Find lesson plan by ID
- Verify belongs to current_teacher
- Accept: subject_id, title, objectives, materials, activities, assessment, notes, duration, resources (array), date_taught
- Update fields
- Handle file uploads
- Return 200 with updated lesson plan
- Return 404 if not found
- Return 403 if wrong teacher
- Return 422 with validation errors

`DELETE /api/v1/lesson_plans/:id`
- Find lesson plan by ID
- Verify belongs to current_teacher
- Delete uploaded resource files from cloud storage
- Delete lesson plan
- Return 204
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/lesson_plans/:id/duplicate`
- Find lesson plan by ID
- Verify belongs to current_teacher
- Create copy without date_taught and notes
- Copy resources (files remain in storage, reference copied)
- Return 201 with new lesson plan
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/lesson_plans/:id/mark_taught`
- Find lesson plan by ID
- Verify belongs to current_teacher
- Accept: date (optional, defaults to today)
- Set date_taught
- Return 200 with updated lesson plan
- Return 404 if not found
- Return 403 if wrong teacher

`POST /api/v1/lesson_plans/:id/resources`
- Find lesson plan by ID
- Verify belongs to current_teacher
- Accept: file upload or link (name, url)
- Upload file to cloud storage if file
- Add to resources array
- Return 200 with updated resources list
- Return 404 if not found
- Return 403 if wrong teacher

---

### Testing Requirements

**Subject Model:**
- Cannot create without name
- Name must be unique per teacher
- Can have multiple lesson plans
- Lesson count accurate
- Cannot delete subject with lesson plans

**Lesson Plan Model:**
- Cannot create without required fields
- Can mark as taught
- Duplication creates independent copy
- Resources stored correctly in JSON
- Completed status works

**Controllers:**
- Can create subjects and lessons
- Can upload resource files
- Can filter lessons by multiple criteria
- Can duplicate lessons
- Taught date updates correctly
- Resource management works
- Requires authentication
- Teachers only see own data

---

### Validation Checklist

- [ ] Subject model created
- [ ] Lesson plan model created
- [ ] Can create subjects
- [ ] Can create lesson plans
- [ ] Can upload resource files
- [ ] Can mark lessons as taught
- [ ] Can duplicate lessons
- [ ] Can filter and search lessons
- [ ] File uploads work
- [ ] Resources JSON structure correct
- [ ] All lesson planning tests pass

---

## Phase 8: Production Preparation

### Technical Requirements

**Error Handling:**
- Global exception handling
- Structured logging
- Error tracking service integration
- Request ID tracking

**Rate Limiting:**
- Per-IP rate limiting
- Per-user rate limiting
- Configurable limits
- Rate limit headers in responses

**Performance:**
- Database query optimization
- Caching strategy
- Background job processing
- Connection pooling

**Security:**
- Security headers
- Input validation
- HTTPS enforcement
- SQL injection prevention
- XSS prevention

**Monitoring:**
- Application metrics
- Business metrics
- Health checks
- Alerting

---

### Error Handling Implementation

**Global Exception Handler:**

**Location:** `app/controllers/concerns/exception_handler.rb`

**Functionality:**
- Rescue from all StandardError
- Log error with context (user, request, timestamp)
- Generate request ID for tracking
- Return generic error to user (never expose stack trace)
- Integrate with error tracking service
- Alert on 500 errors

**Custom Error Classes:**
- Create in `app/errors/`
- AuthenticationError
- AuthorizationError
- ValidationError
- NotFoundError
- RateLimitError

**Error Logging:**
- Use Rails logger
- Include request ID
- Include user ID
- Include timestamp
- Include error class and message
- Include backtrace (in logs only, never to user)

---

### Rate Limiting Implementation

**Strategy:**
- Use middleware (Rack::Attack or similar)
- Track by IP address
- Track by user ID (if authenticated)
- Different limits for different endpoints

**Default Limits:**
- Unauthenticated: 60 requests per hour per IP
- Authenticated: 1000 requests per hour per user
- Login endpoint: 5 requests per 15 minutes per IP
- Signup endpoint: 3 requests per hour per IP

**Response Headers:**
- X-RateLimit-Limit: Total allowed
- X-RateLimit-Remaining: Remaining in window
- X-RateLimit-Reset: When limit resets (Unix timestamp)

**Response on Limit Exceeded:**
- Status: 429 Too Many Requests
- Body: JSON with error message and retry_after

---

### Health Check Implementation

**Endpoints:**

`GET /health`
- No authentication required
- Check Rails is responding
- Return 200 with { status: "ok" }

`GET /health/detailed`
- Authentication required (or internal only)
- Check database connection
- Check Redis connection
- Check external service dependencies
- Return status of each component
- Return 200 if all healthy
- Return 503 if any component unhealthy

**Response Format:**
```json
{
  "status": "ok",
  "timestamp": "2024-01-20T12:00:00Z",
  "components": {
    "database": "ok",
    "redis": "ok",
    "storage": "ok"
  }
}
```

---

### Performance Optimization

**Database Optimization:**

**Missing Indexes:**
- Run query analysis
- Add indexes where needed
- Focus on foreign keys and frequently queried columns

**N+1 Query Prevention:**
- Use includes/joins in controllers
- Eager load associations
- Example: `Assignment.includes(:students).where(teacher_id: teacher.id)`

**Query Optimization:**
- Use select to limit columns
- Use pluck for single column
- Add database views for complex queries
- Use counter caches where appropriate

**Connection Pooling:**
- Configure pool size in database.yml
- Recommended: 5-10 connections per process
- Monitor connection usage

---

**Caching Strategy:**

**What to Cache:**
- Authentication lookups (teacher by ID)
- Expensive calculations (GPA, statistics)
- Frequently accessed data (categories, subjects)

**Cache Keys:**
- Include record ID and updated_at
- Example: `teacher-#{id}-#{updated_at.to_i}`
- Invalidate on update

**TTL Settings:**
- Authentication: 5 minutes
- Calculations: 1 hour
- Reference data: 24 hours

**Implementation:**
- Use Rails.cache
- Redis as cache store
- Fragment caching for complex responses

---

**Background Jobs:**

**Move to Background:**
- Email sending
- File processing
- Report generation
- Data exports

**Job Processing:**
- Use Sidekiq or similar
- Configure queues by priority
- Set retry logic
- Monitor job queue

---

### Security Hardening

**Security Headers:**

Configure in `config/application.rb`:
- X-Frame-Options: DENY
- X-Content-Type-Options: nosniff
- X-XSS-Protection: 1; mode=block
- Strict-Transport-Security: max-age=31536000
- Content-Security-Policy: default-src 'self'

**HTTPS Enforcement:**
- Force SSL in production
- Redirect HTTP to HTTPS
- Configure in Rails: `config.force_ssl = true`

**Input Validation:**
- Strong parameters in all controllers
- Validate all user input
- Sanitize text fields
- Prevent mass assignment

**SQL Injection Prevention:**
- Use parameterized queries (ActiveRecord does this)
- Never interpolate user input into SQL
- Use where with hash or array syntax

**XSS Prevention:**
- Rails escapes output by default
- Don't use html_safe on user input
- Sanitize rich text if needed

**Rate Limiting:**
- Implement as described above
- Protect authentication endpoints especially
- Monitor for brute force attempts

**Token Security:**
- Short access token expiry (1 hour)
- Refresh token rotation (optional)
- Revoke refresh tokens on logout
- Store tokens securely (httpOnly cookies for web)

---

### Monitoring Implementation

**Application Metrics:**

Track:
- Request rate (requests per second)
- Response time (p50, p95, p99)
- Error rate (errors per second, percentage)
- Database query time
- Cache hit rate

**Business Metrics:**

Track:
- User registrations per day
- Active users (DAU, WAU, MAU)
- API calls per user
- Most used endpoints
- Average session duration

**Health Monitoring:**

Monitor:
- CPU usage
- Memory usage
- Disk space
- Database connections
- Background job queue size

**Alerting:**

Alert on:
- Error rate spike (> 5% of requests)
- Response time spike (p95 > 2 seconds)
- Database connection pool exhausted
- Background job queue growing
- Service unavailable (500 errors)
- High CPU/memory usage

**Tools:**
- APM: New Relic, DataDog, or similar
- Error tracking: Sentry, Bugsnag, or similar
- Logging: LogDNA, Papertrail, or similar

---

### Documentation

**API Documentation:**

Create documentation including:
- All endpoints with HTTP methods
- Request parameters and body schemas
- Response schemas and examples
- Authentication requirements
- Error codes and messages
- Rate limiting information

**Format:**
- OpenAPI/Swagger specification
- Interactive documentation (Swagger UI)
- Code examples in multiple languages

---

**Deployment Documentation:**

Document:
- Environment setup
- Required environment variables
- Database setup and migrations
- SSL certificate setup
- Deployment process
- Rollback procedure
- Monitoring setup
- Backup and restore procedures

---

### Deployment Preparation

**Database:**
- Production database provisioned
- Automated backups configured (daily at minimum)
- Backup restoration tested
- Connection pooling configured
- Read replicas (if needed for scale)

**Environment:**
- All environment variables documented
- Secrets stored securely (not in code)
- SSL certificates obtained and configured
- DNS configured

**Hosting:**
- Application server configured
- Database server configured (or managed service)
- Redis server configured (or managed service)
- Cloud storage configured (S3 or equivalent)
- CDN configured (if needed)

**CI/CD:**
- Automated testing on push
- Automated deployment on merge to main
- Rollback mechanism
- Deployment notifications

**Monitoring:**
- APM configured
- Error tracking configured
- Logging configured
- Alerts configured
- Dashboards created

---

### Testing Requirements

**Error Handling:**
- Unhandled exceptions return 500
- Error logs contain needed context
- No sensitive data in error responses
- Request IDs tracked correctly

**Rate Limiting:**
- Requests limited correctly
- Headers show remaining requests
- 429 returned when exceeded
- Different limits for different endpoints

**Health Checks:**
- Returns 200 when healthy
- Returns error when database unavailable
- No authentication required for basic health
- Detailed health requires authentication

**Performance:**
- No N+1 queries in controllers
- Indexes improve query speed
- Caching reduces database load
- Background jobs process correctly

**Security:**
- Security headers present
- HTTPS enforced in production
- SQL injection prevented
- XSS prevented
- Authorization checks on all protected endpoints
- Rate limiting protects against brute force

---

### Validation Checklist

- [ ] Global error handling implemented
- [ ] Errors logged with context
- [ ] Error tracking service integrated
- [ ] Rate limiting implemented
- [ ] Rate limit headers returned
- [ ] Health check endpoints created
- [ ] Database indexes optimized
- [ ] N+1 queries eliminated
- [ ] Caching implemented
- [ ] Background jobs configured
- [ ] Security headers configured
- [ ] HTTPS enforced
- [ ] Input validation thorough
- [ ] SQL injection prevented
- [ ] XSS prevented
- [ ] Monitoring configured
- [ ] Alerts configured
- [ ] API documentation complete
- [ ] Deployment documentation complete
- [ ] Production environment configured
- [ ] Backups configured and tested
- [ ] CI/CD pipeline configured
- [ ] All production tests pass

---

## Conclusion

This guide provides the technical specifications for building the Homeschool Master Rails API. Each phase builds upon the previous, creating a complete backend system ready for production deployment.

**Implementation order is important** - complete phases sequentially as later phases depend on earlier infrastructure.

**Testing at each phase** ensures quality and prevents regressions as features are added.

**Production preparation** is critical for a reliable, secure, and performant API.

---

## Appendix: Technical Reference

**Rails Documentation:**
- https://guides.rubyonrails.org
- https://api.rubyonrails.org

**Ruby Documentation:**
- https://ruby-doc.org
- https://rubystyle.guide

**PostgreSQL:**
- https://www.postgresql.org/docs

**Testing:**
- https://rspec.info

**Security:**
- https://owasp.org

**Deployment:**
- https://12factor.net
