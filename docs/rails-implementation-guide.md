# Homeschool Master - Rails API Implementation Guide

Technical implementation specification for the Homeschool Master Rails API backend.

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

### Gemfile

```ruby
source "https://rubygems.org"

ruby "3.2.2"

# Core
gem "rails", "~> 7.1.0"
gem "pg", "~> 1.5"
gem "puma", ">= 5.0"

# API
gem "rack-cors", "~> 2.0"

# Environment Variables
gem "dotenv-rails", groups: [:development, :test]

# Timezone data for Windows
gem "tzinfo-data", platforms: %i[windows jruby]

# Performance
gem "bootsnap", require: false

group :development, :test do
  gem "debug", platforms: %i[mri windows]
  gem "rspec-rails", "~> 6.0"
  gem "factory_bot_rails", "~> 6.2"
  gem "faker", "~> 3.2"
  gem "pry-rails"
end

group :development do
  gem "rubocop-rails", require: false
end

group :test do
  gem "shoulda-matchers", "~> 5.3"
  gem "database_cleaner-active_record", "~> 2.1"
end
```

---

### PostgreSQL Setup

Enable UUID extension via migration:

```ruby
class EnableUuid < ActiveRecord::Migration[7.1]
  def change
    enable_extension 'pgcrypto'
  end
end
```

Configure generators for UUID primary keys in `config/application.rb`:

```ruby
config.generators do |g|
  g.orm :active_record, primary_key_type: :uuid
end
```

---

### CORS Configuration

Create `config/initializers/cors.rb`:

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'localhost:8081', '127.0.0.1:8081'
    
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      expose: ['Authorization']
  end
end
```

---

### Environment Variables

Create `.env`:

```bash
# Database
DATABASE_URL=postgres://localhost/homeschool_api_development

# App
APP_HOST=http://localhost:3000
FRONTEND_URL=http://localhost:8081
```

---

### Base Controller

**Location:** `app/controllers/api/v1/base_controller.rb`

Phase 0 version has response helpers only. Authentication is added in Phase 1.

```ruby
module Api
  module V1
    class BaseController < ApplicationController
      private

      def render_success(data, status: :ok, meta: nil)
        response = { success: true, data: data }
        response[:meta] = meta if meta.present?
        render json: response, status: status
      end

      def render_created(data)
        render_success(data, status: :created)
      end

      def render_no_content
        head :no_content
      end

      def render_error(message, code: 'ERROR', status: :bad_request, details: nil)
        response = {
          success: false,
          error: {
            code: code,
            message: message
          }
        }
        response[:error][:details] = details if details.present?
        render json: response, status: status
      end

      def render_unauthorized(message = 'Unauthorized')
        render_error(message, code: 'UNAUTHORIZED', status: :unauthorized)
      end

      def render_forbidden(message = 'Forbidden')
        render_error(message, code: 'FORBIDDEN', status: :forbidden)
      end

      def render_not_found(resource = 'Resource')
        render_error("#{resource} not found", code: 'NOT_FOUND', status: :not_found)
      end

      def render_validation_errors(model)
        render_error(
          'Validation failed',
          code: 'VALIDATION_ERROR',
          status: :unprocessable_entity,
          details: model.errors.messages
        )
      end
    end
  end
end
```

---

### Routes

**Location:** `config/routes.rb`

Phase 0 only defines the health check:

```ruby
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      # Routes added in subsequent phases
    end
  end

  get '/health', to: proc { [200, {}, ['OK']] }
end
```

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
- `render_success` returns `{ success: true, data: ... }` with 200
- `render_created` returns 201 status
- `render_error` returns `{ success: false, error: { code: ..., message: ... } }`
- `render_unauthorized` returns 401
- `render_forbidden` returns 403
- `render_not_found` returns 404
- `render_validation_errors` returns 422 with error details

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

### New Dependencies

Add to Gemfile:

```ruby
# Authentication
gem "bcrypt", "~> 3.1.7"
gem "jwt", "~> 2.7"
```

Run `bundle install`.

---

### Environment Variables

Add to `.env`:

```bash
# JWT
JWT_SECRET_KEY=your-super-secret-key-change-in-production
JWT_ACCESS_TOKEN_EXPIRY=3600
JWT_REFRESH_TOKEN_EXPIRY=2592000
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
- teacher_id references teachers(id) with ON DELETE CASCADE

---

### JWT Service

**Location:** `app/services/jwt_service.rb`

**Methods:**

`encode(payload, exp: nil)`
- Add expiration to payload (default: ENV['JWT_ACCESS_TOKEN_EXPIRY'] or 3600 seconds)
- Add issued-at timestamp
- Sign with HS256 using secret key
- Return JWT string

`encode_refresh_token(teacher_id)`
- Create payload with teacher_id, type: 'refresh', unique jti (SecureRandom.uuid)
- Set expiration (ENV['JWT_REFRESH_TOKEN_EXPIRY'] or 2592000 seconds)
- Sign with HS256
- Return JWT string

`decode(token)`
- Verify signature and expiration
- Return payload hash with symbolized keys
- Return nil on any JWT error

`decode_refresh_token(token)`
- Decode token
- Verify type is 'refresh'
- Return payload or nil

---

### Teacher Model

**Location:** `app/models/teacher.rb`

**Includes:**
- has_secure_password

**Callbacks:**
- before_save: downcase email
- before_create: generate email verification token

**Validations:**
- first_name: presence, max length 100
- last_name: presence, max length 100
- nickname: max length 100 (optional)
- email: presence, uniqueness (case insensitive), valid format
- password: minimum 8 characters (on create or when present)
- phone: max length 20 (optional)

**Associations:**
- has_many :refresh_tokens, dependent: :destroy

**Scopes:**
- active: `where(is_active: true)`
- verified: `where.not(email_verified_at: nil)`

**Instance Methods:**
- `full_name` - "first_name last_name"
- `display_name` - nickname or full_name
- `email_verified?` - email_verified_at present?
- `verify_email!` - set email_verified_at, clear token
- `generate_password_reset_token!` - generate token, set timestamp
- `password_reset_token_valid?` - token present and less than 2 hours old
- `clear_password_reset_token!` - clear token and timestamp

**Class Methods:**
- `find_by_email(email)` - case-insensitive lookup

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
- active: not revoked and not expired

**Instance Methods:**
- `expired?` - expires_at <= now
- `revoked?` - revoked_at present?
- `valid_token?` - not expired and not revoked
- `revoke!` - set revoked_at to now

**Class Methods:**
- `find_valid_by_token(token)` - find in active scope
- `revoke_all_for_teacher(teacher_id)` - revoke all tokens for teacher

---

### Base Controller (Updated)

Add authentication to `app/controllers/api/v1/base_controller.rb`:

```ruby
module Api
  module V1
    class BaseController < ApplicationController
      before_action :authenticate_request

      attr_reader :current_teacher

      private

      def authenticate_request
        token = extract_token_from_header
        return render_unauthorized('Missing authentication token') if token.nil?

        decoded = JwtService.decode(token)
        return render_unauthorized('Invalid or expired token') if decoded.nil?

        @current_teacher = Teacher.find_by(id: decoded[:teacher_id])
        return render_unauthorized('Teacher not found') if @current_teacher.nil?
        return render_unauthorized('Account is deactivated') unless @current_teacher.is_active?
      end

      def extract_token_from_header
        header = request.headers['Authorization']
        return nil unless header
        header.split(' ').last
      end

      # ... all response helpers from Phase 0 ...
    end
  end
end
```

---

### Routes (Updated)

```ruby
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      namespace :auth do
        post 'register', to: 'authentication#register'
        post 'login', to: 'authentication#login'
        post 'refresh', to: 'authentication#refresh'
        post 'logout', to: 'authentication#logout'
        post 'password/reset-request', to: 'passwords#reset_request'
        post 'password/reset', to: 'passwords#reset'
        post 'password/change', to: 'passwords#change'
        post 'email/verify', to: 'email_verification#verify'
        post 'email/resend-verification', to: 'email_verification#resend'
      end
    end
  end

  get '/health', to: proc { [200, {}, ['OK']] }
end
```

---

### Authentication Controller

**Location:** `app/controllers/api/v1/auth/authentication_controller.rb`

**Skip authentication for:** register, login, refresh, logout

**Endpoints:**

`POST /api/v1/auth/register`
- Accept: first_name, last_name, email, password
- Create teacher
- Return 201 with teacher data (exclude password_digest)
- Return 422 for validation errors

`POST /api/v1/auth/login`
- Accept: email, password
- Verify credentials and account active
- Generate access token and refresh token
- Store refresh token in database
- Return 200 with tokens
- Return 401 for invalid credentials or inactive account

`POST /api/v1/auth/refresh`
- Accept: refresh_token
- Decode and validate refresh token
- Find in database, verify not revoked/expired
- Generate new access token
- Return 200 with new access_token
- Return 401 for invalid token

`POST /api/v1/auth/logout`
- Accept: refresh_token
- Revoke token in database
- Return 204
- Return 401 for invalid token

---

### Passwords Controller

**Location:** `app/controllers/api/v1/auth/passwords_controller.rb`

**Skip authentication for:** reset_request, reset

**Endpoints:**

`POST /api/v1/auth/password/reset-request`
- Accept: email
- Generate reset token if teacher exists
- Always return 200 (security)

`POST /api/v1/auth/password/reset`
- Accept: token, password
- Validate token not expired (2 hour window)
- Update password, clear token
- Return 200 or 400

`POST /api/v1/auth/password/change`
- Requires authentication
- Accept: current_password, new_password
- Verify current password
- Update password
- Return 200 or 401

---

### Email Verification Controller

**Location:** `app/controllers/api/v1/auth/email_verification_controller.rb`

**Skip authentication for:** verify

**Endpoints:**

`POST /api/v1/auth/email/verify`
- Accept: token
- Find teacher, verify email
- Return 200 or 400

`POST /api/v1/auth/email/resend-verification`
- Requires authentication
- Generate new token
- Return 200

---

### Testing Requirements

**JWT Service:**
- Encodes payload into valid JWT
- Decoded token contains original payload with symbolized keys
- Access tokens expire after configured time
- Refresh tokens expire after configured time
- Refresh tokens include unique jti
- Returns nil for expired tokens
- Returns nil for invalid signatures
- Returns nil for malformed tokens

**Teacher Model:**
- Validates presence of first_name, last_name, email, password
- Email must be unique (case insensitive)
- Email must match valid format
- Password minimum 8 characters
- Password is hashed (password_digest differs from input)
- Email downcased before save
- Verification token generated on create
- `find_by_email` is case insensitive
- `full_name` returns combined name
- `display_name` returns nickname if present
- `verify_email!` sets timestamp and clears token
- `generate_password_reset_token!` creates token
- `password_reset_token_valid?` false after 2 hours
- Scopes filter correctly

**Refresh Token Model:**
- Validates presence of token, jti, expires_at
- Token and jti must be unique
- `active` scope excludes revoked and expired
- `revoke!` sets revoked_at
- `revoke_all_for_teacher` revokes all for teacher

**Base Controller Authentication:**
- Returns 401 without token
- Returns 401 with invalid token
- Returns 401 with expired token
- Returns 401 for inactive teacher
- Sets current_teacher with valid token

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
- [ ] JwtService implemented and tested
- [ ] Teacher model with validations, callbacks, methods
- [ ] RefreshToken model with validations, scopes, methods
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

### New Dependencies

Add to Gemfile:

```ruby
# Pagination
gem "kaminari", "~> 1.2"
```

Run `bundle install`.

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

### Teacher Model (Updated)

Add association:

```ruby
has_many :students, dependent: :destroy
```

---

### Student Model

**Location:** `app/models/student.rb`

**Associations:**
- belongs_to :teacher

**Validations:**
- first_name: presence, max length 100
- last_name: presence, max length 100
- nickname: max length 100 (optional)
- grade_level: max length 50 (optional)

**Scopes:**
- active: `where(is_active: true)`

**Instance Methods:**
- `full_name` - "first_name last_name"
- `display_name` - nickname or full_name
- `age` - calculated from date_of_birth

---

### Base Controller (Updated)

Add pagination helper:

```ruby
def paginate(collection)
  paginated = collection.page(params[:page]).per(params[:limit] || 20)
  {
    data: paginated,
    meta: {
      page: paginated.current_page,
      limit: paginated.limit_value,
      total: paginated.total_count,
      total_pages: paginated.total_pages,
      has_next: !paginated.last_page?,
      has_prev: !paginated.first_page?
    }
  }
end
```

---

### Routes (Updated)

Add to api/v1 namespace:

```ruby
resources :students
```

---

### Students Controller

**Location:** `app/controllers/api/v1/students_controller.rb`

**Endpoints:**

`GET /api/v1/students`
- List current teacher's students
- Support pagination (page, limit params)
- Support filtering by is_active
- Return paginated response

`GET /api/v1/students/:id`
- Show student details
- Return 404 if not found or not owned by teacher

`POST /api/v1/students`
- Create student for current teacher
- Return 201 with student data
- Return 422 for validation errors

`PATCH /api/v1/students/:id`
- Update student
- Return 200 with updated data
- Return 404 if not found
- Return 422 for validation errors

`DELETE /api/v1/students/:id`
- Delete student
- Return 204
- Return 404 if not found

---

### Testing Requirements

**Student Model:**
- Validates presence of first_name, last_name
- Belongs to teacher
- `full_name` returns combined name
- `display_name` returns nickname if present
- `age` calculates correctly from date_of_birth
- `active` scope filters correctly

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
- [ ] Student model with validations and methods
- [ ] Teacher has_many :students association added
- [ ] Pagination helper added to base controller
- [ ] Students controller implemented
- [ ] Routes configured
- [ ] All tests pass

---

## Phase 3: Subjects & Calendar

### Goal

Add subjects and calendar events. Teachers can create subjects (like "Math" or "Science") and schedule events on the calendar.

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

---

### Teacher Model (Updated)

Add associations:

```ruby
has_many :subjects, dependent: :destroy
has_many :calendar_events, dependent: :destroy
```

---

### Student Model (Updated)

Add association:

```ruby
has_many :calendar_events, dependent: :nullify
```

---

### Subject Model

**Location:** `app/models/subject.rb`

**Associations:**
- belongs_to :teacher
- has_many :calendar_events, dependent: :nullify

**Validations:**
- name: presence, max length 100, unique per teacher
- color: valid hex format (optional)

**Scopes:**
- active: `where(is_active: true)`

---

### Calendar Event Model

**Location:** `app/models/calendar_event.rb`

**Associations:**
- belongs_to :teacher
- belongs_to :student, optional: true
- belongs_to :subject, optional: true

**Validations:**
- title: presence, max length 255
- start_time: presence
- end_time: presence, must be after start_time

**Scopes:**
- for_date_range(start_date, end_date)
- for_student(student_id)
- for_subject(subject_id)

---

### Routes (Updated)

Add to api/v1 namespace:

```ruby
resources :subjects
resources :calendar_events
```

---

### Subjects Controller

**Location:** `app/controllers/api/v1/subjects_controller.rb`

Standard CRUD for current teacher's subjects.

---

### Calendar Events Controller

**Location:** `app/controllers/api/v1/calendar_events_controller.rb`

**Endpoints:**

`GET /api/v1/calendar_events`
- List events with date range filtering (required: start_date, end_date)
- Optional filters: student_id, subject_id
- Return events in range

`GET /api/v1/calendar_events/:id`
- Show event details

`POST /api/v1/calendar_events`
- Create event
- Validate student and subject belong to teacher

`PATCH /api/v1/calendar_events/:id`
- Update event

`DELETE /api/v1/calendar_events/:id`
- Delete event

---

### Testing Requirements

**Subject Model:**
- Validates presence of name
- Name unique per teacher (can duplicate across teachers)
- Belongs to teacher
- `active` scope filters correctly

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
- [ ] CalendarEvent model implemented
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

**Teacher:**
```ruby
has_many :assignments, dependent: :destroy
has_many :tasks, dependent: :destroy
```

**Student:**
```ruby
has_many :assignments, dependent: :destroy
```

**Subject:**
```ruby
has_many :assignments, dependent: :nullify
```

---

### Assignment Model

**Location:** `app/models/assignment.rb`

**Associations:**
- belongs_to :teacher
- belongs_to :student
- belongs_to :subject, optional: true

**Validations:**
- title: presence, max length 255
- status: inclusion in valid values

**Scopes:**
- by_status(status)
- due_between(start_date, end_date)
- for_student(student_id)
- overdue (due_date < today and not completed/graded)

**Instance Methods:**
- `overdue?`
- `grade!` - set grade and graded_at

---

### Task Model

**Location:** `app/models/task.rb`

**Associations:**
- belongs_to :teacher

**Validations:**
- title: presence, max length 255
- priority: inclusion in valid values
- status: inclusion in valid values

**Scopes:**
- by_status(status)
- by_priority(priority)
- due_between(start_date, end_date)
- overdue

**Instance Methods:**
- `complete!` - set status and completed_at
- `overdue?`

---

### Routes (Updated)

```ruby
resources :assignments do
  member do
    patch :grade
  end
end

resources :tasks do
  member do
    patch :complete
  end
end
```

---

### Controllers

Standard CRUD plus:
- Assignments: `grade` action to set grade
- Tasks: `complete` action to mark done

---

### Testing Requirements

**Assignment Model:**
- Validates presence of title
- Validates status values
- Belongs to teacher and student
- `overdue?` calculates correctly
- Scopes filter correctly

**Task Model:**
- Validates presence of title
- Validates priority and status values
- `complete!` sets status and timestamp
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

---

### Model Updates

**Teacher:**
```ruby
has_many :report_cards, dependent: :destroy
```

**Student:**
```ruby
has_many :report_cards, dependent: :destroy
```

---

### Report Card Model

**Location:** `app/models/report_card.rb`

**Associations:**
- belongs_to :teacher
- belongs_to :student
- has_many :grade_entries, dependent: :destroy

**Validations:**
- title: presence
- period_start: presence
- period_end: presence, after period_start

**Instance Methods:**
- `issue!` - set issued_at
- `issued?`

---

### Grade Entry Model

**Location:** `app/models/grade_entry.rb`

**Associations:**
- belongs_to :report_card
- belongs_to :subject

**Validations:**
- grade: presence
- subject unique per report card

---

### Routes (Updated)

```ruby
resources :report_cards do
  member do
    patch :issue
  end
  resources :grade_entries, only: [:create, :update, :destroy]
end
```

---

### Testing Requirements

**Report Card Model:**
- Validates required fields
- period_end after period_start
- `issue!` sets timestamp

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
- [ ] ReportCard model implemented
- [ ] GradeEntry model implemented
- [ ] Associations added
- [ ] Controllers implemented
- [ ] Routes configured
- [ ] All tests pass

---

## Phase 6: Expense Tracking

### Goal

Add expense categories and expenses for tracking homeschool costs.

---

### Database Schema

**Expense Categories Table:**

| Column | Type | Constraints |
|--------|------|-------------|
| id | uuid | Primary key |
| teacher_id | uuid | Foreign key, not null |
| name | string | Not null |
| color | string | Nullable |
| budget | decimal | Nullable |
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
| amount | decimal | Not null |
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

---

### Model Updates

**Teacher:**
```ruby
has_many :expense_categories, dependent: :destroy
has_many :expenses, dependent: :destroy
```

**Student:**
```ruby
has_many :expenses, dependent: :nullify
```

---

### Expense Category Model

**Location:** `app/models/expense_category.rb`

**Associations:**
- belongs_to :teacher
- has_many :expenses, dependent: :nullify

**Validations:**
- name: presence, unique per teacher
- budget: numericality, greater than 0 (if present)

**Scopes:**
- active

**Instance Methods:**
- `total_spent(start_date, end_date)` - sum expenses in range
- `budget_remaining(start_date, end_date)`

---

### Expense Model

**Location:** `app/models/expense.rb`

**Associations:**
- belongs_to :teacher
- belongs_to :expense_category, optional: true
- belongs_to :student, optional: true

**Validations:**
- description: presence
- amount: presence, numericality greater than 0
- date: presence

**Scopes:**
- for_date_range(start_date, end_date)
- for_category(category_id)
- for_student(student_id)

---

### Routes (Updated)

```ruby
resources :expense_categories
resources :expenses
```

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
- [ ] ExpenseCategory model implemented
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
- [lesson_plan_id, student_id] (unique)

---

### Model Updates

**Teacher:**
```ruby
has_many :lesson_plans, dependent: :destroy
```

**Subject:**
```ruby
has_many :lesson_plans, dependent: :nullify
```

---

### Lesson Plan Model

**Location:** `app/models/lesson_plan.rb`

**Associations:**
- belongs_to :teacher
- belongs_to :subject, optional: true
- has_many :lesson_plan_students, dependent: :destroy
- has_many :students, through: :lesson_plan_students

**Validations:**
- title: presence, max length 255
- duration_minutes: numericality, greater than 0 (if present)

**Scopes:**
- templates: `where(is_template: true)`
- for_subject(subject_id)

**Instance Methods:**
- `duplicate` - create copy (for templates)

---

### Lesson Plan Student Model

**Location:** `app/models/lesson_plan_student.rb`

**Associations:**
- belongs_to :lesson_plan
- belongs_to :student

**Validations:**
- student unique per lesson plan

---

### Routes (Updated)

```ruby
resources :lesson_plans do
  member do
    post :duplicate
  end
end
```

---

### Testing Requirements

**Lesson Plan Model:**
- Validates title presence
- Many-to-many with students works
- `duplicate` creates copy

**Controllers:**
- CRUD operations work
- Duplicate action works
- Can assign students to lesson plan

---

### Validation Checklist

- [ ] Lesson plans migration created and run
- [ ] Lesson plan students migration created and run
- [ ] LessonPlan model implemented
- [ ] LessonPlanStudent model implemented
- [ ] Associations added
- [ ] Controller with duplicate action
- [ ] Routes configured
- [ ] All tests pass

---

## Phase 8: Production Preparation

### Goal

Add production dependencies, file uploads, background jobs, and deployment configuration.

---

### New Dependencies

Add to Gemfile:

```ruby
# Background Jobs
gem "sidekiq", "~> 7.1"
gem "redis", ">= 4.0.1"

# File Uploads
gem "aws-sdk-s3", "~> 1.136"
gem "image_processing", "~> 1.12"
```

Run `bundle install`.

---

### Environment Variables (Updated)

Add to `.env`:

```bash
# Redis
REDIS_URL=redis://localhost:6379/0

# AWS S3
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_REGION=us-east-1
AWS_BUCKET=homeschool-app-uploads
```

---

### Active Storage Setup

```bash
rails active_storage:install
rails db:migrate
```

Configure S3 in `config/storage.yml`:

```yaml
amazon:
  service: S3
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: <%= ENV['AWS_REGION'] %>
  bucket: <%= ENV['AWS_BUCKET'] %>
```

Set production storage in `config/environments/production.rb`:

```ruby
config.active_storage.service = :amazon
```

---

### Sidekiq Configuration

Create `config/initializers/sidekiq.rb`:

```ruby
Sidekiq.configure_server do |config|
  config.redis = { url: ENV['REDIS_URL'] }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV['REDIS_URL'] }
end
```

---

### File Uploads

Add to Teacher model:

```ruby
has_one_attached :profile_image
```

Add to Student model:

```ruby
has_one_attached :profile_image
```

Add to Expense model:

```ruby
has_one_attached :receipt
```

---

### CORS (Updated for Production)

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV.fetch('FRONTEND_URL', 'http://localhost:8081')
    
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      expose: ['Authorization']
  end
end
```

---

### Email Jobs

Create mailers for:
- Email verification
- Password reset
- (Future: assignment reminders, etc.)

Use Sidekiq for async delivery:

```ruby
UserMailer.verification_email(teacher).deliver_later
```

---

### Testing Requirements

**File Uploads:**
- Profile images attach to Teacher and Student
- Receipts attach to Expense
- S3 configuration valid (use mocks in test)

**Background Jobs:**
- Sidekiq processes jobs
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
```ruby
has_many :refresh_tokens, dependent: :destroy      # Phase 1
has_many :students, dependent: :destroy            # Phase 2
has_many :subjects, dependent: :destroy            # Phase 3
has_many :calendar_events, dependent: :destroy     # Phase 3
has_many :assignments, dependent: :destroy         # Phase 4
has_many :tasks, dependent: :destroy               # Phase 4
has_many :report_cards, dependent: :destroy        # Phase 5
has_many :expense_categories, dependent: :destroy  # Phase 6
has_many :expenses, dependent: :destroy            # Phase 6
has_many :lesson_plans, dependent: :destroy        # Phase 7
```

**Student:**
```ruby
belongs_to :teacher                                # Phase 2
has_many :calendar_events, dependent: :nullify     # Phase 3
has_many :assignments, dependent: :destroy         # Phase 4
has_many :report_cards, dependent: :destroy        # Phase 5
has_many :expenses, dependent: :nullify            # Phase 6
has_many :lesson_plan_students, dependent: :destroy # Phase 7
has_many :lesson_plans, through: :lesson_plan_students # Phase 7
```

**Subject:**
```ruby
belongs_to :teacher                                # Phase 3
has_many :calendar_events, dependent: :nullify     # Phase 3
has_many :assignments, dependent: :nullify         # Phase 4
has_many :lesson_plans, dependent: :nullify        # Phase 7
```

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