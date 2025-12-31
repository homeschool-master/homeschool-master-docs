# Homeschool Master - Rails API Implementation Guide (Learning Edition)

This guide helps you build the Rails API from scratch by understanding concepts, not copying code. You'll research solutions, write your own implementations, and validate your work through testing.

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

### Concepts to Understand

**What is an API-only Rails application?**
- Doesn't serve HTML pages
- Returns JSON data
- No views, sessions, or cookies by default
- Lightweight middleware stack

**What are UUIDs and why use them?**
- Universally unique identifiers instead of incrementing integers
- Better security (can't guess IDs)
- Easier database merging
- Distributed system friendly

**What is CORS and why does it matter?**
- Browser security feature that blocks cross-origin requests
- Your mobile app and API are different origins
- Must explicitly allow your frontend to access your API

<<<<<<< HEAD
Replace the contents of `Gemfile` with:

```ruby
source "https://rubygems.org"

ruby "3.2.2"

# Core
gem "rails", "~> 7.1.0"
gem "pg", "~> 1.5"
gem "puma", ">= 5.0"

# Authentication
gem "bcrypt", "~> 3.1.7"
gem "jwt", "~> 2.7"

# API & Serialization
gem "jbuilder", "~> 2.11"
gem "rack-cors", "~> 2.0"

# Background Jobs (for emails, etc.)
gem "sidekiq", "~> 7.1"
gem "redis", ">= 4.0.1"

# File Uploads
gem "aws-sdk-s3", "~> 1.136"
gem "image_processing", "~> 1.12"

# Pagination
gem "kaminari", "~> 1.2"

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
  gem "annotate", "~> 3.2"
  gem "rubocop-rails", require: false
end

group :test do
  gem "shoulda-matchers", "~> 5.3"
  gem "database_cleaner-active_record", "~> 2.1"
end
```

### 0.3 Install Dependencies

```bash
bundle install
```

### 0.4 Setup Database

```bash
# Create databases
rails db:create
```

### 0.5 Configure CORS

Create/update `config/initializers/cors.rb`:

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'  # In production, replace with actual domains
    
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      expose: ['Authorization', 'X-RateLimit-Limit', 'X-RateLimit-Remaining']
  end
end
```

### 0.6 Create Environment Variables

Create `.env` file in project root:

```bash
# Database
DATABASE_URL=postgres://localhost/homeschool_api_development

# JWT
JWT_SECRET_KEY=super-secret-key-change-in-production
JWT_ACCESS_TOKEN_EXPIRY=3600
JWT_REFRESH_TOKEN_EXPIRY=2592000

# Redis
REDIS_URL=redis://localhost:6379/0

# AWS S3 (for file uploads)
AWS_ACCESS_KEY_ID=access-key
AWS_SECRET_ACCESS_KEY=secret-key
AWS_REGION=us-east-1
AWS_BUCKET=homeschool-app-uploads

# App
APP_HOST=http://localhost:3000
FRONTEND_URL=http://localhost:8081
```

### 0.7 Setup RSpec

```bash
rails generate rspec:install
```

### 0.8 Create Base API Structure

Create the following directory structure:

```bash
mkdir -p app/controllers/api/v1
mkdir -p app/services
mkdir -p app/serializers
```

### 0.9 Create Base API Controller

Create `app/controllers/api/v1/base_controller.rb`:

```ruby
module Api
  module V1
    class BaseController < ApplicationController
      include ActionController::HttpAuthentication::Token::ControllerMethods
      
      before_action :authenticate_request
      
      attr_reader :current_teacher
      
      private
      
      def authenticate_request
        token = extract_token_from_header
        
        if token.nil?
          render_unauthorized("Missing authentication token")
          return
        end
        
        decoded = JwtService.decode(token)
        
        if decoded.nil?
          render_unauthorized("Invalid or expired token")
          return
        end
        
        @current_teacher = Teacher.find_by(id: decoded[:teacher_id])
        
        if @current_teacher.nil?
          render_unauthorized("Teacher not found")
          return
        end
        
        unless @current_teacher.is_active?
          render_unauthorized("Account is deactivated")
        end
      end
      
      def extract_token_from_header
        header = request.headers['Authorization']
        return nil unless header
        
        header.split(' ').last
      end
      
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
      
      def render_error(message, code: "ERROR", status: :bad_request, details: nil)
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
      
      def render_unauthorized(message = "Unauthorized")
        render_error(message, code: "UNAUTHORIZED", status: :unauthorized)
      end
      
      def render_forbidden(message = "Forbidden")
        render_error(message, code: "FORBIDDEN", status: :forbidden)
      end
      
      def render_not_found(resource = "Resource")
        render_error("#{resource} not found", code: "NOT_FOUND", status: :not_found)
      end
      
      def render_validation_errors(model)
        render_error(
          "Validation failed",
          code: "VALIDATION_ERROR",
          status: :unprocessable_entity,
          details: model.errors.messages
        )
      end
      
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
    end
  end
end
```

### 0.10 Configure Routes Base

Update `config/routes.rb`:

```ruby
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      # Authentication (no auth required)
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
      
      # Protected routes will be added here
    end
  end
  
  # Health check
  get '/health', to: proc { [200, {}, ['OK']] }
end
```

### ✅ Phase 0 Checklist

- [ ] Created Rails API project
- [ ] Updated Gemfile and installed gems
- [ ] Created databases
- [ ] Configured CORS
- [ ] Created .env file
- [ ] Setup RSpec
- [ ] Created directory structure
- [ ] Created BaseController
- [ ] Configured routes base
=======
**Environment variables - why not hardcode secrets?**
- Different values for development/production
- Keep secrets out of version control
- Easy to change without code changes
>>>>>>> 0b783d9 (update implementation plan.)

---

### Architecture Overview

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

### What to Build

**1. Initialize Rails Project**

Research how to create a Rails project that:
- Only serves API requests (no views)
- Uses PostgreSQL as the database
- Skips the default test framework (you'll use RSpec)

**2. Configure Dependencies**

Your application needs gems for:
- Password encryption (research bcrypt)
- JSON Web Token handling (research JWT libraries)
- Cross-origin resource sharing
- Background job processing
- Caching layer
- Cloud file storage
- Pagination
- Testing framework and factories
- Code quality checking

Don't just install these - understand what each does and why you need it.

**3. Enable UUID Support**

PostgreSQL needs an extension to generate UUIDs. Research:
- What extension enables UUID generation in PostgreSQL
- How to enable extensions in Rails migrations
- Why this must happen before creating any tables

**4. Configure CORS**

Your API must allow requests from:
- Local development server (where your React Native app runs)
- Production mobile app domain
- Future web application

Research:
- How to configure CORS in Rails
- What headers need to be allowed
- What HTTP methods should be permitted
- Which headers should be exposed to the frontend

**5. Set Up Environment Variables**

Create a template for environment variables including:
- Database connection details
- Secret keys for token generation
- Token expiration times
- Redis connection (for caching/jobs)
- Cloud storage credentials
- Application URLs

Research best practices for managing environment variables in Rails.

**6. Create Base API Structure**

Design a base controller that all API endpoints will inherit from. This controller should handle:
- Authentication checks before any action runs
- Making the authenticated user available to child controllers
- Standardized JSON response formatting for success and errors
- Consistent error handling
- Pagination helpers

Think about:
- Where should this controller live in your file structure?
- What should it inherit from?
- How can you skip authentication for certain endpoints (like login)?

---

### What to Test

**Project Setup Tests:**
- Can the Rails server start without errors?
- Can you connect to PostgreSQL?
- Are all required gems installed?
- Does the database support UUID generation?

**CORS Tests:**
- Can you make a cross-origin request to the API?
- Are the correct headers returned?
- Are credentials properly handled?

**Base Controller Tests:**
- Does it reject requests without authentication tokens?
- Does it properly format success responses?
- Does it properly format error responses?
- Does it handle different HTTP status codes correctly?
- Can specific endpoints skip authentication?

---

### Validation Checklist

- [ ] Rails server starts on port 3000
- [ ] PostgreSQL database exists and is accessible
- [ ] UUID extension is enabled in PostgreSQL
- [ ] CORS responds with proper headers
- [ ] Environment variables load from .env file
- [ ] Base controller structure exists
- [ ] Can make a test API request and get JSON response
- [ ] Code passes linter checks

---

### Research Resources

**Topics to explore:**
- Rails API-only applications
- PostgreSQL extensions and UUIDs
- CORS and preflight requests
- Rails before_action callbacks
- Rails service objects pattern
- Environment variable management
- Rails concerns and modules

**Where to look:**
- Rails API documentation
- PostgreSQL documentation
- RubyGems.org for available libraries
- Rails guides on API-only applications

---

## Phase 1: Authentication System

### Concepts to Understand

**What is JWT and how does it work?**
- JSON Web Token - a way to transmit information securely
- Three parts: header, payload, signature
- Stateless - server doesn't store tokens
- Self-contained - all info is in the token

**Why two types of tokens?**
- Access tokens: short-lived (1 hour), used for API requests
- Refresh tokens: long-lived (30 days), used to get new access tokens
- If access token is stolen, it expires quickly
- If refresh token is stolen and user logs out, it becomes invalid

**How does password hashing work?**
- Never store plain-text passwords
- One-way function - can't reverse the hash
- Same password + different salt = different hash
- Slow algorithm to prevent brute force attacks

**What is a JWT ID (jti)?**
- Unique identifier for each token
- Allows tracking individual tokens in database
- Enables token revocation (logout)
- Critical for refresh tokens

---

### Architecture: Authentication Flow

**Simple Flow:**
```
User → Login → Server checks credentials → Generate tokens → Return tokens
User → API request with access token → Server verifies → Return data
User → Refresh request with refresh token → Server verifies → Return new access token
User → Logout → Server revokes refresh token → Success
```

**Detailed Login Sequence:**
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

**Detailed Authenticated Request Sequence:**
```
┌──────┐                    ┌──────┐                   ┌──────────┐
│Client│                    │ API  │                   │ Database │
└──┬───┘                    └──┬───┘                   └────┬─────┘
   │                           │                            │
   │ GET /students             │                            │
   │ Header: Authorization:    │                            │
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

### What to Build

**1. Token Service**

Create a service that handles all token operations:

**Token encoding:**
- Take a data payload (like teacher ID)
- Add an expiration timestamp
- Add an issued-at timestamp
- Sign the payload with a secret key
- Return the encoded token string

**Refresh token encoding:**
- Include teacher ID
- Mark token type as "refresh"
- Add expiration (much longer than access token)
- Add issued-at timestamp
- Generate and include a unique identifier
- Sign and return

**Token decoding:**
- Take an encoded token string
- Verify the signature matches
- Check if token has expired
- Extract and return the payload
- Handle errors gracefully (expired, invalid, malformed)

**Refresh token decoding:**
- Decode the token
- Verify it's marked as a refresh token type
- Return the payload or nothing if invalid

Research:
- What library can handle JWT encoding/decoding in Ruby?
- How to handle cryptographic signing?
- What algorithm is secure for JWT signing?
- How to structure error handling for token operations?

---

**2. Teacher Model**

Create a model representing teachers with:

**Authentication:**
- Secure password storage (never plain text)
- Password validation (minimum length, presence)
- Email uniqueness enforcement
- Case-insensitive email lookup

**Email Verification:**
- Token generation for email verification
- Timestamp tracking when email was verified
- Method to verify email and clear token
- Method to check if email is verified

**Password Reset:**
- Token generation for password reset
- Timestamp tracking when reset was requested
- Method to check if reset token is still valid (time limit)
- Method to clear reset token after use

**Utility Methods:**
- Combine first and last name
- Display name (nickname or full name)
- Find teacher by email (case insensitive)

**Associations:**
Will relate to students, calendar events, assignments, etc. (built in later phases)

Research:
- How does Rails handle password hashing?
- What validations does Rails provide?
- How to generate secure random tokens?
- How to implement email format validation?

---

**3. Refresh Token Model**

Create a model to track long-lived refresh tokens:

**Storage:**
- Link to teacher who owns the token
- Store the token string
- Store the unique identifier
- Store expiration timestamp
- Store revocation timestamp (if revoked)

**Validation:**
- Token must exist and be unique
- Unique identifier must exist and be unique
- Expiration must be set

**Queries:**
- Find active tokens (not revoked, not expired)
- Find valid token by token string

**Actions:**
- Mark token as revoked
- Revoke all tokens for a specific teacher

Research:
- How to set up model associations in Rails?
- How to write database scopes for filtering?
- How to validate uniqueness?

---

**4. Database Migrations**

Create migrations for:

**Teachers table:**
- UUID primary key
- Name fields (first, last, nickname)
- Email (unique, indexed)
- Phone (optional)
- Password hash
- Newsletter subscription flag
- Profile image URL
- Email verification token and timestamp
- Password reset token and timestamp
- Active/inactive flag
- Automatic timestamps

**Refresh tokens table:**
- UUID primary key
- Reference to teacher (foreign key)
- Token string (unique, indexed)
- Unique identifier (unique, indexed)
- Expiration timestamp
- Revocation timestamp
- Automatic timestamps

Research:
- How to enable UUID primary keys in Rails migrations?
- How to add indexes for better query performance?
- How to add foreign key constraints?
- Why index certain columns?

---

**5. Authentication Controller**

Create controller endpoints for:

**Registration:**
- Accept name, email, password
- Validate input
- Check email isn't already registered
- Create teacher account
- Generate email verification token
- Return success with teacher info
- Handle validation errors

**Login:**
- Accept email and password
- Find teacher by email
- Verify password is correct
- Check account is active
- Generate access token
- Generate refresh token
- Store refresh token in database
- Return both tokens to client
- Handle invalid credentials

**Token Refresh:**
- Accept refresh token
- Verify token is valid
- Check token hasn't been revoked
- Check token hasn't expired
- Generate new access token
- Return new access token
- Handle invalid/expired tokens

**Logout:**
- Accept refresh token
- Find token in database
- Mark as revoked
- Return success
- Handle missing/invalid tokens

**Email Verification:**
- Accept verification token from email link
- Find teacher with that token
- Mark email as verified
- Clear verification token
- Return success
- Handle invalid/expired tokens

**Password Reset Request:**
- Accept email address
- Find teacher by email
- Generate reset token
- Store token with timestamp
- Send reset email (simulate for now)
- Return success
- Handle teacher not found

**Password Reset:**
- Accept reset token and new password
- Find teacher with that token
- Verify token is still valid (time limit)
- Update password
- Clear reset token
- Return success
- Handle invalid/expired tokens

**Password Change:**
- Accept current password and new password
- Verify current password is correct
- Update to new password
- Return success
- Handle incorrect current password

Research:
- How to organize controller actions?
- How to handle strong parameters in Rails?
- How to skip authentication for public endpoints?
- How to return consistent JSON responses?
- Proper HTTP status codes for different scenarios

---

### What to Test

**Token Service Tests:**
- Can encode a payload into a valid token
- Encoded token contains the correct payload when decoded
- Access tokens expire after the configured time
- Refresh tokens expire after the configured time
- Decoding returns nothing for expired tokens
- Decoding returns nothing for tokens with invalid signatures
- Decoding returns nothing for malformed tokens
- Refresh tokens include a unique identifier

**Teacher Model Tests:**
- Cannot create teacher without required fields
- Email must be unique (case insensitive)
- Email must be valid format
- Password must meet minimum length
- Password is hashed, not stored in plain text
- Can find teacher by email (case insensitive)
- Email verification token is generated on creation
- Can verify email and clear token
- Can generate password reset token
- Reset token expires after time limit
- Can check if email is verified
- Full name combines first and last name
- Display name uses nickname if present, otherwise full name

**Refresh Token Model Tests:**
- Cannot create without required fields
- Token string must be unique
- Unique identifier must be unique
- Can find active tokens (not revoked, not expired)
- Revoked tokens are not included in active scope
- Expired tokens are not included in active scope
- Can revoke a single token
- Can revoke all tokens for a teacher

**Authentication Controller Tests:**

*Registration:*
- Can register with valid credentials
- Returns error for duplicate email
- Returns error for invalid email format
- Returns error for weak password
- Returns error for missing required fields
- Creates teacher in database
- Generates email verification token

*Login:*
- Can login with correct credentials
- Returns access and refresh tokens
- Returns error for wrong password
- Returns error for non-existent email
- Returns error for inactive account
- Stores refresh token in database

*Token Refresh:*
- Can get new access token with valid refresh token
- Returns error for invalid refresh token
- Returns error for expired refresh token
- Returns error for revoked refresh token

*Logout:*
- Can logout with valid refresh token
- Marks refresh token as revoked
- Returns error for invalid token
- Multiple logouts don't cause errors

*Email Verification:*
- Can verify email with valid token
- Marks email as verified
- Clears verification token
- Returns error for invalid token
- Returns error for already verified email

*Password Reset Request:*
- Can request reset with valid email
- Generates reset token
- Returns success for non-existent email (security)
- Token has expiration

*Password Reset:*
- Can reset password with valid token
- Clears reset token after use
- Returns error for expired token
- Returns error for invalid token

*Password Change:*
- Can change password with correct current password
- Returns error for incorrect current password
- Requires authentication
- New password must meet requirements

---

### Validation Checklist

- [ ] Token service can encode and decode tokens
- [ ] Access tokens expire after 1 hour
- [ ] Refresh tokens expire after 30 days
- [ ] Teacher model encrypts passwords
- [ ] Email validation works correctly
- [ ] Email verification flow works
- [ ] Password reset flow works
- [ ] Can register a new teacher account
- [ ] Can login with correct credentials
- [ ] Login returns both token types
- [ ] Can use access token to authenticate requests
- [ ] Can refresh access token with refresh token
- [ ] Can logout and revoke refresh token
- [ ] Cannot use revoked refresh token
- [ ] All authentication tests pass

---

### Research Resources

**Topics to explore:**
- JWT structure and security
- bcrypt password hashing
- Token-based authentication vs sessions
- Why refresh tokens need database storage
- Secure random token generation
- Time-based token expiration
- Rails has_secure_password
- Rails validations
- Rails callbacks
- ActiveRecord associations

---

## Phase 2: Student Management

### Concepts to Understand

**What is a one-to-many relationship?**
- One teacher has many students
- Each student belongs to one teacher
- Database enforces referential integrity with foreign keys
- Deleting parent can cascade to children

**What is soft deletion?**
- Don't actually delete records from database
- Mark them as archived/deleted with a timestamp
- Preserve data for history and reporting
- Can be "undeleted" if needed

**Why validate at both model and database level?**
- Model validations give friendly error messages
- Database constraints prevent bad data even if validations are bypassed
- Belt and suspenders approach for data integrity

---

### Architecture: Student Data Model

```
┌─────────────────────────────────────┐
│           teachers                   │
│─────────────────────────────────────│
│ id (uuid)                            │
│ first_name                           │
│ last_name                            │
│ email                                │
│ ...                                  │
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

### What to Build

**1. Student Model**

Create a model for students with:

**Data Fields:**
- Names (first, last, optional nickname)
- Date of birth
- Current grade level
- Free-form notes field
- Soft deletion timestamp

**Associations:**
- Belongs to a teacher
- Cannot exist without a teacher

**Validations:**
- Required fields must be present
- Names have maximum lengths
- Date of birth must be in the past
- Grade level must be valid (research typical grade ranges)
- Teacher must exist

**Scopes:**
- Active students (not archived)
- Archived students
- Students in a specific grade
- Students ordered by name

**Methods:**
- Full name combination
- Display name (nickname or full name)
- Calculate age from date of birth
- Archive student (soft delete)
- Restore archived student
- Check if archived

Research:
- How to implement soft deletion in Rails?
- How to write custom validations?
- How to validate foreign key existence?
- How to calculate age from date of birth in Ruby?

---

**2. Database Migration**

Create migration for students table:

**Columns:**
- UUID primary key
- Teacher foreign key (UUID)
- Name fields
- Date of birth
- Grade level
- Notes (text field)
- Archived timestamp
- Automatic timestamps

**Indexes:**
- Teacher ID (for fast teacher lookup)
- Archived timestamp (for soft delete queries)
- Composite index on teacher + archived (for filtered queries)

**Constraints:**
- Foreign key to teachers
- Not null on required fields
- Check constraint on date of birth (must be past)

Research:
- How to add foreign key constraints?
- What is a composite index and when to use it?
- How to add check constraints in PostgreSQL?

---

**3. Students Controller**

Create CRUD endpoints:

**List Students:**
- Show all active students for authenticated teacher
- Support pagination
- Support filtering by grade
- Support searching by name
- Return count and pagination metadata

**Get Single Student:**
- Find student by ID
- Verify student belongs to authenticated teacher
- Return student details
- Handle not found

**Create Student:**
- Accept student details
- Link to authenticated teacher
- Validate all fields
- Return created student
- Handle validation errors

**Update Student:**
- Find student by ID
- Verify ownership
- Update allowed fields
- Validate changes
- Return updated student
- Handle validation errors

**Archive Student:**
- Find student by ID
- Verify ownership
- Mark as archived with timestamp
- Return success
- Handle already archived

**Restore Student:**
- Find archived student by ID
- Verify ownership
- Clear archived timestamp
- Return success
- Handle not archived

Research:
- How to implement authorization (checking ownership)?
- How to structure controller actions?
- How to handle pagination in Rails?
- How to implement search functionality?
- Difference between update and patch?

---

### What to Test

**Student Model Tests:**
- Cannot create student without teacher
- Cannot create student without required fields
- Names have appropriate length limits
- Date of birth must be in the past
- Can calculate age correctly
- Full name combines correctly
- Display name uses nickname if available
- Can archive student
- Archived students not in active scope
- Can restore archived student
- Scope filters by grade level correctly

**Students Controller Tests:**

*List:*
- Returns only authenticated teacher's students
- Returns only active students by default
- Pagination works correctly
- Returns metadata (page, total, etc.)
- Can filter by grade level
- Can search by name
- Requires authentication

*Show:*
- Returns student details
- Returns error if student doesn't belong to teacher
- Returns error for non-existent student
- Requires authentication

*Create:*
- Can create student with valid data
- Student is linked to authenticated teacher
- Returns error for missing required fields
- Returns error for invalid date of birth
- Returns created student data
- Requires authentication

*Update:*
- Can update student fields
- Returns error if student doesn't belong to teacher
- Returns error for invalid data
- Cannot change teacher ID
- Requires authentication

*Archive:*
- Can archive student
- Student no longer appears in active list
- Can still be retrieved by ID
- Returns error if already archived
- Requires authentication

*Restore:*
- Can restore archived student
- Student appears in active list again
- Returns error if not archived
- Requires authentication

---

### Validation Checklist

- [ ] Student model exists with all fields
- [ ] Student belongs to teacher association works
- [ ] Validations prevent invalid data
- [ ] Soft deletion works correctly
- [ ] Can create students via API
- [ ] Can list students with pagination
- [ ] Can update student details
- [ ] Can archive and restore students
- [ ] Students are filtered by teacher
- [ ] Search functionality works
- [ ] All student tests pass

---

## Phase 3: Calendar & Events

### Concepts to Understand

**What makes a good calendar data model?**
- Events have start and end times
- Events can be all-day or specific times
- Events can recur (daily, weekly, monthly)
- Need to query events by date range efficiently

**What is polymorphic association?**
- An event can belong to different types of records
- Could be linked to a student, assignment, or just general
- One association that works for multiple model types
- Requires type and ID columns

**How do timezones work in databases?**
- Always store in UTC in database
- Convert to user's timezone for display
- User's timezone should be configurable
- Rails handles conversion automatically if set up correctly

---

### Architecture: Calendar System

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
         ├──► Student (field trip for one student)
         ├──► Assignment (assignment due date)
         └──► Lesson Plan (scheduled lesson)
```

---

### What to Build

**1. Calendar Event Model**

Create a model for calendar events with:

**Data Fields:**
- Title and description
- Start and end timestamps
- All-day flag
- Event type (lesson, field trip, appointment, holiday, etc.)
- Recurrence rule (for repeating events)
- Link to teacher
- Optional link to related record (polymorphic)

**Associations:**
- Belongs to teacher
- Optionally belongs to student, assignment, or lesson plan

**Validations:**
- Title is required
- Start time is required
- End time must be after start time
- Event type must be valid
- Teacher must exist

**Scopes:**
- Events in a date range
- Events of a specific type
- All-day events
- Recurring events
- Events for a specific student

**Methods:**
- Check if event is currently happening
- Check if event is all day
- Calculate duration
- Check if event recurs
- Generate occurrences for recurring events

Research:
- How to implement polymorphic associations in Rails?
- How to store and query date ranges?
- How to implement enum fields in Rails?
- How to handle recurring events (research iCal RRULE)?
- How to work with time zones in Rails?

---

**2. Database Migration**

Create migration for calendar events table:

**Columns:**
- UUID primary key
- Teacher foreign key
- Title and description
- Start and end timestamps (with timezone)
- All-day boolean
- Event type string
- Recurrence rule text (JSON or string)
- Polymorphic type and ID
- Automatic timestamps

**Indexes:**
- Teacher ID
- Start time (for date range queries)
- Composite index on teacher + start time
- Polymorphic type and ID

Research:
- How to index timestamp columns for range queries?
- How to store JSON data in PostgreSQL?
- How to add polymorphic foreign keys?

---

**3. Calendar Events Controller**

Create endpoints:

**List Events:**
- Filter by date range (required)
- Filter by event type
- Filter by student (if linked)
- Support pagination
- Return with related records included
- Expand recurring events into instances

**Get Single Event:**
- Find event by ID
- Verify ownership
- Include related records
- Handle not found

**Create Event:**
- Accept event details
- Link to authenticated teacher
- Optionally link to student/assignment
- Validate times
- Return created event
- Handle validation errors

**Update Event:**
- Find event by ID
- Verify ownership
- Update allowed fields
- Validate changes
- Return updated event
- Handle recurring event updates (this instance vs all instances)

**Delete Event:**
- Find event by ID
- Verify ownership
- Delete from database
- Handle recurring event deletion (this instance vs all instances)

Research:
- How to efficiently query date ranges?
- How to expand recurring events?
- How to handle "update all" vs "update one" for series?
- How to include associated records in JSON?

---

### What to Test

**Calendar Event Model Tests:**
- Cannot create without teacher
- Cannot create without title
- Cannot create without start time
- End time must be after start time
- All-day events don't require specific times
- Event type validation works
- Can link to student (polymorphic)
- Can filter events by date range
- Recurring events are identified correctly
- Duration calculation works

**Calendar Events Controller Tests:**

*List:*
- Returns events in date range
- Returns only authenticated teacher's events
- Can filter by event type
- Can filter by student
- Pagination works
- Recurring events expand correctly
- Includes related records
- Requires authentication

*Show:*
- Returns event details
- Includes linked records
- Returns error for other teacher's event
- Requires authentication

*Create:*
- Can create event with valid data
- Can create all-day event
- Can link to student
- Returns error for end before start
- Returns error for missing required fields
- Requires authentication

*Update:*
- Can update event fields
- Cannot change teacher
- End time validation still applies
- Can update recurring event
- Requires authentication

*Delete:*
- Can delete event
- Event no longer appears in list
- Returns error for non-existent event
- Requires authentication

---

### Validation Checklist

- [ ] Calendar event model exists
- [ ] Polymorphic associations work
- [ ] Date range queries are efficient
- [ ] Can create events via API
- [ ] Can list events for date range
- [ ] Can filter events by type
- [ ] All-day events work correctly
- [ ] Can link events to students
- [ ] Event validation prevents invalid data
- [ ] All calendar tests pass

---

## Phase 4: Assignments & Tasks

### Concepts to Understand

**Difference between assignments and tasks?**
- Assignments: Given to students, have due dates, tracked for completion
- Tasks: Teacher's personal to-do items, not linked to students
- Different models but similar structure

**What is a join table?**
- Links two models in many-to-many relationship
- An assignment can have many students
- A student can have many assignments
- Join table tracks which students have which assignments

**How to track completion status?**
- Join table can store additional data
- Not just "does relationship exist" but "what's the status"
- Completed date, grade, notes all in join table

---

### Architecture: Assignments and Tasks

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

### What to Build

**1. Assignment Model**

Create model for assignments with:

**Data Fields:**
- Title and description
- Subject area
- Due date
- Estimated time to complete
- Instructions or requirements
- Link to teacher

**Associations:**
- Belongs to teacher
- Has many students through join table
- Optionally belongs to a subject

**Validations:**
- Title required
- Due date required
- Subject must be valid
- Teacher must exist

**Scopes:**
- Upcoming assignments
- Past due assignments
- Assignments due in a date range
- Assignments for a subject

**Methods:**
- Check if overdue
- Count students assigned
- Count students completed
- Completion percentage

---

**2. Student Assignment Model (Join Table)**

Create join table model with:

**Data Fields:**
- Link to student
- Link to assignment
- Status (not started, in progress, completed, graded)
- Completion timestamp
- Grade or score
- Teacher notes

**Validations:**
- Student and assignment required
- Unique combination of student + assignment
- Status must be valid value
- Grade must be in valid range (if present)

**Scopes:**
- Completed assignments
- Graded assignments
- Not started assignments
- Assignments for a specific student

**Methods:**
- Mark as completed
- Mark as graded
- Check if overdue

---

**3. Task Model**

Create model for teacher tasks with:

**Data Fields:**
- Title and description
- Due date
- Priority level
- Status
- Link to teacher

**Associations:**
- Belongs to teacher

**Validations:**
- Title required
- Priority must be valid
- Status must be valid

**Scopes:**
- Incomplete tasks
- High priority tasks
- Due today
- Overdue tasks

**Methods:**
- Mark as complete
- Check if overdue

---

**4. Database Migrations**

Create migrations for:

**Assignments table:**
- UUID primary key
- Teacher foreign key
- Title, description
- Subject
- Due date
- Estimated time
- Instructions
- Timestamps

**Student assignments table:**
- UUID primary key
- Student foreign key
- Assignment foreign key
- Status
- Completed timestamp
- Grade
- Notes
- Timestamps

**Tasks table:**
- UUID primary key
- Teacher foreign key
- Title, description
- Due date
- Priority
- Status
- Timestamps

**Indexes:**
- Teacher IDs on all tables
- Due dates for sorting
- Status columns for filtering
- Composite indexes for common queries

Research:
- How to create join tables in Rails?
- How to add enum columns?
- Best practices for indexing foreign keys?

---

**5. Assignments Controller**

Create endpoints:

**List Assignments:**
- Filter by subject
- Filter by date range
- Include completion statistics
- Support pagination

**Get Assignment:**
- Show assignment details
- Include list of assigned students with status
- Show completion statistics

**Create Assignment:**
- Accept assignment details
- Accept list of student IDs to assign to
- Create student assignments
- Return assignment with assignments

**Update Assignment:**
- Update assignment details
- Handle adding/removing students
- Don't affect existing completion status

**Delete Assignment:**
- Remove assignment
- Remove all student assignments
- Handle cascading deletion

**Assign to Students:**
- Add assignment to specific students
- Don't duplicate existing assignments
- Return updated assignment list

**Unassign from Students:**
- Remove assignment from specific students
- Mark as removed, not deleted (for history)

---

**6. Student Assignments Controller**

Create endpoints:

**List for Student:**
- Show all assignments for a student
- Filter by status
- Filter by date range
- Support pagination

**Update Status:**
- Mark as started, completed, graded
- Set completion date
- Add grade
- Add notes

**Bulk Update:**
- Update multiple student assignments at once
- Mark multiple as graded with same grade

---

**7. Tasks Controller**

Create standard CRUD endpoints:

**List Tasks:**
- Filter by status
- Filter by priority
- Sort by due date
- Support pagination

**Create/Update/Delete:**
- Standard operations
- Validate data

**Mark Complete:**
- Toggle completion status
- Set completion timestamp

---

### What to Test

**Assignment Model Tests:**
- Cannot create without teacher
- Cannot create without title or due date
- Can assign to multiple students
- Calculates completion percentage correctly
- Identifies overdue assignments
- Scopes filter correctly

**Student Assignment Model Tests:**
- Cannot create duplicate student + assignment
- Status transitions work correctly
- Can add grade
- Can add notes
- Completion timestamp set automatically

**Task Model Tests:**
- Cannot create without required fields
- Priority validation works
- Status validation works
- Can mark complete
- Overdue detection works

**Controllers Tests:**
- Can create assignments
- Can assign to students
- Can update assignment status
- Can grade assignments
- Students only see their own assignments
- Teachers only see their own assignments/tasks
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

### Concepts to Understand

**How to model academic progress?**
- Report cards are snapshots in time
- Each card has multiple subject grades
- Grades use consistent scale (letter or numeric)
- Comments provide qualitative feedback

**What is a nested resource?**
- Report cards belong to students
- Grades belong to report cards
- API reflects this hierarchy
- `/students/:id/report_cards/:id`

**How to aggregate data?**
- Calculate GPA from multiple grades
- Track progress over time
- Compare current to previous periods

---

### Architecture: Report Cards

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

### What to Build

**1. Report Card Model**

Create model for report cards with:

**Data Fields:**
- Link to student
- Link to teacher
- Grading period (Q1, Q2, Semester, Year, etc.)
- Start and end dates
- Overall comments
- Status (draft, finalized)

**Associations:**
- Belongs to student
- Belongs to teacher
- Has many subject grades

**Validations:**
- Student and teacher required
- Period required
- End date after start date
- No overlapping periods for same student

**Scopes:**
- Finalized report cards
- Draft report cards
- For a specific grading period

**Methods:**
- Calculate overall GPA
- Finalize report card (lock it)
- Check if finalized

---

**2. Subject Grade Model**

Create model for individual subject grades:

**Data Fields:**
- Link to report card
- Subject name
- Letter grade
- Numeric grade (percentage)
- Comments
- Teacher notes (private)

**Associations:**
- Belongs to report card

**Validations:**
- Report card required
- Subject name required
- Grade must be valid (A-F or 0-100)
- Both letter and numeric grades consistent

**Methods:**
- Convert between letter and numeric grades
- Grade point value (for GPA)

---

**3. Database Migrations**

Create migrations for:

**Report cards table:**
- UUID primary key
- Teacher foreign key
- Student foreign key
- Grading period
- Start and end dates
- Overall comments
- Finalized timestamp
- Timestamps

**Subject grades table:**
- UUID primary key
- Report card foreign key
- Subject name
- Letter grade
- Numeric grade
- Comments
- Teacher notes
- Timestamps

**Indexes:**
- Teacher and student IDs
- Grading period
- Finalized status

Research:
- How to prevent overlapping date ranges?
- How to implement soft locks (finalized status)?

---

**4. Report Cards Controller**

Create endpoints:

**List Report Cards:**
- For a specific student
- Filter by grading period
- Filter by status (draft/finalized)
- Support pagination

**Get Report Card:**
- Show full details
- Include all subject grades
- Include student info
- Calculate GPA

**Create Report Card:**
- Create for a student
- Set grading period and dates
- Initialize as draft
- Optionally include initial grades

**Update Report Card:**
- Update comments
- Update grades (if not finalized)
- Cannot update if finalized

**Finalize Report Card:**
- Lock the report card
- Prevent further edits
- Set finalized timestamp

**Delete Report Card:**
- Only if draft
- Remove all subject grades
- Cannot delete finalized cards

---

**5. Subject Grades Controller**

Create endpoints nested under report cards:

**Add Grade:**
- Add subject grade to report card
- Validate grade values
- Cannot add to finalized card

**Update Grade:**
- Update grade or comments
- Cannot update on finalized card

**Delete Grade:**
- Remove subject grade
- Cannot delete from finalized card

---

### What to Test

**Report Card Model Tests:**
- Cannot create without student and teacher
- Cannot overlap grading periods for same student
- GPA calculates correctly from subject grades
- Cannot modify finalized report card
- Status transitions work correctly

**Subject Grade Model Tests:**
- Cannot create without report card
- Grade validation works (A-F, 0-100)
- Letter and numeric grades stay consistent
- Grade point calculation works

**Report Cards Controller Tests:**
- Can create draft report card
- Can add subject grades
- Can finalize report card
- Cannot edit finalized report card
- Can list report cards for student
- Can filter by status and period
- GPA appears in response
- Requires authentication and authorization

**Subject Grades Controller Tests:**
- Can add grades to draft report card
- Cannot add grades to finalized report card
- Can update grades on draft
- Cannot update on finalized
- Grade validation works
- Requires authentication

---

### Validation Checklist

- [ ] Report card model works
- [ ] Subject grades model works
- [ ] GPA calculation works
- [ ] Can create report cards with grades
- [ ] Can finalize report cards
- [ ] Finalized cards cannot be edited
- [ ] Can view report history
- [ ] All report card tests pass

---

## Phase 6: Expense Tracking

### Concepts to Understand

**Why track homeschool expenses?**
- Tax deductions (in some jurisdictions)
- Budget planning
- Grant/reimbursement documentation
- Curriculum cost analysis

**How to categorize expenses?**
- Flexible categories (user-defined)
- Default categories provided
- Multiple levels of categorization possible

**What about recurring expenses?**
- Subscriptions, memberships
- Different from one-time purchases
- Need to project future costs

---

### Architecture: Expense Tracking

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

### What to Build

**1. Expense Category Model**

Create model for expense categories:

**Data Fields:**
- Name
- Description
- Color (for UI)
- Link to teacher
- Parent category (for sub-categories)

**Associations:**
- Belongs to teacher
- Has many expenses
- Optionally belongs to parent category
- Has many child categories

**Validations:**
- Name required and unique per teacher
- Color must be valid hex code

**Scopes:**
- Top-level categories (no parent)
- Active categories

**Methods:**
- Full category path (Parent > Child)
- Total expenses in category

---

**2. Expense Model**

Create model for expenses:

**Data Fields:**
- Link to teacher
- Link to category
- Optional link to student
- Amount
- Date of expense
- Description
- Receipt image URL
- Is recurring flag
- Recurring frequency (if applicable)
- Vendor/merchant

**Associations:**
- Belongs to teacher
- Belongs to category
- Optionally belongs to student

**Validations:**
- Amount required and positive
- Date required
- Category required
- Teacher required

**Scopes:**
- In date range
- For a category
- For a student
- Recurring expenses
- One-time expenses

**Methods:**
- Check if recurring
- Calculate annual cost (for recurring)
- Belongs to category path

---

**3. Database Migrations**

Create migrations for:

**Expense categories table:**
- UUID primary key
- Teacher foreign key
- Parent category foreign key (self-referential)
- Name
- Description
- Color
- Timestamps

**Expenses table:**
- UUID primary key
- Teacher foreign key
- Category foreign key
- Student foreign key (optional)
- Amount (decimal with precision)
- Date
- Description
- Receipt URL
- Recurring boolean
- Recurring frequency
- Vendor
- Timestamps

**Indexes:**
- Teacher IDs
- Category ID
- Student ID
- Date (for range queries)
- Amount (for sorting)

Research:
- How to store currency values in database?
- How to handle self-referential associations?
- Best decimal precision for money?

---

**4. Expense Categories Controller**

Create endpoints:

**List Categories:**
- Show all categories for teacher
- Include hierarchy
- Include expense totals

**Create Category:**
- Accept category details
- Optionally set parent category
- Return created category

**Update/Delete:**
- Standard operations
- Handle category with expenses (prevent deletion)

---

**5. Expenses Controller**

Create endpoints:

**List Expenses:**
- Filter by date range
- Filter by category
- Filter by student
- Filter by recurring status
- Sort by date or amount
- Include pagination
- Calculate totals

**Get Expense:**
- Show details
- Include receipt URL
- Include category and student info

**Create Expense:**
- Accept expense details
- Upload receipt image
- Link to category and optionally student
- Return created expense

**Update Expense:**
- Update any field
- Handle receipt upload

**Delete Expense:**
- Remove expense
- Handle receipt cleanup

**Generate Report:**
- Expenses by category
- Expenses by month
- Expenses by student
- Recurring expense projection

---

### What to Test

**Expense Category Model Tests:**
- Cannot create without name
- Name must be unique per teacher
- Can nest categories
- Full path calculation works
- Total expenses calculation works

**Expense Model Tests:**
- Cannot create without required fields
- Amount must be positive
- Date must be valid
- Annual cost calculation for recurring expenses
- Category path lookup works

**Controllers Tests:**
- Can create categories
- Can create expenses
- Can upload receipts
- Can filter expenses by multiple criteria
- Can generate expense reports
- Totals calculate correctly
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
- [ ] All expense tests pass

---

## Phase 7: Lesson Planning

### Concepts to Understand

**What is a lesson plan?**
- Structured outline for teaching a topic
- Includes objectives, materials, activities, assessment
- Can be reused and adapted
- Links to curriculum standards

**How to organize curriculum?**
- Subjects contain units
- Units contain lessons
- Lessons can have resources attached
- Progress tracked at lesson level

**What about resource management?**
- Files, links, materials
- Reusable across lessons
- Need cloud storage integration

---

### Architecture: Lesson Planning

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

### What to Build

**1. Subject Model**

Create model for subjects:

**Data Fields:**
- Name
- Description
- Color for UI
- Link to teacher
- Grade level

**Associations:**
- Belongs to teacher
- Has many lesson plans

**Validations:**
- Name required
- Color must be valid

**Methods:**
- Count of lesson plans
- Count of completed lessons

---

**2. Lesson Plan Model**

Create model for lesson plans:

**Data Fields:**
- Link to teacher
- Link to subject
- Title
- Learning objectives (text/array)
- Required materials list
- Activity descriptions
- Assessment method
- Resources (files/links)
- Date taught (when used)
- Teacher notes
- Duration estimate

**Associations:**
- Belongs to teacher
- Belongs to subject

**Validations:**
- Title required
- Subject required
- Objectives required

**Scopes:**
- Upcoming lessons
- Completed lessons
- For a subject

**Methods:**
- Mark as taught
- Check if completed
- Duplicate lesson (for reuse)

---

**3. Database Migrations**

Create migrations for:

**Subjects table:**
- UUID primary key
- Teacher foreign key
- Name
- Description
- Color
- Grade level
- Timestamps

**Lesson plans table:**
- UUID primary key
- Teacher foreign key
- Subject foreign key
- Title
- Objectives (text or JSON)
- Materials (text or JSON)
- Activities (text)
- Assessment (text)
- Resources (JSON array)
- Date taught
- Notes
- Duration
- Timestamps

**Indexes:**
- Teacher and subject IDs
- Date taught
- Subject + date for sorting

Research:
- How to store arrays/lists in PostgreSQL?
- JSON vs text for structured data?
- File upload strategies?

---

**4. Subjects Controller**

Create standard CRUD endpoints:

**List Subjects:**
- Show all for teacher
- Include lesson count
- Support pagination

**Create/Update/Delete:**
- Standard operations
- Prevent deletion with lessons

---

**5. Lesson Plans Controller**

Create endpoints:

**List Lesson Plans:**
- Filter by subject
- Filter by taught/not taught
- Filter by date range
- Sort by date or title
- Support pagination

**Get Lesson Plan:**
- Show full details
- Include resources
- Include subject info

**Create Lesson Plan:**
- Accept all fields
- Handle file uploads for resources
- Return created plan

**Update Lesson Plan:**
- Update any field
- Handle file uploads
- Mark as taught with date

**Delete Lesson Plan:**
- Remove plan
- Clean up uploaded files

**Duplicate Lesson:**
- Copy lesson plan
- Clear taught date
- Create new copy

**Upload Resource:**
- Add file or link to lesson
- Return updated resource list

---

### What to Test

**Subject Model Tests:**
- Cannot create without name
- Can have multiple lesson plans
- Lesson count accurate

**Lesson Plan Model Tests:**
- Cannot create without required fields
- Can mark as taught
- Duplication creates independent copy
- Resources stored correctly

**Controllers Tests:**
- Can create subjects and lessons
- Can upload resources
- Can filter lessons
- Can duplicate lessons
- Taught date updates correctly
- Requires authentication
- Teachers only see own data

---

### Validation Checklist

- [ ] Can create subjects
- [ ] Can create lesson plans
- [ ] Can upload resources
- [ ] Can mark lessons as taught
- [ ] Can duplicate lessons
- [ ] Can filter and search lessons
- [ ] File uploads work
- [ ] All lesson planning tests pass

---

## Phase 8: Production Preparation

### Concepts to Understand

**What makes an API production-ready?**
- Error handling and logging
- Rate limiting
- Monitoring and alerts
- Automated backups
- Security hardening
- Performance optimization

**What is a health check endpoint?**
- Simple endpoint to verify service is running
- Used by load balancers and monitoring
- Checks database connectivity
- Returns service status

**How to handle errors gracefully?**
- Never expose stack traces to users
- Log errors for debugging
- Return consistent error format
- Alert on critical errors

---

### What to Build

**1. Error Handling**

Implement global error handling:

**Application-level error handling:**
- Catch all unhandled exceptions
- Log with context (user, request, timestamp)
- Return generic error to user
- Alert on 500 errors

**Custom error classes:**
- Authentication errors
- Authorization errors
- Validation errors
- Not found errors
- Rate limit errors

**Error logging:**
- Structured logs
- Error tracking service integration
- Request ID tracking

Research:
- How to implement global exception handling in Rails?
- What error tracking services are available?
- How to generate request IDs?

---

**2. Rate Limiting**

Implement request rate limiting:

**Per-IP rate limiting:**
- Limit requests per hour/day
- Return 429 status when exceeded
- Provide headers showing limit/remaining

**Per-user rate limiting:**
- Different limits for authenticated users
- Track by user ID

Research:
- What rate limiting strategies exist?
- How to implement in Rails?
- What's a reasonable limit?

---

**3. Health Checks**

Create health check endpoints:

**Basic health check:**
- Returns 200 if service is up
- No authentication required

**Detailed health check:**
- Check database connection
- Check Redis connection
- Check external service dependencies
- Return status of each

Research:
- Standard health check endpoint patterns?
- What should be included?

---

**4. Performance Optimization**

Optimize API performance:

**Database optimization:**
- Add missing indexes
- Optimize slow queries
- Use eager loading to avoid N+1 queries
- Add database connection pooling

**Caching:**
- Cache expensive queries
- Cache authentication lookups
- Set appropriate cache TTLs

**Background jobs:**
- Move slow operations to background
- Email sending
- File processing
- Report generation

Research:
- How to identify slow queries?
- What should be cached?
- How to implement background jobs?

---

**5. Security Hardening**

Implement security best practices:

**Headers:**
- Set security headers (HSTS, X-Frame-Options, etc.)
- Content Security Policy
- CORS properly configured

**Input validation:**
- Strong parameter filtering
- SQL injection prevention
- XSS prevention

**Authentication:**
- Enforce HTTPS only
- Short access token expiry
- Refresh token rotation

**Authorization:**
- Check ownership on all operations
- Prevent enumeration attacks
- Rate limit authentication attempts

Research:
- Rails security best practices
- OWASP top 10
- Security headers

---

**6. Monitoring and Alerts**

Set up monitoring:

**Application metrics:**
- Request rate
- Response time
- Error rate
- Database query time

**Business metrics:**
- User registrations
- Active users
- API usage

**Alerts:**
- Error rate spike
- Slow response time
- Database connection issues
- Service downtime

Research:
- What monitoring tools are available?
- What metrics matter most?
- How to set alert thresholds?

---

**7. Documentation**

Create API documentation:

**Endpoint documentation:**
- All endpoints listed
- Request/response examples
- Authentication requirements
- Error codes

**Getting started guide:**
- How to authenticate
- Example API calls
- Common workflows

**Deployment guide:**
- Environment setup
- Database migrations
- Environment variables
- SSL certificate setup

Research:
- API documentation tools
- Documentation best practices

---

**8. Deployment Setup**

Prepare for deployment:

**Database:**
- Production database created
- Backups configured
- Connection pooling set up

**Environment:**
- All environment variables set
- Secrets properly managed
- SSL certificates configured

**Hosting:**
- Application server configured
- Database server configured
- Redis server configured
- File storage configured

**CI/CD:**
- Automated testing on push
- Automated deployment on merge
- Rollback strategy

Research:
- Hosting options for Rails APIs
- CI/CD tools
- Database hosting
- File storage options

---

### What to Test

**Error Handling:**
- Unhandled exceptions return 500
- Error logs contain needed context
- No sensitive data in error responses

**Rate Limiting:**
- Requests are limited correctly
- Headers show remaining requests
- 429 returned when exceeded

**Health Checks:**
- Returns 200 when healthy
- Returns error when database unavailable
- No authentication required

**Performance:**
- No N+1 queries
- Indexes improve query speed
- Caching works correctly

**Security:**
- Security headers present
- HTTPS enforced
- SQL injection prevented
- XSS prevented
- Authorization checks work

---

### Validation Checklist

- [ ] Global error handling implemented
- [ ] Errors logged with context
- [ ] Rate limiting works
- [ ] Health check endpoints exist
- [ ] Database indexes optimized
- [ ] N+1 queries eliminated
- [ ] Caching implemented
- [ ] Background jobs set up
- [ ] Security headers configured
- [ ] HTTPS enforced
- [ ] Monitoring configured
- [ ] API documentation complete
- [ ] Deployment guide written
- [ ] Production environment ready
- [ ] Backups configured
- [ ] All production tests pass

---

## Conclusion

You've now learned the concepts and architecture behind building a complete Rails API. The key to mastery is:

1. **Understand why** before building
2. **Research solutions** rather than copying
3. **Test thoroughly** at each step
4. **Validate understanding** through working code

Good luck building your Homeschool Master API!

---

## Appendix: Learning Resources

**Rails Documentation:**
- Rails Guides: https://guides.rubyonrails.org
- Rails API Documentation: https://api.rubyonrails.org

**Ruby Documentation:**
- Ruby Documentation: https://ruby-doc.org
- Ruby Style Guide: https://rubystyle.guide

**PostgreSQL:**
- PostgreSQL Documentation: https://www.postgresql.org/docs

**Testing:**
- RSpec Documentation: https://rspec.info
- Testing best practices

**Security:**
- OWASP Top 10: https://owasp.org
- Rails Security Guide

**Deployment:**
- 12 Factor App: https://12factor.net
- Various hosting platform documentation

**General:**
- Stack Overflow for specific questions
- GitHub for example code
- Dev.to and Medium for tutorials
