# Homeschool Master - Rails API Implementation Guide

Technical implementation specification for the Homeschool Master Rails API backend.

**Philosophy:** Each phase is self-contained. At the end of every phase, the application compiles, all tests pass.
---

## Table of Contents

- [Phase 0: Project Foundation](#phase-0-project-foundation)
- [Phase 1: Authentication System](#phase-1-authentication-system)
- [Phase 2: Student Management](#phase-2-student-management)
- [Phase 3: Subjects & Calendar](#phase-3-subjects--calendar)
- [Phase 4: Assignments & Tasks](#phase-4-assignments--tasks)
- [Phase 5: Report Cards](#phase-5-report-cards)
- [Phase 6: Expense Tracking](#phase-6-expense-tracking)
- [Phase 7: Lesson Planning](#phase-7-lesson-planning)
- [Phase 8: Production Preparation](#phase-8-production-preparation)

---

## Phase 0: Project Foundation

### Goal

Set up a working Rails API with PostgreSQL, UUID primary keys, CORS, linting, and testing infrastructure. At the end of this phase, the server starts and responds to a health check.

---

### Dependencies

| Gem | Purpose | Why This Phase | Why This Gem |
|-----|---------|----------------|--------------|
| rails | Web framework | Foundation of the entire API. | Only choice for a Rails API. |
| pg | PostgreSQL adapter | Need to connect to PostgreSQL database. | Only maintained PostgreSQL adapter for Rails. |
| puma | Web server | Need to serve HTTP requests. | Rails default, handles concurrent requests well. Falcon is faster but less battle-tested. |
| rack-cors | CORS middleware | Mobile app runs on different origin than API—browsers block cross-origin requests without CORS headers. | Standard choice, simple configuration. |
| dotenv-rails | Environment variable loading | Need to keep secrets out of code, load from .env files in development. | Simple and widely used. Alternatives like figaro work but dotenv is more common. |
| rspec-rails | Testing framework | Need to write tests from day one. | More expressive than Minitest, better community resources. Minitest is lighter but less flexible. |
| factory_bot_rails | Test data factories | Need to create test records without fixtures. | Industry standard for Rails. Fabrication is similar but less popular. |
| faker | Fake data generation | Need realistic test data (names, emails, etc.). | Most comprehensive fake data library. |
| rubocop-rails | Code linting | Enforce consistent code style from the start. | Standard Ruby linter with Rails-specific rules. |
| shoulda-matchers | Model test helpers | Simplifies testing validations and associations. | Makes model specs much cleaner. No real alternative. |
| database_cleaner-active_record | Test database cleanup | Need clean database state between tests. | Standard choice. Rails transactional tests work but can have edge cases. |

---

### PostgreSQL Setup

- Enable UUID extension (pgcrypto) via migration
- Configure generators to use UUID primary keys by default
- Create development and test databases

---

### CORS Configuration

- Allow origins: localhost:8081, 127.0.0.1:8081
- Allow methods: GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD
- Allow headers: any
- Expose headers: Authorization

---

### Environment Variables

| Variable | Description |
|----------|-------------|
| DATABASE_URL | PostgreSQL connection string |
| APP_HOST | API host URL (http://localhost:3000) |
| FRONTEND_URL | Mobile app URL (http://localhost:8081) |

---

### Directory Structure

```
app/
  controllers/
    api/
      v1/
        base_controller.rb
  models/
  services/
```

---

### Base Controller

**Location:** `app/controllers/api/v1/base_controller.rb`

Phase 0 version has response helpers only. No authentication (added in Phase 1).

**Required Capabilities:**

| Response Type | Status | Structure |
|---------------|--------|-----------|
| Success | 200 | `{ success: true, data: ..., meta?: ... }` |
| Created | 201 | `{ success: true, data: ... }` |
| No Content | 204 | Empty |
| Error | varies | `{ success: false, error: { code: ..., message: ..., details?: ... } }` |
| Unauthorized | 401 | Error with code 'UNAUTHORIZED' |
| Forbidden | 403 | Error with code 'FORBIDDEN' |
| Not Found | 404 | Error with code 'NOT_FOUND' |
| Validation Error | 422 | Error with code 'VALIDATION_ERROR', details from model errors |

---

### Routes

Phase 0 only defines:
- Health check: GET /health → returns 200 OK
- Empty api/v1 namespace (placeholder for future routes)

---

### Testing Requirements

**Project Setup:**
- Rails server starts without errors
- PostgreSQL connection works
- UUID extension enabled (pgcrypto)
- All gems installed

**CORS:**
- Cross-origin requests from localhost:8081 succeed
- Correct headers exposed

**Health Endpoint:**
- GET /health returns 200 OK

**Base Controller Response Helpers:**
- Success responses return correct structure with 200
- Created responses return 201 status
- Error responses return correct error structure
- Unauthorized returns 401
- Forbidden returns 403
- Not found returns 404
- Validation errors return 422 with error details

---

### Validation Checklist

- [ ] Rails project initialized in API mode
- [ ] PostgreSQL configured and databases created
- [ ] UUID extension migration created and run
- [ ] Generators configured for UUID primary keys
- [ ] CORS configured for localhost:8081
- [ ] Environment variables file created
- [ ] RSpec installed and configured
- [ ] RuboCop installed and configured
- [ ] Base controller with response helpers created
- [ ] Server starts without errors
- [ ] GET /health returns 200 OK
- [ ] All tests pass

---

## Phase 1: Authentication System

### Goal

Implement complete authentication with JWT tokens, teacher registration, login, logout, password reset, and email verification. At the end of this phase, users can register, log in, and access protected endpoints.

---

### Dependencies

| Gem | Purpose | Why This Phase | Why This Gem |
|-----|---------|----------------|--------------|
| bcrypt | Password hashing for has_secure_password | Teacher model needs secure password storage for registration and login. | Rails' built-in has_secure_password expects bcrypt. It's the standard—no real alternatives here. |
| jwt | Encode/decode JSON Web Tokens | Authentication requires issuing access tokens and refresh tokens to clients. | The most popular Ruby JWT library. jose is an alternative but more complex and overkill for simple token auth. |

---

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| JWT_SECRET_KEY | Secret for signing tokens | (required) |
| JWT_ACCESS_TOKEN_EXPIRY | Access token lifetime in seconds | 3600 |
| JWT_REFRESH_TOKEN_EXPIRY | Refresh token lifetime in seconds | 2592000 |

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
- teacher_id references teachers(id) with ON DELETE CASCADE

---

### JWT Service

**Location:** `app/services/jwt_service.rb`

**Required Capabilities:**
- Encode a payload into a signed JWT with expiration (default from ENV)
- Encode a refresh token with teacher_id, type identifier, and unique jti
- Decode and verify a token, returning the payload or nil on any error
- Decode a refresh token and verify it's the correct type

**Configuration:**
- Algorithm: HS256
- Claims: exp (expiration), iat (issued at)
- Refresh tokens also include: jti (unique identifier), type identifier

---

### Teacher Model

**Behaviors:**
- Secure password storage (Rails built-in)
- Email always stored lowercase
- Email verification token generated automatically on creation
- Has many refresh tokens (destroyed when teacher deleted)

**Validations:**

| Field | Rules |
|-------|-------|
| first_name | presence, max length 100 |
| last_name | presence, max length 100 |
| nickname | max length 100 (optional) |
| email | presence, uniqueness (case insensitive), valid format |
| password | minimum 8 characters (on create or when present) |
| phone | max length 20 (optional) |

**Required Capabilities:**
- Query active teachers only
- Query verified teachers only
- Look up teacher by email (case insensitive)
- Get teacher's full name and display name (nickname if set)
- Verify a teacher's email (sets timestamp, clears token)
- Generate password reset token (valid for 2 hours)
- Check if reset token is still valid
- Clear password reset token after use

---

### Refresh Token Model

**Behaviors:**
- Belongs to a teacher
- Destroyed when teacher deleted (via foreign key cascade)

**Validations:**

| Field | Rules |
|-------|-------|
| token | presence, uniqueness |
| jti | presence, uniqueness |
| expires_at | presence |

**Required Capabilities:**
- Query only active tokens (not revoked and not expired)
- Find a valid token by its token string
- Check if a token is expired
- Check if a token is revoked
- Check if a token is valid (not expired and not revoked)
- Revoke a single token
- Revoke all tokens for a specific teacher

---

### Base Controller (Updated)

Add authentication to the base controller.

**New Behaviors:**
- Authenticate requests before all actions
- Make current teacher available to child controllers
- Child controllers can skip authentication for specific actions

**Authentication Logic:**
1. Extract token from Authorization header (Bearer scheme)
2. Decode token using JWT service
3. Find teacher by decoded teacher_id
4. Verify teacher exists and is active
5. Return 401 for any failure

---

### Routes

Add to api/v1 namespace under auth namespace:

| Method | Path | Controller#Action |
|--------|------|-------------------|
| POST | /auth/register | authentication#register |
| POST | /auth/login | authentication#login |
| POST | /auth/refresh | authentication#refresh |
| POST | /auth/logout | authentication#logout |
| POST | /auth/password/reset-request | passwords#reset_request |
| POST | /auth/password/reset | passwords#reset |
| POST | /auth/password/change | passwords#change |
| POST | /auth/email/verify | email_verification#verify |
| POST | /auth/email/resend-verification | email_verification#resend |

---

### Authentication Controller

**Location:** `app/controllers/api/v1/auth/authentication_controller.rb`

**Skip authentication for:** register, login, refresh, logout

**Endpoints:**

| Endpoint | Request | Success | Errors |
|----------|---------|---------|--------|
| register | first_name, last_name, email, password | 201, teacher data (no password_digest) | 422 validation errors |
| login | email, password | 200, access_token + refresh_token | 401 invalid credentials, 401 inactive |
| refresh | refresh_token | 200, new access_token | 401 invalid/expired/revoked |
| logout | refresh_token | 204 | 401 invalid token |

**Login Flow:**
1. Find teacher by email
2. Verify password
3. Check account is active
4. Generate access token with teacher_id
5. Generate refresh token
6. Store refresh token in database (token string, jti, expires_at)
7. Return both tokens

**Refresh Flow:**
1. Decode refresh token
2. Find RefreshToken record by jti
3. Verify not revoked and not expired
4. Generate new access token
5. Return new access token

---

### Passwords Controller

**Location:** `app/controllers/api/v1/auth/passwords_controller.rb`

**Skip authentication for:** reset_request, reset

**Endpoints:**

| Endpoint | Request | Success | Errors |
|----------|---------|---------|--------|
| reset_request | email | 200 (always, for security) | - |
| reset | token, password | 200 | 400 invalid/expired token |
| change | current_password, new_password | 200 | 401 wrong current password |

**Note:** change requires authentication

---

### Email Verification Controller

**Location:** `app/controllers/api/v1/auth/email_verification_controller.rb`

**Skip authentication for:** verify

**Endpoints:**

| Endpoint | Request | Success | Errors |
|----------|---------|---------|--------|
| verify | token | 200 | 400 invalid token |
| resend | - | 200 | - |

**Note:** resend requires authentication

---

### Testing Requirements

**JWT Service:**
- Encodes payload into valid JWT string
- Decoded token contains original payload
- Access tokens expire after configured time
- Refresh tokens expire after configured time
- Refresh tokens include unique jti
- Returns nil for expired tokens
- Returns nil for invalid signatures
- Returns nil for malformed tokens

**Teacher Model:**
- Validates presence of required fields
- Email must be unique (case insensitive)
- Email must match valid format
- Password minimum 8 characters
- Password is hashed (not stored as plain text)
- Email downcased before save
- Verification token generated on create
- Email lookup is case insensitive
- Full name and display name work correctly
- Email verification flow works
- Password reset token generation and validation works
- Scopes filter correctly

**Refresh Token Model:**
- Validates presence of required fields
- Token and jti must be unique
- Active scope excludes revoked and expired
- Revocation works for single token and all tokens for teacher

**Base Controller Authentication:**
- Returns 401 without token
- Returns 401 with invalid token
- Returns 401 with expired token
- Returns 401 for inactive teacher
- Sets current teacher with valid token

**Authentication Controller:**
- Register creates teacher (201)
- Register returns 422 for validation errors
- Login returns tokens (200)
- Login returns 401 for bad credentials
- Login returns 401 for inactive account
- Refresh returns new access token (200)
- Refresh returns 401 for invalid/revoked token
- Logout revokes token (204)

**Passwords Controller:**
- Reset request always returns 200
- Reset with valid token updates password
- Reset with expired token returns 400
- Change with correct password succeeds
- Change with wrong password returns 401

**Email Verification Controller:**
- Verify with valid token succeeds
- Verify with invalid token returns 400
- Resend generates new token

---

### Validation Checklist

- [ ] bcrypt and jwt gems installed
- [ ] JWT environment variables configured
- [ ] Teachers migration created and run
- [ ] Refresh tokens migration created and run
- [ ] JWT service implemented and tested
- [ ] Teacher model with validations and required capabilities
- [ ] Refresh token model with validations and required capabilities
- [ ] Base controller authentication working
- [ ] Authentication controller endpoints working
- [ ] Passwords controller endpoints working
- [ ] Email verification controller endpoints working
- [ ] All tests pass

---

## Phase 2: Student Management

### Goal

Add student profiles linked to teachers. Teachers can create, view, update, and delete their students.

---

### Dependencies

| Gem | Purpose | Why This Phase | Why This Gem |
|-----|---------|----------------|--------------|
| kaminari | Pagination for ActiveRecord collections | Students index is the first list endpoint—can't return unbounded results to mobile clients. | Clean API, well-maintained, plays nice with Rails. Pagy is faster but more manual setup. Will-paginate is older and less actively maintained. |

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
| date_of_birth | date | Nullable |
| grade_level | string | Nullable |
| profile_image_url | string | Nullable |
| notes | text | Nullable |
| is_active | boolean | Default: true |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- is_active

**Foreign Keys:**
- teacher_id references teachers(id) with ON DELETE CASCADE

---

### Model Updates

**Teacher:** Add association to students (destroyed when teacher deleted)

---

### Student Model

**Behaviors:**
- Belongs to a teacher
- Destroyed when teacher deleted (via foreign key cascade)

**Validations:**

| Field | Rules |
|-------|-------|
| first_name | presence, max length 100 |
| last_name | presence, max length 100 |
| nickname | max length 100 (optional) |
| grade_level | max length 50 (optional) |

**Required Capabilities:**
- Query active students only
- Get student's full name and display name (nickname if set)
- Calculate student's age from date of birth

---

### Base Controller (Updated)

Add pagination helper.

**Pagination Behavior:**
- Accept page and limit params (default limit: 20)
- Return paginated data with metadata: page, limit, total, total_pages, has_next, has_prev

---

### Routes

Add to api/v1 namespace:
- resources :students (full CRUD)

---

### Students Controller

**Location:** `app/controllers/api/v1/students_controller.rb`

**Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| GET | /students | List current teacher's students (paginated) |
| GET | /students/:id | Show student details |
| POST | /students | Create student for current teacher |
| PATCH | /students/:id | Update student |
| DELETE | /students/:id | Delete student |

**Index Filters:**
- is_active (boolean)
- Pagination: page, limit

**Authorization:** All actions scoped to current teacher's students only

---

### Testing Requirements

**Student Model:**
- Validates presence of first_name, last_name
- Belongs to teacher
- Full name and display name work correctly
- Age calculates correctly from date_of_birth
- Active scope filters correctly

**Students Controller:**
- Index returns only current teacher's students
- Index paginates correctly
- Index filters by is_active
- Show returns student details
- Show returns 404 for other teacher's student
- Create creates student for current teacher
- Create returns 422 for validation errors
- Update modifies student
- Update returns 404 for other teacher's student
- Delete removes student
- Delete returns 404 for other teacher's student

**Pagination Helper:**
- Returns correct metadata structure
- Respects page and limit params
- has_next/has_prev correct at boundaries

---

### Validation Checklist

- [ ] kaminari gem installed
- [ ] Students migration created and run
- [ ] Student model with validations and required capabilities
- [ ] Teacher association to students added
- [ ] Pagination helper added to base controller
- [ ] Students controller implemented
- [ ] Routes configured
- [ ] All tests pass

---

## Phase 3: Subjects & Calendar

### Goal

Add subjects and calendar events. Teachers can create subjects (like "Math" or "Science") and schedule events on the calendar.

---

### Dependencies

None—all required gems already installed.

---

### Database Schema

**Subjects Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| name | string | Not null |
| color | string | Nullable |
| description | text | Nullable |
| is_active | boolean | Default: true |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- [teacher_id, name] (unique)

---

**Calendar Events Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| student_id | uuid | Foreign key, nullable |
| subject_id | uuid | Foreign key, nullable |
| title | string | Not null |
| description | text | Nullable |
| start_time | datetime | Not null |
| end_time | datetime | Not null |
| all_day | boolean | Default: false |
| recurrence_rule | string | Nullable |
| location | string | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- student_id
- subject_id
- start_time
- [teacher_id, start_time]

**Foreign Keys:**
- teacher_id references teachers(id)
- student_id references students(id) with ON DELETE SET NULL
- subject_id references subjects(id) with ON DELETE SET NULL

---

### Model Updates

**Teacher:** Add associations to subjects and calendar events (destroyed when teacher deleted)

**Student:** Add association to calendar events (set null when student deleted)

---

### Subject Model

**Behaviors:**
- Belongs to a teacher
- Has many calendar events (set null when subject deleted)
- Destroyed when teacher deleted

**Validations:**

| Field | Rules |
|-------|-------|
| name | presence, max length 100, unique per teacher |
| color | valid hex format (optional) |

**Required Capabilities:**
- Query active subjects only

---

### Calendar Event Model

**Behaviors:**
- Belongs to a teacher (required)
- Belongs to a student (optional)
- Belongs to a subject (optional)

**Validations:**

| Field | Rules |
|-------|-------|
| title | presence, max length 255 |
| start_time | presence |
| end_time | presence, must be after start_time |

**Required Capabilities:**
- Query events within a date range
- Query events for a specific student
- Query events for a specific subject

---

### Routes

Add to api/v1 namespace:
- resources :subjects (full CRUD)
- resources :calendar_events (full CRUD)

---

### Subjects Controller

**Location:** `app/controllers/api/v1/subjects_controller.rb`

Standard CRUD scoped to current teacher's subjects.

---

### Calendar Events Controller

**Location:** `app/controllers/api/v1/calendar_events_controller.rb`

**Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| GET | /calendar_events | List events (requires start_date, end_date params) |
| GET | /calendar_events/:id | Show event details |
| POST | /calendar_events | Create event |
| PATCH | /calendar_events/:id | Update event |
| DELETE | /calendar_events/:id | Delete event |

**Index Filters:**
- start_date, end_date (required)
- student_id (optional)
- subject_id (optional)

**Create/Update Validation:**
- If student_id provided, verify student belongs to current teacher
- If subject_id provided, verify subject belongs to current teacher

---

### Testing Requirements

**Subject Model:**
- Validates presence of name
- Name unique per teacher (can duplicate across teachers)
- Belongs to teacher
- Active scope filters correctly

**Calendar Event Model:**
- Validates presence of title, start_time, end_time
- end_time must be after start_time
- Belongs to teacher
- Optional student and subject associations
- Date range scope filters correctly

**Subjects Controller:**
- CRUD operations work
- Cannot access other teacher's subjects

**Calendar Events Controller:**
- Index requires date range params
- Index filters by student and subject
- Create validates student/subject ownership
- Cannot access other teacher's events

---

### Validation Checklist

- [ ] Subjects migration created and run
- [ ] Calendar events migration created and run
- [ ] Subject model implemented
- [ ] Calendar event model implemented
- [ ] Teacher associations added
- [ ] Student association added
- [ ] Subjects controller implemented
- [ ] Calendar events controller implemented
- [ ] Routes configured
- [ ] All tests pass

---

## Phase 4: Assignments & Tasks

### Goal

Add assignments (homework, projects) and tasks (to-do items). Assignments are tied to students and optionally subjects. Tasks are standalone to-do items for the teacher.

---

### Dependencies

None—all required gems already installed.

---

### Database Schema

**Assignments Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| student_id | uuid | Foreign key, not null |
| subject_id | uuid | Foreign key, nullable |
| title | string | Not null |
| description | text | Nullable |
| due_date | date | Nullable |
| status | string | Default: 'pending' |
| grade | string | Nullable |
| max_grade | string | Nullable |
| graded_at | datetime | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- student_id
- subject_id
- due_date
- status

**Status values:** pending, in_progress, completed, graded

**Foreign Keys:**
- teacher_id references teachers(id)
- student_id references students(id) with ON DELETE CASCADE
- subject_id references subjects(id) with ON DELETE SET NULL

---

**Tasks Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| title | string | Not null |
| description | text | Nullable |
| due_date | date | Nullable |
| priority | string | Default: 'medium' |
| status | string | Default: 'pending' |
| completed_at | datetime | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- due_date
- status
- priority

**Priority values:** low, medium, high
**Status values:** pending, completed

---

### Model Updates

**Teacher:** Add associations to assignments and tasks (destroyed when teacher deleted)

**Student:** Add association to assignments (destroyed when student deleted)

**Subject:** Add association to assignments (set null when subject deleted)

---

### Assignment Model

**Behaviors:**
- Belongs to a teacher (required)
- Belongs to a student (required)
- Belongs to a subject (optional)
- Destroyed when student deleted

**Validations:**

| Field | Rules |
|-------|-------|
| title | presence, max length 255 |
| status | inclusion in valid values |

**Required Capabilities:**
- Query by status
- Query by due date range
- Query by student
- Query overdue assignments (past due and not completed/graded)
- Check if assignment is overdue
- Grade an assignment (set grade, max_grade, graded_at, status)

---

### Task Model

**Behaviors:**
- Belongs to a teacher
- Destroyed when teacher deleted

**Validations:**

| Field | Rules |
|-------|-------|
| title | presence, max length 255 |
| priority | inclusion in valid values |
| status | inclusion in valid values |

**Required Capabilities:**
- Query by status
- Query by priority
- Query by due date range
- Query overdue tasks (past due and pending)
- Check if task is overdue
- Complete a task (set status and timestamp)

---

### Routes

Add to api/v1 namespace:

**Assignments:**
- resources :assignments (full CRUD)
- PATCH /assignments/:id/grade (member route)

**Tasks:**
- resources :tasks (full CRUD)
- PATCH /tasks/:id/complete (member route)

---

### Controllers

**Assignments Controller:** `app/controllers/api/v1/assignments_controller.rb`
- Standard CRUD plus grade action
- Validate student/subject ownership

**Tasks Controller:** `app/controllers/api/v1/tasks_controller.rb`
- Standard CRUD plus complete action

---

### Testing Requirements

**Assignment Model:**
- Validates presence of title
- Validates status values
- Belongs to teacher and student
- Overdue check calculates correctly
- Scopes filter correctly

**Task Model:**
- Validates presence of title
- Validates priority and status values
- Complete action sets status and timestamp
- Scopes filter correctly

**Controllers:**
- CRUD operations work
- Grade action updates assignment
- Complete action updates task
- Validates ownership of related records

---

### Validation Checklist

- [ ] Assignments migration created and run
- [ ] Tasks migration created and run
- [ ] Assignment model implemented
- [ ] Task model implemented
- [ ] Associations added to Teacher, Student, Subject
- [ ] Assignments controller with grade action
- [ ] Tasks controller with complete action
- [ ] Routes configured
- [ ] All tests pass

---

## Phase 5: Report Cards

### Goal

Add report cards with grade entries per subject for each student.

---

### Dependencies

None—all required gems already installed.

---

### Database Schema

**Report Cards Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| student_id | uuid | Foreign key, not null |
| title | string | Not null |
| period_start | date | Not null |
| period_end | date | Not null |
| notes | text | Nullable |
| issued_at | datetime | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- student_id
- [student_id, period_start]

**Foreign Keys:**
- teacher_id references teachers(id)
- student_id references students(id) with ON DELETE CASCADE

---

**Grade Entries Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| report_card_id | uuid | Foreign key, not null |
| subject_id | uuid | Foreign key, not null |
| grade | string | Not null |
| notes | text | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- report_card_id
- subject_id
- [report_card_id, subject_id] (unique)

**Foreign Keys:**
- report_card_id references report_cards(id) with ON DELETE CASCADE
- subject_id references subjects(id)

---

### Model Updates

**Teacher:** Add association to report cards (destroyed when teacher deleted)

**Student:** Add association to report cards (destroyed when student deleted)

---

### Report Card Model

**Behaviors:**
- Belongs to a teacher (required)
- Belongs to a student (required)
- Has many grade entries (destroyed when report card deleted)
- Destroyed when student deleted

**Validations:**

| Field | Rules |
|-------|-------|
| title | presence |
| period_start | presence |
| period_end | presence, must be after period_start |

**Required Capabilities:**
- Issue a report card (set timestamp)
- Check if report card has been issued

---

### Grade Entry Model

**Behaviors:**
- Belongs to a report card (required)
- Belongs to a subject (required)
- Destroyed when report card deleted

**Validations:**

| Field | Rules |
|-------|-------|
| grade | presence |
| subject_id | unique per report card |

---

### Routes

Add to api/v1 namespace:

**Report Cards:**
- resources :report_cards (full CRUD)
- PATCH /report_cards/:id/issue (member route)
- Nested: resources :grade_entries, only: [:create, :update, :destroy]

---

### Testing Requirements

**Report Card Model:**
- Validates required fields
- period_end after period_start
- Issue action sets timestamp

**Grade Entry Model:**
- Validates grade presence
- Subject unique per report card

**Controllers:**
- CRUD for report cards
- Nested grade entries
- Issue action works

---

### Validation Checklist

- [ ] Report cards migration created and run
- [ ] Grade entries migration created and run
- [ ] Report card model implemented
- [ ] Grade entry model implemented
- [ ] Associations added
- [ ] Controllers implemented
- [ ] Routes configured
- [ ] All tests pass

---

## Phase 6: Expense Tracking

### Goal

Add expense categories and expenses for tracking homeschool costs.

---

### Dependencies

None—all required gems already installed.

---

### Database Schema

**Expense Categories Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| name | string | Not null |
| color | string | Nullable |
| budget | decimal(10,2) | Nullable |
| is_active | boolean | Default: true |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- [teacher_id, name] (unique)

---

**Expenses Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| expense_category_id | uuid | Foreign key, nullable |
| student_id | uuid | Foreign key, nullable |
| description | string | Not null |
| amount | decimal(10,2) | Not null |
| date | date | Not null |
| vendor | string | Nullable |
| receipt_url | string | Nullable |
| notes | text | Nullable |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- expense_category_id
- student_id
- date

**Foreign Keys:**
- teacher_id references teachers(id)
- expense_category_id references expense_categories(id) with ON DELETE SET NULL
- student_id references students(id) with ON DELETE SET NULL

---

### Model Updates

**Teacher:** Add associations to expense categories and expenses (destroyed when teacher deleted)

**Student:** Add association to expenses (set null when student deleted)

---

### Expense Category Model

**Behaviors:**
- Belongs to a teacher
- Has many expenses (set null when category deleted)
- Destroyed when teacher deleted

**Validations:**

| Field | Rules |
|-------|-------|
| name | presence, unique per teacher |
| budget | numericality, greater than 0 (if present) |

**Required Capabilities:**
- Query active categories only
- Calculate total spent in a date range
- Calculate remaining budget in a date range

---

### Expense Model

**Behaviors:**
- Belongs to a teacher (required)
- Belongs to an expense category (optional)
- Belongs to a student (optional)

**Validations:**

| Field | Rules |
|-------|-------|
| description | presence |
| amount | presence, numericality greater than 0 |
| date | presence |

**Required Capabilities:**
- Query by date range
- Query by category
- Query by student

---

### Routes

Add to api/v1 namespace:
- resources :expense_categories (full CRUD)
- resources :expenses (full CRUD)

---

### Testing Requirements

**Expense Category Model:**
- Validates name presence and uniqueness per teacher
- Budget calculations correct

**Expense Model:**
- Validates required fields
- Amount must be positive
- Scopes filter correctly

**Controllers:**
- CRUD operations work
- Validates ownership

---

### Validation Checklist

- [ ] Expense categories migration created and run
- [ ] Expenses migration created and run
- [ ] Expense category model implemented
- [ ] Expense model implemented
- [ ] Associations added
- [ ] Controllers implemented
- [ ] Routes configured
- [ ] All tests pass

---

## Phase 7: Lesson Planning

### Goal

Add lesson plans that tie together subjects, students, and calendar events.

---

### Dependencies

None—all required gems already installed.

---

### Database Schema

**Lesson Plans Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| subject_id | uuid | Foreign key, nullable |
| title | string | Not null |
| description | text | Nullable |
| objectives | text | Nullable |
| materials | text | Nullable |
| procedure | text | Nullable |
| assessment | text | Nullable |
| duration_minutes | integer | Nullable |
| is_template | boolean | Default: false |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- teacher_id
- subject_id
- is_template

**Foreign Keys:**
- teacher_id references teachers(id)
- subject_id references subjects(id) with ON DELETE SET NULL

---

**Lesson Plan Students Table (join):**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| lesson_plan_id | uuid | Foreign key, not null |
| student_id | uuid | Foreign key, not null |
| created_at | datetime | Not null |
| updated_at | datetime | Not null |

**Indexes:**
- lesson_plan_id
- student_id
- [lesson_plan_id, student_id] (unique)

**Foreign Keys:**
- lesson_plan_id references lesson_plans(id) with ON DELETE CASCADE
- student_id references students(id) with ON DELETE CASCADE

---

### Model Updates

**Teacher:** Add association to lesson plans (destroyed when teacher deleted)

**Subject:** Add association to lesson plans (set null when subject deleted)

---

### Lesson Plan Model

**Behaviors:**
- Belongs to a teacher (required)
- Belongs to a subject (optional)
- Has many students through join table
- Destroyed when teacher deleted

**Validations:**

| Field | Rules |
|-------|-------|
| title | presence, max length 255 |
| duration_minutes | numericality, greater than 0 (if present) |

**Required Capabilities:**
- Query template lesson plans only
- Query by subject
- Duplicate a lesson plan (create copy for reuse)

---

### Lesson Plan Student Model (Join)

**Behaviors:**
- Belongs to a lesson plan
- Belongs to a student
- Destroyed when either lesson plan or student deleted

**Validations:**
- student_id unique per lesson plan

---

### Routes

Add to api/v1 namespace:
- resources :lesson_plans (full CRUD)
- POST /lesson_plans/:id/duplicate (member route)

---

### Testing Requirements

**Lesson Plan Model:**
- Validates title presence
- Many-to-many with students works
- Duplicate creates copy

**Controllers:**
- CRUD operations work
- Duplicate action works
- Can assign students to lesson plan

---

### Validation Checklist

- [ ] Lesson plans migration created and run
- [ ] Lesson plan students migration created and run
- [ ] Lesson plan model implemented
- [ ] Lesson plan student model implemented
- [ ] Associations added
- [ ] Controller with duplicate action
- [ ] Routes configured
- [ ] All tests pass

---

## Phase 8: Production Preparation

### Goal

Add production dependencies, file uploads, background jobs, and deployment configuration.

---

### Dependencies

| Gem | Purpose | Why This Phase | Why This Gem |
|-----|---------|----------------|--------------|
| sidekiq | Background job processing | Email delivery (verification, password reset) should be async in production—can't block requests waiting for SMTP. | Fast, battle-tested, great dashboard. Good_job is a solid DB-based alternative if you want to avoid Redis. Resque is older and slower. |
| redis | In-memory data store for Sidekiq | Required by Sidekiq for job queue storage. | Required by Sidekiq. No alternative if using Sidekiq. |
| aws-sdk-s3 | S3 file uploads for Active Storage | Profile images and receipt uploads need cloud storage in production—local disk doesn't scale and doesn't persist on Heroku. | Official AWS SDK. Shrine is an alternative to Active Storage entirely, but Active Storage is built-in and simpler to start. |
| image_processing | Resize/transform uploaded images | Profile images need thumbnails—can't serve full-size images to mobile clients. | Required by Active Storage for variants. Uses libvips (fast) or ImageMagick under the hood. |

---

### Environment Variables

| Variable | Description |
|----------|-------------|
| REDIS_URL | Redis connection string |
| AWS_ACCESS_KEY_ID | AWS access key |
| AWS_SECRET_ACCESS_KEY | AWS secret key |
| AWS_REGION | AWS region (e.g., us-east-1) |
| AWS_BUCKET | S3 bucket name |

---

### Active Storage Setup

- Run Active Storage installation
- Configure S3 service in storage.yml
- Set production storage service to S3

---

### Sidekiq Configuration

- Configure server and client Redis connections
- Create initializer

---

### File Uploads

**Add attachments to models:**

| Model | Attachment |
|-------|------------|
| Teacher | profile_image |
| Student | profile_image |
| Expense | receipt |

---

### CORS (Updated)

- Change origins to use ENV for production flexibility

---

### Email Configuration

Create mailers for:
- Email verification
- Password reset

Use async delivery via background jobs.

---

### Testing Requirements

**File Uploads:**
- Profile images attach to Teacher and Student
- Receipts attach to Expense
- S3 configuration valid (use mocks in test)

**Background Jobs:**
- Job processor handles jobs
- Mailers queue correctly

**CORS:**
- Production origins work

---

### Validation Checklist

- [ ] Sidekiq and Redis gems installed
- [ ] AWS S3 gem installed
- [ ] Redis configured and running
- [ ] Sidekiq configured
- [ ] Active Storage configured with S3
- [ ] File attachments added to models
- [ ] Mailers created
- [ ] CORS updated for production
- [ ] Deployment configuration ready
- [ ] All tests pass

---

## Complete Model Association Reference

After all phases, here are the complete associations:

**Teacher:**
- has_many refresh_tokens (Phase 1)
- has_many students (Phase 2)
- has_many subjects (Phase 3)
- has_many calendar_events (Phase 3)
- has_many assignments (Phase 4)
- has_many tasks (Phase 4)
- has_many report_cards (Phase 5)
- has_many expense_categories (Phase 6)
- has_many expenses (Phase 6)
- has_many lesson_plans (Phase 7)

**Student:**
- belongs_to teacher (Phase 2)
- has_many calendar_events (Phase 3)
- has_many assignments (Phase 4)
- has_many report_cards (Phase 5)
- has_many expenses (Phase 6)
- has_many lesson_plans through join (Phase 7)

**Subject:**
- belongs_to teacher (Phase 3)
- has_many calendar_events (Phase 3)
- has_many assignments (Phase 4)
- has_many lesson_plans (Phase 7)

---

## API Endpoints Summary

| Phase | Endpoints |
|-------|-----------|
| 0 | GET /health |
| 1 | POST /api/v1/auth/* (register, login, refresh, logout, password/*, email/*) |
| 2 | GET/POST/PATCH/DELETE /api/v1/students |
| 3 | CRUD /api/v1/subjects, CRUD /api/v1/calendar_events |
| 4 | CRUD /api/v1/assignments (+ grade), CRUD /api/v1/tasks (+ complete) |
| 5 | CRUD /api/v1/report_cards (+ issue), nested grade_entries |
| 6 | CRUD /api/v1/expense_categories, CRUD /api/v1/expenses |
| 7 | CRUD /api/v1/lesson_plans (+ duplicate) |
