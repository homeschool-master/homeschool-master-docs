# Homeschool Management App - Rails API Implementation Guide

**Version:** 1.0  
**Date:** November 30, 2025  
**Status:** Implementation Ready  
**Framework:** Ruby on Rails 7 (API Mode)  
**Database:** PostgreSQL

---

## Table of Contents
1. [Project Setup](#phase-0-project-setup)
2. [Phase 1: Authentication](#phase-1-authentication)
3. [Phase 2: Teachers & Students](#phase-2-teachers--students)
4. [Phase 3: Subjects & Calendar](#phase-3-subjects--calendar)
5. [Phase 4: Assignments & Tasks](#phase-4-assignments--tasks)
6. [Phase 5: Report Cards](#phase-5-report-cards)
7. [Phase 6: Expenses](#phase-6-expenses)
8. [Phase 7: Lesson Plans](#phase-7-lesson-plans)
9. [Testing Checklist](#testing-checklist)
10. [Deployment Preparation](#deployment-preparation)

---

## Phase 0: Project Setup

### 0.1 Create New Rails API Project

```bash
# Create new Rails API-only application
rails new homeschool-api --api --database=postgresql -T

# Navigate to project
cd homeschool-api
```

### 0.2 Update Gemfile

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
JWT_SECRET_KEY=your-super-secret-key-change-in-production
JWT_ACCESS_TOKEN_EXPIRY=3600
JWT_REFRESH_TOKEN_EXPIRY=2592000

# Redis
REDIS_URL=redis://localhost:6379/0

# AWS S3 (for file uploads)
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
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

---

## Phase 1: Authentication

### 1.1 Create JWT Service

Create `app/services/jwt_service.rb`:

```ruby
class JwtService
  ALGORITHM = 'HS256'
  
  class << self
    def encode(payload, exp: nil)
      payload[:exp] = exp || access_token_expiry
      payload[:iat] = Time.now.to_i
      
      JWT.encode(payload, secret_key, ALGORITHM)
    end
    
    def encode_refresh_token(teacher_id)
      payload = {
        teacher_id: teacher_id,
        type: 'refresh',
        exp: refresh_token_expiry,
        iat: Time.now.to_i,
        jti: SecureRandom.uuid  # Unique token ID for revocation
      }
      
      JWT.encode(payload, secret_key, ALGORITHM)
    end
    
    def decode(token)
      decoded = JWT.decode(token, secret_key, true, { algorithm: ALGORITHM })
      HashWithIndifferentAccess.new(decoded.first)
    rescue JWT::ExpiredSignature
      nil
    rescue JWT::DecodeError
      nil
    end
    
    def decode_refresh_token(token)
      decoded = decode(token)
      return nil unless decoded
      return nil unless decoded[:type] == 'refresh'
      
      decoded
    end
    
    private
    
    def secret_key
      ENV.fetch('JWT_SECRET_KEY') { Rails.application.credentials.secret_key_base }
    end
    
    def access_token_expiry
      Time.now.to_i + ENV.fetch('JWT_ACCESS_TOKEN_EXPIRY', 3600).to_i
    end
    
    def refresh_token_expiry
      Time.now.to_i + ENV.fetch('JWT_REFRESH_TOKEN_EXPIRY', 2592000).to_i
    end
  end
end
```

### 1.2 Create Teacher Model & Migration

```bash
rails generate model Teacher \
  first_name:string \
  last_name:string \
  nickname:string \
  email:string \
  phone:string \
  newsletter_subscribed:boolean \
  password_digest:string \
  profile_image_url:string \
  email_verified_at:datetime \
  email_verification_token:string \
  password_reset_token:string \
  password_reset_sent_at:datetime \
  is_active:boolean
```

Update the migration file `db/migrate/XXXXXX_create_teachers.rb`:

```ruby
class CreateTeachers < ActiveRecord::Migration[7.1]
  def change
    create_table :teachers, id: :uuid do |t|
      t.string :first_name, null: false
      t.string :last_name, null: false
      t.string :nickname
      t.string :email, null: false
      t.string :phone
      t.boolean :newsletter_subscribed, default: false
      t.string :password_digest, null: false
      t.string :profile_image_url
      t.datetime :email_verified_at
      t.string :email_verification_token
      t.string :password_reset_token
      t.datetime :password_reset_sent_at
      t.boolean :is_active, default: true

      t.timestamps
    end
    
    add_index :teachers, :email, unique: true
    add_index :teachers, :email_verification_token, unique: true
    add_index :teachers, :password_reset_token, unique: true
    add_index :teachers, :is_active
  end
end
```

### 1.3 Enable UUID Support

Create a migration to enable UUID:

```bash
rails generate migration EnableUuidExtension
```

Update the migration:

```ruby
class EnableUuidExtension < ActiveRecord::Migration[7.1]
  def change
    enable_extension 'pgcrypto'
  end
end
```

Run migrations:

```bash
rails db:migrate
```

### 1.4 Update Teacher Model

Update `app/models/teacher.rb`:

```ruby
class Teacher < ApplicationRecord
  has_secure_password
  
  # Associations (will be added later)
  has_many :students, dependent: :destroy
  has_many :calendar_events, dependent: :destroy
  has_many :assignments, dependent: :destroy
  has_many :tasks, dependent: :destroy
  has_many :report_cards, dependent: :destroy
  has_many :expenses, dependent: :destroy
  has_many :subjects, dependent: :destroy
  has_many :lesson_plans, dependent: :destroy
  has_many :expense_categories, dependent: :destroy
  
  # Validations
  validates :first_name, presence: true, length: { maximum: 100 }
  validates :last_name, presence: true, length: { maximum: 100 }
  validates :nickname, length: { maximum: 100 }, allow_blank: true
  validates :email, presence: true, 
                    uniqueness: { case_sensitive: false },
                    format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, 
                       length: { minimum: 8 }, 
                       if: :password_required?
  validates :phone, length: { maximum: 20 }, allow_blank: true
  
  # Callbacks
  before_save :downcase_email
  before_create :generate_email_verification_token
  
  # Scopes
  scope :active, -> { where(is_active: true) }
  scope :verified, -> { where.not(email_verified_at: nil) }
  
  # Instance methods
  def full_name
    "#{first_name} #{last_name}"
  end
  
  def display_name
    nickname.presence || full_name
  end
  
  def email_verified?
    email_verified_at.present?
  end
  
  def verify_email!
    update!(
      email_verified_at: Time.current,
      email_verification_token: nil
    )
  end
  
  def generate_password_reset_token!
    update!(
      password_reset_token: SecureRandom.urlsafe_base64(32),
      password_reset_sent_at: Time.current
    )
  end
  
  def password_reset_token_valid?
    return false if password_reset_token.blank? || password_reset_sent_at.blank?
    
    password_reset_sent_at > 2.hours.ago
  end
  
  def clear_password_reset_token!
    update!(
      password_reset_token: nil,
      password_reset_sent_at: nil
    )
  end
  
  # Class methods
  def self.find_by_email(email)
    find_by(email: email.downcase)
  end
  
  private
  
  def downcase_email
    self.email = email.downcase
  end
  
  def generate_email_verification_token
    self.email_verification_token = SecureRandom.urlsafe_base64(32)
  end
  
  def password_required?
    new_record? || password.present?
  end
end
```

### 1.5 Create Refresh Token Model

```bash
rails generate model RefreshToken \
  teacher:references \
  token:string \
  jti:string \
  expires_at:datetime \
  revoked_at:datetime
```

Update the migration:

```ruby
class CreateRefreshTokens < ActiveRecord::Migration[7.1]
  def change
    create_table :refresh_tokens, id: :uuid do |t|
      t.references :teacher, null: false, foreign_key: true, type: :uuid
      t.string :token, null: false
      t.string :jti, null: false
      t.datetime :expires_at, null: false
      t.datetime :revoked_at

      t.timestamps
    end
    
    add_index :refresh_tokens, :token, unique: true
    add_index :refresh_tokens, :jti, unique: true
    add_index :refresh_tokens, :expires_at
  end
end
```

Update `app/models/refresh_token.rb`:

```ruby
class RefreshToken < ApplicationRecord
  belongs_to :teacher
  
  validates :token, presence: true, uniqueness: true
  validates :jti, presence: true, uniqueness: true
  validates :expires_at, presence: true
  
  scope :active, -> { where(revoked_at: nil).where('expires_at > ?', Time.current) }
  
  def expired?
    expires_at <= Time.current
  end
  
  def revoked?
    revoked_at.present?
  end
  
  def valid_token?
    !expired? && !revoked?
  end
  
  def revoke!
    update!(revoked_at: Time.current)
  end
  
  def self.find_valid_by_token(token)
    active.find_by(token: token)
  end
  
  def self.revoke_all_for_teacher(teacher_id)
    where(teacher_id: teacher_id).update_all(revoked_at: Time.current)
  end
end
```

Run migration:

```bash
rails db:migrate
```

### 1.6 Create Authentication Controller

Create `app/controllers/api/v1/auth/authentication_controller.rb`:

```ruby
module Api
  module V1
    module Auth
      class AuthenticationController < ApplicationController
        skip_before_action :verify_authenticity_token, raise: false
        before_action :authenticate_request, only: [:logout]
        
        # POST /api/v1/auth/register
        def register
          teacher = Teacher.new(register_params)
          
          if teacher.save
            tokens = generate_tokens(teacher)
            render_success(
              {
                teacher: teacher_response(teacher),
                tokens: tokens
              },
              status: :created
            )
          else
            render_validation_errors(teacher)
          end
        end
        
        # POST /api/v1/auth/login
        def login
          teacher = Teacher.find_by_email(params[:email])
          
          if teacher.nil?
            render_error("Invalid email or password", code: "INVALID_CREDENTIALS", status: :unauthorized)
            return
          end
          
          unless teacher.is_active?
            render_error("Account is deactivated", code: "ACCOUNT_DEACTIVATED", status: :unauthorized)
            return
          end
          
          unless teacher.authenticate(params[:password])
            render_error("Invalid email or password", code: "INVALID_CREDENTIALS", status: :unauthorized)
            return
          end
          
          tokens = generate_tokens(teacher)
          render_success(
            {
              teacher: teacher_response(teacher),
              tokens: tokens
            }
          )
        end
        
        # POST /api/v1/auth/refresh
        def refresh
          token = params[:refresh_token]
          
          if token.blank?
            render_error("Refresh token is required", code: "MISSING_TOKEN", status: :bad_request)
            return
          end
          
          refresh_token = RefreshToken.find_valid_by_token(token)
          
          if refresh_token.nil?
            render_error("Invalid or expired refresh token", code: "TOKEN_EXPIRED", status: :unauthorized)
            return
          end
          
          teacher = refresh_token.teacher
          
          unless teacher.is_active?
            render_error("Account is deactivated", code: "ACCOUNT_DEACTIVATED", status: :unauthorized)
            return
          end
          
          # Generate new access token
          access_token = JwtService.encode(teacher_id: teacher.id)
          
          render_success(
            {
              access_token: access_token,
              expires_in: ENV.fetch('JWT_ACCESS_TOKEN_EXPIRY', 3600).to_i,
              token_type: "Bearer"
            }
          )
        end
        
        # POST /api/v1/auth/logout
        def logout
          token = params[:refresh_token]
          
          if token.present?
            refresh_token = RefreshToken.find_by(token: token)
            refresh_token&.revoke!
          end
          
          render_success({ message: "Successfully logged out" })
        end
        
        private
        
        def register_params
          params.permit(
            :first_name,
            :last_name,
            :email,
            :password,
            :phone,
            :newsletter_subscribed
          )
        end
        
        def generate_tokens(teacher)
          # Generate access token
          access_token = JwtService.encode(teacher_id: teacher.id)
          
          # Generate refresh token
          jti = SecureRandom.uuid
          refresh_token_value = SecureRandom.urlsafe_base64(64)
          expires_at = Time.current + ENV.fetch('JWT_REFRESH_TOKEN_EXPIRY', 2592000).to_i.seconds
          
          RefreshToken.create!(
            teacher: teacher,
            token: refresh_token_value,
            jti: jti,
            expires_at: expires_at
          )
          
          {
            access_token: access_token,
            refresh_token: refresh_token_value,
            expires_in: ENV.fetch('JWT_ACCESS_TOKEN_EXPIRY', 3600).to_i,
            token_type: "Bearer"
          }
        end
        
        def teacher_response(teacher)
          {
            id: teacher.id,
            first_name: teacher.first_name,
            last_name: teacher.last_name,
            nickname: teacher.nickname,
            email: teacher.email,
            phone: teacher.phone,
            newsletter_subscribed: teacher.newsletter_subscribed,
            profile_image_url: teacher.profile_image_url,
            created_at: teacher.created_at.iso8601,
            updated_at: teacher.updated_at.iso8601
          }
        end
        
        def render_success(data, status: :ok)
          render json: { success: true, data: data }, status: status
        end
        
        def render_error(message, code:, status:)
          render json: {
            success: false,
            error: { code: code, message: message }
          }, status: status
        end
        
        def render_validation_errors(model)
          render json: {
            success: false,
            error: {
              code: "VALIDATION_ERROR",
              message: "Validation failed",
              details: model.errors.messages
            }
          }, status: :unprocessable_entity
        end
        
        def authenticate_request
          token = extract_token_from_header
          
          if token.nil?
            render_error("Missing authentication token", code: "UNAUTHORIZED", status: :unauthorized)
            return
          end
          
          decoded = JwtService.decode(token)
          
          if decoded.nil?
            render_error("Invalid or expired token", code: "TOKEN_EXPIRED", status: :unauthorized)
            return
          end
          
          @current_teacher = Teacher.find_by(id: decoded[:teacher_id])
          
          if @current_teacher.nil?
            render_error("Teacher not found", code: "UNAUTHORIZED", status: :unauthorized)
          end
        end
        
        def extract_token_from_header
          header = request.headers['Authorization']
          return nil unless header
          
          header.split(' ').last
        end
      end
    end
  end
end
```

### 1.7 Create Password Controller

Create `app/controllers/api/v1/auth/passwords_controller.rb`:

```ruby
module Api
  module V1
    module Auth
      class PasswordsController < ApplicationController
        skip_before_action :verify_authenticity_token, raise: false
        before_action :authenticate_request, only: [:change]
        
        # POST /api/v1/auth/password/reset-request
        def reset_request
          teacher = Teacher.find_by_email(params[:email])
          
          # Always return success to prevent email enumeration
          if teacher.present? && teacher.is_active?
            teacher.generate_password_reset_token!
            # TODO: Send password reset email
            # PasswordMailer.reset_email(teacher).deliver_later
          end
          
          render_success({
            message: "If an account exists with this email, a password reset link has been sent."
          })
        end
        
        # POST /api/v1/auth/password/reset
        def reset
          token = params[:reset_token]
          new_password = params[:new_password]
          
          if token.blank? || new_password.blank?
            render_error("Reset token and new password are required", 
                        code: "VALIDATION_ERROR", status: :bad_request)
            return
          end
          
          teacher = Teacher.find_by(password_reset_token: token)
          
          if teacher.nil? || !teacher.password_reset_token_valid?
            render_error("Reset token is invalid or has expired", 
                        code: "INVALID_TOKEN", status: :bad_request)
            return
          end
          
          if new_password.length < 8
            render_error("Password must be at least 8 characters", 
                        code: "VALIDATION_ERROR", status: :unprocessable_entity)
            return
          end
          
          teacher.password = new_password
          teacher.clear_password_reset_token!
          
          # Revoke all existing refresh tokens
          RefreshToken.revoke_all_for_teacher(teacher.id)
          
          render_success({
            message: "Password successfully reset. Please log in with your new password."
          })
        end
        
        # POST /api/v1/auth/password/change
        def change
          current_password = params[:current_password]
          new_password = params[:new_password]
          
          if current_password.blank? || new_password.blank?
            render_error("Current password and new password are required", 
                        code: "VALIDATION_ERROR", status: :bad_request)
            return
          end
          
          unless @current_teacher.authenticate(current_password)
            render_error("Current password is incorrect", 
                        code: "INVALID_CREDENTIALS", status: :unauthorized)
            return
          end
          
          if new_password.length < 8
            render_error("Password must be at least 8 characters", 
                        code: "VALIDATION_ERROR", status: :unprocessable_entity)
            return
          end
          
          @current_teacher.password = new_password
          @current_teacher.save!
          
          render_success({ message: "Password successfully changed" })
        end
        
        private
        
        def render_success(data, status: :ok)
          render json: { success: true, data: data }, status: status
        end
        
        def render_error(message, code:, status:)
          render json: {
            success: false,
            error: { code: code, message: message }
          }, status: status
        end
        
        def authenticate_request
          token = extract_token_from_header
          
          if token.nil?
            render_error("Missing authentication token", code: "UNAUTHORIZED", status: :unauthorized)
            return
          end
          
          decoded = JwtService.decode(token)
          
          if decoded.nil?
            render_error("Invalid or expired token", code: "TOKEN_EXPIRED", status: :unauthorized)
            return
          end
          
          @current_teacher = Teacher.find_by(id: decoded[:teacher_id])
          
          if @current_teacher.nil?
            render_error("Teacher not found", code: "UNAUTHORIZED", status: :unauthorized)
          end
        end
        
        def extract_token_from_header
          header = request.headers['Authorization']
          return nil unless header
          
          header.split(' ').last
        end
      end
    end
  end
end
```

### 1.8 Create Teachers Controller (Profile)

Create `app/controllers/api/v1/teachers_controller.rb`:

```ruby
module Api
  module V1
    class TeachersController < BaseController
      # GET /api/v1/teachers/me
      def me
        render_success(teacher_response(current_teacher))
      end
      
      # PUT /api/v1/teachers/me
      def update
        if current_teacher.update(teacher_params)
          render_success(teacher_response(current_teacher))
        else
          render_validation_errors(current_teacher)
        end
      end
      
      # DELETE /api/v1/teachers/me
      def destroy
        password = params[:password]
        confirmation = params[:confirmation]
        
        unless password.present? && confirmation == "DELETE"
          render_error(
            "Password and confirmation are required",
            code: "VALIDATION_ERROR",
            status: :bad_request
          )
          return
        end
        
        unless current_teacher.authenticate(password)
          render_error(
            "Password is incorrect",
            code: "INVALID_CREDENTIALS",
            status: :unauthorized
          )
          return
        end
        
        # Soft delete (deactivate) instead of hard delete
        current_teacher.update!(is_active: false)
        
        # Revoke all refresh tokens
        RefreshToken.revoke_all_for_teacher(current_teacher.id)
        
        render_no_content
      end
      
      private
      
      def teacher_params
        params.permit(
          :first_name,
          :last_name,
          :nickname,
          :phone,
          :newsletter_subscribed
        )
      end
      
      def teacher_response(teacher)
        {
          id: teacher.id,
          first_name: teacher.first_name,
          last_name: teacher.last_name,
          nickname: teacher.nickname,
          email: teacher.email,
          phone: teacher.phone,
          newsletter_subscribed: teacher.newsletter_subscribed,
          profile_image_url: teacher.profile_image_url,
          created_at: teacher.created_at.iso8601,
          updated_at: teacher.updated_at.iso8601
        }
      end
    end
  end
end
```

### 1.9 Update Routes for Teachers

Add to `config/routes.rb` inside the `api/v1` namespace:

```ruby
# Teacher profile
get 'teachers/me', to: 'teachers#me'
put 'teachers/me', to: 'teachers#update'
delete 'teachers/me', to: 'teachers#destroy'
```

### ✅ Phase 1 Checklist

- [ ] Created JWT service
- [ ] Created Teacher model and migration
- [ ] Enabled UUID extension
- [ ] Created RefreshToken model
- [ ] Created AuthenticationController
- [ ] Created PasswordsController
- [ ] Created TeachersController
- [ ] Updated routes
- [ ] Run migrations
- [ ] Test with curl/Postman:
  - [ ] POST /api/v1/auth/register
  - [ ] POST /api/v1/auth/login
  - [ ] POST /api/v1/auth/refresh
  - [ ] POST /api/v1/auth/logout
  - [ ] GET /api/v1/teachers/me
  - [ ] PUT /api/v1/teachers/me

---

## Phase 2: Teachers & Students

### 2.1 Create Student Model & Migration

```bash
rails generate model Student \
  teacher:references \
  first_name:string \
  last_name:string \
  nickname:string \
  date_of_birth:date \
  grade_level:string \
  profile_image_url:string \
  notes:text \
  is_active:boolean
```

Update the migration:

```ruby
class CreateStudents < ActiveRecord::Migration[7.1]
  def change
    create_table :students, id: :uuid do |t|
      t.references :teacher, null: false, foreign_key: true, type: :uuid
      t.string :first_name, null: false
      t.string :last_name, null: false
      t.string :nickname
      t.date :date_of_birth
      t.string :grade_level
      t.string :profile_image_url
      t.text :notes
      t.boolean :is_active, default: true

      t.timestamps
    end
    
    add_index :students, [:teacher_id, :is_active]
    add_index :students, :is_active
  end
end
```

Run migration:

```bash
rails db:migrate
```

### 2.2 Update Student Model

Update `app/models/student.rb`:

```ruby
class Student < ApplicationRecord
  belongs_to :teacher
  
  # Future associations
  has_many :assignments, dependent: :destroy
  has_many :tasks, dependent: :destroy
  has_many :report_cards, dependent: :destroy
  has_many :expenses, dependent: :nullify
  has_many :event_attendees, dependent: :destroy
  has_many :calendar_events, through: :event_attendees
  
  # Validations
  validates :first_name, presence: true, length: { maximum: 100 }
  validates :last_name, presence: true, length: { maximum: 100 }
  validates :nickname, length: { maximum: 100 }, allow_blank: true
  validates :grade_level, length: { maximum: 20 }, allow_blank: true
  
  # Scopes
  scope :active, -> { where(is_active: true) }
  scope :by_grade, ->(grade) { where(grade_level: grade) if grade.present? }
  scope :ordered, -> { order(:first_name, :last_name) }
  
  # Instance methods
  def full_name
    "#{first_name} #{last_name}"
  end
  
  def display_name
    nickname.presence || full_name
  end
  
  def age
    return nil unless date_of_birth.present?
    
    now = Date.current
    age = now.year - date_of_birth.year
    age -= 1 if now.yday < date_of_birth.yday
    age
  end
end
```

### 2.3 Create Students Controller

Create `app/controllers/api/v1/students_controller.rb`:

```ruby
module Api
  module V1
    class StudentsController < BaseController
      before_action :set_student, only: [:show, :update, :destroy]
      
      # GET /api/v1/students
      def index
        students = current_teacher.students
        
        # Apply filters
        students = students.active if filter_params[:is_active] == 'true'
        students = students.by_grade(filter_params[:grade_level])
        
        # Apply sorting
        sort_by = filter_params[:sort_by] || 'first_name'
        sort_order = filter_params[:sort_order] == 'desc' ? :desc : :asc
        
        allowed_sort_columns = %w[first_name last_name grade_level created_at]
        sort_by = 'first_name' unless allowed_sort_columns.include?(sort_by)
        
        students = students.order(sort_by => sort_order)
        
        render_success(
          students.map { |s| student_response(s) },
          meta: { total: students.count }
        )
      end
      
      # GET /api/v1/students/:id
      def show
        render_success(student_response(@student))
      end
      
      # POST /api/v1/students
      def create
        student = current_teacher.students.new(student_params)
        
        if student.save
          render_created(student_response(student))
        else
          render_validation_errors(student)
        end
      end
      
      # PUT /api/v1/students/:id
      def update
        if @student.update(student_params)
          render_success(student_response(@student))
        else
          render_validation_errors(@student)
        end
      end
      
      # DELETE /api/v1/students/:id
      def destroy
        if params[:permanent] == 'true'
          @student.destroy
        else
          @student.update!(is_active: false)
        end
        
        render_no_content
      end
      
      private
      
      def set_student
        @student = current_teacher.students.find_by(id: params[:id])
        render_not_found("Student") unless @student
      end
      
      def student_params
        params.permit(
          :first_name,
          :last_name,
          :nickname,
          :date_of_birth,
          :grade_level,
          :notes
        )
      end
      
      def filter_params
        params.permit(:is_active, :grade_level, :sort_by, :sort_order)
      end
      
      def student_response(student)
        {
          id: student.id,
          teacher_id: student.teacher_id,
          first_name: student.first_name,
          last_name: student.last_name,
          nickname: student.nickname,
          date_of_birth: student.date_of_birth&.iso8601,
          grade_level: student.grade_level,
          profile_image_url: student.profile_image_url,
          notes: student.notes,
          is_active: student.is_active,
          created_at: student.created_at.iso8601,
          updated_at: student.updated_at.iso8601
        }
      end
    end
  end
end
```

### 2.4 Update Routes for Students

Add to `config/routes.rb` inside the `api/v1` namespace:

```ruby
resources :students, only: [:index, :show, :create, :update, :destroy]
```

### ✅ Phase 2 Checklist

- [ ] Created Student model and migration
- [ ] Updated Student model with validations and associations
- [ ] Created StudentsController
- [ ] Updated routes
- [ ] Run migrations
- [ ] Test with curl/Postman:
  - [ ] GET /api/v1/students
  - [ ] GET /api/v1/students/:id
  - [ ] POST /api/v1/students
  - [ ] PUT /api/v1/students/:id
  - [ ] DELETE /api/v1/students/:id

---

## Phase 3: Subjects & Calendar

### 3.1 Create Subject Model & Migration

```bash
rails generate model Subject \
  teacher:references \
  name:string \
  description:text \
  color_code:string \
  icon:string
```

Update migration:

```ruby
class CreateSubjects < ActiveRecord::Migration[7.1]
  def change
    create_table :subjects, id: :uuid do |t|
      t.references :teacher, null: false, foreign_key: true, type: :uuid
      t.string :name, null: false
      t.text :description
      t.string :color_code
      t.string :icon

      t.timestamps
    end
    
    add_index :subjects, [:teacher_id, :name]
  end
end
```

### 3.2 Create Event Type Model & Migration

```bash
rails generate model EventType \
  name:string \
  color_code:string \
  icon:string \
  description:text \
  is_system_default:boolean
```

Update migration:

```ruby
class CreateEventTypes < ActiveRecord::Migration[7.1]
  def change
    create_table :event_types, id: :uuid do |t|
      t.string :name, null: false
      t.string :color_code
      t.string :icon
      t.text :description
      t.boolean :is_system_default, default: false

      t.timestamps
    end
  end
end
```

### 3.3 Create Calendar Event Model & Migration

```bash
rails generate model CalendarEvent \
  teacher:references \
  event_type:references \
  title:string \
  description:text \
  location:string \
  start_time:datetime \
  end_time:datetime \
  all_day:boolean \
  recurrence_rule:text \
  recurrence_end_date:date \
  is_recurring:boolean \
  parent_event:references \
  color_code:string \
  reminder_minutes:integer
```

Update migration:

```ruby
class CreateCalendarEvents < ActiveRecord::Migration[7.1]
  def change
    create_table :calendar_events, id: :uuid do |t|
      t.references :teacher, null: false, foreign_key: true, type: :uuid
      t.references :event_type, foreign_key: true, type: :uuid
      t.string :title, null: false
      t.text :description
      t.string :location
      t.datetime :start_time, null: false
      t.datetime :end_time
      t.boolean :all_day, default: false
      t.text :recurrence_rule
      t.date :recurrence_end_date
      t.boolean :is_recurring, default: false
      t.uuid :parent_event_id
      t.string :color_code
      t.integer :reminder_minutes

      t.timestamps
    end
    
    add_index :calendar_events, :start_time
    add_index :calendar_events, [:teacher_id, :start_time]
    add_index :calendar_events, :is_recurring
    add_foreign_key :calendar_events, :calendar_events, column: :parent_event_id
  end
end
```

### 3.4 Create Event Attendee Model & Migration

```bash
rails generate model EventAttendee \
  calendar_event:references \
  student:references \
  attendance_status:string
```

Update migration:

```ruby
class CreateEventAttendees < ActiveRecord::Migration[7.1]
  def change
    create_table :event_attendees, id: :uuid do |t|
      t.references :calendar_event, null: false, foreign_key: true, type: :uuid
      t.references :student, null: false, foreign_key: true, type: :uuid
      t.string :attendance_status, default: 'pending'

      t.timestamps
    end
    
    add_index :event_attendees, [:calendar_event_id, :student_id], unique: true
    add_index :event_attendees, :student_id
  end
end
```

Run migrations:

```bash
rails db:migrate
```

### 3.5 Update Models

**Subject Model (`app/models/subject.rb`):**

```ruby
class Subject < ApplicationRecord
  belongs_to :teacher
  
  has_many :assignments, dependent: :nullify
  has_many :expenses, dependent: :nullify
  has_many :lesson_plans, dependent: :nullify
  has_many :report_card_entries, dependent: :nullify
  
  validates :name, presence: true, length: { maximum: 100 }
  validates :color_code, format: { with: /\A#[0-9A-Fa-f]{6}\z/ }, allow_blank: true
  
  scope :ordered, -> { order(:name) }
end
```

**EventType Model (`app/models/event_type.rb`):**

```ruby
class EventType < ApplicationRecord
  has_many :calendar_events, dependent: :nullify
  
  validates :name, presence: true, length: { maximum: 100 }
  validates :color_code, format: { with: /\A#[0-9A-Fa-f]{6}\z/ }, allow_blank: true
  
  scope :system_defaults, -> { where(is_system_default: true) }
  scope :ordered, -> { order(:name) }
end
```

**CalendarEvent Model (`app/models/calendar_event.rb`):**

```ruby
class CalendarEvent < ApplicationRecord
  belongs_to :teacher
  belongs_to :event_type, optional: true
  belongs_to :parent_event, class_name: 'CalendarEvent', optional: true
  
  has_many :child_events, class_name: 'CalendarEvent', foreign_key: :parent_event_id, dependent: :destroy
  has_many :event_attendees, dependent: :destroy
  has_many :students, through: :event_attendees
  has_many :assignments, dependent: :nullify
  
  validates :title, presence: true, length: { maximum: 255 }
  validates :start_time, presence: true
  validate :end_time_after_start_time
  validate :recurrence_rule_presence
  
  scope :for_date_range, ->(start_date, end_date) {
    where(start_time: start_date..end_date)
  }
  scope :upcoming, -> { where('start_time >= ?', Time.current).order(:start_time) }
  scope :recurring, -> { where(is_recurring: true) }
  
  def event_type_name
    event_type&.name
  end
  
  def attendees_list
    event_attendees.includes(:student).map do |ea|
      {
        student_id: ea.student_id,
        student_name: ea.student.display_name,
        attendance_status: ea.attendance_status
      }
    end
  end
  
  private
  
  def end_time_after_start_time
    return unless end_time.present? && start_time.present?
    
    if end_time <= start_time
      errors.add(:end_time, "must be after start time")
    end
  end
  
  def recurrence_rule_presence
    if is_recurring? && recurrence_rule.blank?
      errors.add(:recurrence_rule, "is required for recurring events")
    end
  end
end
```

**EventAttendee Model (`app/models/event_attendee.rb`):**

```ruby
class EventAttendee < ApplicationRecord
  belongs_to :calendar_event
  belongs_to :student
  
  ATTENDANCE_STATUSES = %w[pending attended absent excused].freeze
  
  validates :attendance_status, inclusion: { in: ATTENDANCE_STATUSES }
  validates :student_id, uniqueness: { scope: :calendar_event_id }
end
```

### 3.6 Create Subjects Controller

Create `app/controllers/api/v1/subjects_controller.rb`:

```ruby
module Api
  module V1
    class SubjectsController < BaseController
      before_action :set_subject, only: [:show, :update, :destroy]
      
      # GET /api/v1/subjects
      def index
        subjects = current_teacher.subjects.ordered
        render_success(subjects.map { |s| subject_response(s) })
      end
      
      # GET /api/v1/subjects/:id
      def show
        render_success(subject_response(@subject))
      end
      
      # POST /api/v1/subjects
      def create
        subject = current_teacher.subjects.new(subject_params)
        
        if subject.save
          render_created(subject_response(subject))
        else
          render_validation_errors(subject)
        end
      end
      
      # PUT /api/v1/subjects/:id
      def update
        if @subject.update(subject_params)
          render_success(subject_response(@subject))
        else
          render_validation_errors(@subject)
        end
      end
      
      # DELETE /api/v1/subjects/:id
      def destroy
        @subject.destroy
        render_no_content
      end
      
      private
      
      def set_subject
        @subject = current_teacher.subjects.find_by(id: params[:id])
        render_not_found("Subject") unless @subject
      end
      
      def subject_params
        params.permit(:name, :description, :color_code, :icon)
      end
      
      def subject_response(subject)
        {
          id: subject.id,
          teacher_id: subject.teacher_id,
          name: subject.name,
          description: subject.description,
          color_code: subject.color_code,
          icon: subject.icon,
          created_at: subject.created_at.iso8601,
          updated_at: subject.updated_at.iso8601
        }
      end
    end
  end
end
```

### 3.7 Create Calendar Events Controller

Create `app/controllers/api/v1/calendar_events_controller.rb`:

```ruby
module Api
  module V1
    class CalendarEventsController < BaseController
      before_action :set_event, only: [:show, :update, :destroy, :update_attendance]
      
      # GET /api/v1/calendar/events
      def index
        events = current_teacher.calendar_events.includes(:event_type, :event_attendees, :students)
        
        # Apply date filters
        if params[:start_date].present?
          start_date = DateTime.parse(params[:start_date])
          events = events.where('start_time >= ?', start_date)
        end
        
        if params[:end_date].present?
          end_date = DateTime.parse(params[:end_date])
          events = events.where('start_time <= ?', end_date)
        end
        
        # Apply student filter
        if params[:student_id].present?
          events = events.joins(:event_attendees)
                        .where(event_attendees: { student_id: params[:student_id] })
        end
        
        # Apply event type filter
        if params[:event_type_id].present?
          events = events.where(event_type_id: params[:event_type_id])
        end
        
        events = events.order(:start_time)
        
        result = paginate(events)
        
        render_success(
          result[:data].map { |e| event_response(e) },
          meta: result[:meta]
        )
      end
      
      # GET /api/v1/calendar/events/:id
      def show
        render_success(event_response(@event))
      end
      
      # POST /api/v1/calendar/events
      def create
        event = current_teacher.calendar_events.new(event_params)
        
        if event.save
          # Add attendees
          if params[:student_ids].present?
            params[:student_ids].each do |student_id|
              student = current_teacher.students.find_by(id: student_id)
              event.event_attendees.create!(student: student) if student
            end
          end
          
          render_created(event_response(event.reload))
        else
          render_validation_errors(event)
        end
      end
      
      # PUT /api/v1/calendar/events/:id
      def update
        if @event.update(event_params)
          # Update attendees if provided
          if params.key?(:student_ids)
            @event.event_attendees.destroy_all
            
            params[:student_ids]&.each do |student_id|
              student = current_teacher.students.find_by(id: student_id)
              @event.event_attendees.create!(student: student) if student
            end
          end
          
          render_success(event_response(@event.reload))
        else
          render_validation_errors(@event)
        end
      end
      
      # DELETE /api/v1/calendar/events/:id
      def destroy
        if params[:delete_series] == 'true' && @event.parent_event_id.nil? && @event.is_recurring?
          # Delete all events in series
          @event.child_events.destroy_all
        end
        
        @event.destroy
        render_no_content
      end
      
      # PATCH /api/v1/calendar/events/:id/attendance/:student_id
      def update_attendance
        attendee = @event.event_attendees.find_by(student_id: params[:student_id])
        
        unless attendee
          render_not_found("Attendee")
          return
        end
        
        if attendee.update(attendance_status: params[:attendance_status])
          render_success({
            event_id: @event.id,
            student_id: attendee.student_id,
            attendance_status: attendee.attendance_status,
            updated_at: attendee.updated_at.iso8601
          })
        else
          render_validation_errors(attendee)
        end
      end
      
      private
      
      def set_event
        @event = current_teacher.calendar_events.find_by(id: params[:id])
        render_not_found("Calendar event") unless @event
      end
      
      def event_params
        params.permit(
          :event_type_id,
          :title,
          :description,
          :location,
          :start_time,
          :end_time,
          :all_day,
          :is_recurring,
          :recurrence_rule,
          :recurrence_end_date,
          :color_code,
          :reminder_minutes
        )
      end
      
      def event_response(event)
        {
          id: event.id,
          teacher_id: event.teacher_id,
          event_type_id: event.event_type_id,
          event_type_name: event.event_type_name,
          title: event.title,
          description: event.description,
          location: event.location,
          start_time: event.start_time.iso8601,
          end_time: event.end_time&.iso8601,
          all_day: event.all_day,
          is_recurring: event.is_recurring,
          recurrence_rule: event.recurrence_rule,
          recurrence_end_date: event.recurrence_end_date&.iso8601,
          parent_event_id: event.parent_event_id,
          color_code: event.color_code,
          reminder_minutes: event.reminder_minutes,
          attendees: event.attendees_list,
          created_at: event.created_at.iso8601,
          updated_at: event.updated_at.iso8601
        }
      end
    end
  end
end
```

### 3.8 Create Event Types Controller

Create `app/controllers/api/v1/event_types_controller.rb`:

```ruby
module Api
  module V1
    class EventTypesController < BaseController
      # GET /api/v1/calendar/event-types
      def index
        event_types = EventType.ordered
        render_success(event_types.map { |et| event_type_response(et) })
      end
      
      # POST /api/v1/calendar/event-types
      def create
        event_type = EventType.new(event_type_params)
        event_type.is_system_default = false
        
        if event_type.save
          render_created(event_type_response(event_type))
        else
          render_validation_errors(event_type)
        end
      end
      
      private
      
      def event_type_params
        params.permit(:name, :color_code, :icon, :description)
      end
      
      def event_type_response(event_type)
        {
          id: event_type.id,
          name: event_type.name,
          color_code: event_type.color_code,
          icon: event_type.icon,
          description: event_type.description,
          is_system_default: event_type.is_system_default
        }
      end
    end
  end
end
```

### 3.9 Seed Default Event Types

Create `db/seeds/event_types.rb`:

```ruby
event_types = [
  { name: 'Lesson', color_code: '#4CAF50', icon: 'book', description: 'Regular lesson or teaching session' },
  { name: 'Field Trip', color_code: '#2196F3', icon: 'bus', description: 'Educational field trip or outing' },
  { name: 'Test', color_code: '#F44336', icon: 'pencil', description: 'Test or examination' },
  { name: 'Project Due', color_code: '#FF9800', icon: 'folder', description: 'Project deadline' },
  { name: 'Appointment', color_code: '#9C27B0', icon: 'calendar', description: 'General appointment' },
  { name: 'Break', color_code: '#607D8B', icon: 'coffee', description: 'Break or vacation' }
]

event_types.each do |attrs|
  EventType.find_or_create_by!(name: attrs[:name]) do |et|
    et.color_code = attrs[:color_code]
    et.icon = attrs[:icon]
    et.description = attrs[:description]
    et.is_system_default = true
  end
end

puts "Created #{EventType.count} event types"
```

Update `db/seeds.rb`:

```ruby
load Rails.root.join('db/seeds/event_types.rb')
```

Run seeds:

```bash
rails db:seed
```

### 3.10 Update Routes for Calendar

Add to `config/routes.rb` inside the `api/v1` namespace:

```ruby
resources :subjects, only: [:index, :show, :create, :update, :destroy]

namespace :calendar do
  resources :events, controller: 'calendar_events', only: [:index, :show, :create, :update, :destroy] do
    member do
      patch 'attendance/:student_id', to: 'calendar_events#update_attendance'
    end
  end
  resources :event_types, only: [:index, :create]
end
```

### ✅ Phase 3 Checklist

- [ ] Created Subject model and migration
- [ ] Created EventType model and migration
- [ ] Created CalendarEvent model and migration
- [ ] Created EventAttendee model and migration
- [ ] Updated all models
- [ ] Created SubjectsController
- [ ] Created CalendarEventsController
- [ ] Created EventTypesController
- [ ] Seeded default event types
- [ ] Updated routes
- [ ] Run migrations and seeds
- [ ] Test endpoints

---

## Phase 4: Assignments & Tasks

### 4.1 Create Assignment Model & Migration

```bash
rails generate model Assignment \
  student:references \
  teacher:references \
  subject:references \
  calendar_event:references \
  title:string \
  description:text \
  due_date:datetime \
  assigned_date:date \
  completion_status:string \
  completed_date:datetime \
  grade:string \
  points_earned:decimal \
  points_possible:decimal \
  notes:text \
  attachments:jsonb
```

Update migration:

```ruby
class CreateAssignments < ActiveRecord::Migration[7.1]
  def change
    create_table :assignments, id: :uuid do |t|
      t.references :student, null: false, foreign_key: true, type: :uuid
      t.references :teacher, null: false, foreign_key: true, type: :uuid
      t.references :subject, foreign_key: true, type: :uuid
      t.references :calendar_event, foreign_key: true, type: :uuid
      t.string :title, null: false
      t.text :description
      t.datetime :due_date
      t.date :assigned_date, null: false
      t.string :completion_status, default: 'not_started'
      t.datetime :completed_date
      t.string :grade
      t.decimal :points_earned, precision: 10, scale: 2
      t.decimal :points_possible, precision: 10, scale: 2
      t.text :notes
      t.jsonb :attachments, default: []

      t.timestamps
    end
    
    add_index :assignments, [:student_id, :due_date]
    add_index :assignments, [:student_id, :completion_status]
    add_index :assignments, :due_date
    add_index :assignments, :completion_status
  end
end
```

### 4.2 Create Task Model & Migration

```bash
rails generate model Task \
  teacher:references \
  student:references \
  title:string \
  description:text \
  due_date:datetime \
  priority:string \
  status:string \
  completed_date:datetime \
  category:string
```

Update migration:

```ruby
class CreateTasks < ActiveRecord::Migration[7.1]
  def change
    create_table :tasks, id: :uuid do |t|
      t.references :teacher, null: false, foreign_key: true, type: :uuid
      t.references :student, foreign_key: true, type: :uuid
      t.string :title, null: false
      t.text :description
      t.datetime :due_date
      t.string :priority, default: 'medium'
      t.string :status, default: 'pending'
      t.datetime :completed_date
      t.string :category

      t.timestamps
    end
    
    add_index :tasks, [:teacher_id, :status]
    add_index :tasks, [:student_id, :status]
    add_index :tasks, :due_date
    add_index :tasks, :status
  end
end
```

Run migrations:

```bash
rails db:migrate
```

### 4.3 Update Assignment Model

```ruby
class Assignment < ApplicationRecord
  belongs_to :student
  belongs_to :teacher
  belongs_to :subject, optional: true
  belongs_to :calendar_event, optional: true
  
  COMPLETION_STATUSES = %w[not_started in_progress completed overdue].freeze
  
  validates :title, presence: true, length: { maximum: 255 }
  validates :assigned_date, presence: true
  validates :completion_status, inclusion: { in: COMPLETION_STATUSES }
  validates :points_earned, numericality: { greater_than_or_equal_to: 0 }, allow_nil: true
  validates :points_possible, numericality: { greater_than: 0 }, allow_nil: true
  
  scope :for_student, ->(student_id) { where(student_id: student_id) if student_id.present? }
  scope :for_subject, ->(subject_id) { where(subject_id: subject_id) if subject_id.present? }
  scope :by_status, ->(status) { where(completion_status: status) if status.present? }
  scope :due_between, ->(start_date, end_date) {
    where(due_date: start_date..end_date) if start_date.present? && end_date.present?
  }
  scope :incomplete, -> { where.not(completion_status: 'completed') }
  scope :ordered_by_due_date, -> { order(due_date: :asc) }
  
  def subject_name
    subject&.name
  end
  
  def student_name
    student&.display_name
  end
  
  def mark_complete!(grade: nil, points_earned: nil, notes: nil)
    update!(
      completion_status: 'completed',
      completed_date: Time.current,
      grade: grade,
      points_earned: points_earned,
      notes: notes.presence || self.notes
    )
  end
end
```

### 4.4 Update Task Model

```ruby
class Task < ApplicationRecord
  belongs_to :teacher
  belongs_to :student, optional: true
  
  PRIORITIES = %w[low medium high].freeze
  STATUSES = %w[pending in_progress completed cancelled].freeze
  
  validates :title, presence: true, length: { maximum: 255 }
  validates :priority, inclusion: { in: PRIORITIES }
  validates :status, inclusion: { in: STATUSES }
  validates :category, length: { maximum: 100 }, allow_blank: true
  
  scope :for_student, ->(student_id) { 
    student_id.present? ? where(student_id: student_id) : where(student_id: nil) 
  }
  scope :by_status, ->(status) { where(status: status) if status.present? }
  scope :by_priority, ->(priority) { where(priority: priority) if priority.present? }
  scope :by_category, ->(category) { where(category: category) if category.present? }
  scope :due_between, ->(start_date, end_date) {
    where(due_date: start_date..end_date) if start_date.present? && end_date.present?
  }
  scope :incomplete, -> { where.not(status: %w[completed cancelled]) }
  scope :ordered_by_due_date, -> { order(Arel.sql('due_date ASC NULLS LAST')) }
  
  def student_name
    student&.display_name
  end
  
  def mark_complete!
    update!(
      status: 'completed',
      completed_date: Time.current
    )
  end
end
```

### 4.5 Create Assignments Controller

Create `app/controllers/api/v1/assignments_controller.rb`:

```ruby
module Api
  module V1
    class AssignmentsController < BaseController
      before_action :set_assignment, only: [:show, :update, :destroy, :complete]
      
      # GET /api/v1/assignments
      def index
        assignments = current_teacher.assignments
                                     .includes(:student, :subject, :calendar_event)
        
        # Apply filters
        assignments = assignments.for_student(params[:student_id])
        assignments = assignments.for_subject(params[:subject_id])
        assignments = assignments.by_status(params[:completion_status])
        
        if params[:due_date_from].present? || params[:due_date_to].present?
          from_date = params[:due_date_from].present? ? DateTime.parse(params[:due_date_from]) : nil
          to_date = params[:due_date_to].present? ? DateTime.parse(params[:due_date_to]) : nil
          assignments = assignments.due_between(from_date, to_date)
        end
        
        # Apply sorting
        sort_by = params[:sort_by] || 'due_date'
        sort_order = params[:sort_order] == 'desc' ? :desc : :asc
        
        assignments = case sort_by
                      when 'due_date'
                        assignments.order(Arel.sql("due_date #{sort_order} NULLS LAST"))
                      when 'assigned_date'
                        assignments.order(assigned_date: sort_order)
                      when 'title'
                        assignments.order(title: sort_order)
                      else
                        assignments.ordered_by_due_date
                      end
        
        result = paginate(assignments)
        
        render_success(
          result[:data].map { |a| assignment_response(a) },
          meta: result[:meta]
        )
      end
      
      # GET /api/v1/assignments/:id
      def show
        render_success(assignment_response(@assignment))
      end
      
      # POST /api/v1/assignments
      def create
        # Verify student belongs to teacher
        student = current_teacher.students.find_by(id: params[:student_id])
        unless student
          render_error("Student not found", code: "NOT_FOUND", status: :not_found)
          return
        end
        
        assignment = current_teacher.assignments.new(assignment_params)
        assignment.student = student
        
        if assignment.save
          render_created(assignment_response(assignment))
        else
          render_validation_errors(assignment)
        end
      end
      
      # PUT /api/v1/assignments/:id
      def update
        if @assignment.update(assignment_params)
          render_success(assignment_response(@assignment))
        else
          render_validation_errors(@assignment)
        end
      end
      
      # DELETE /api/v1/assignments/:id
      def destroy
        @assignment.destroy
        render_no_content
      end
      
      # PATCH /api/v1/assignments/:id/complete
      def complete
        @assignment.mark_complete!(
          grade: params[:grade],
          points_earned: params[:points_earned],
          notes: params[:notes]
        )
        
        render_success(assignment_response(@assignment))
      end
      
      private
      
      def set_assignment
        @assignment = current_teacher.assignments.find_by(id: params[:id])
        render_not_found("Assignment") unless @assignment
      end
      
      def assignment_params
        params.permit(
          :subject_id,
          :calendar_event_id,
          :title,
          :description,
          :due_date,
          :assigned_date,
          :completion_status,
          :completed_date,
          :grade,
          :points_earned,
          :points_possible,
          :notes
        )
      end
      
      def assignment_response(assignment)
        {
          id: assignment.id,
          student_id: assignment.student_id,
          student_name: assignment.student_name,
          teacher_id: assignment.teacher_id,
          subject_id: assignment.subject_id,
          subject_name: assignment.subject_name,
          calendar_event_id: assignment.calendar_event_id,
          title: assignment.title,
          description: assignment.description,
          due_date: assignment.due_date&.iso8601,
          assigned_date: assignment.assigned_date.iso8601,
          completion_status: assignment.completion_status,
          completed_date: assignment.completed_date&.iso8601,
          grade: assignment.grade,
          points_earned: assignment.points_earned&.to_f,
          points_possible: assignment.points_possible&.to_f,
          notes: assignment.notes,
          attachments: assignment.attachments,
          created_at: assignment.created_at.iso8601,
          updated_at: assignment.updated_at.iso8601
        }
      end
    end
  end
end
```

### 4.6 Create Tasks Controller

Create `app/controllers/api/v1/tasks_controller.rb`:

```ruby
module Api
  module V1
    class TasksController < BaseController
      before_action :set_task, only: [:show, :update, :destroy, :complete]
      
      # GET /api/v1/tasks
      def index
        tasks = current_teacher.tasks.includes(:student)
        
        # Apply filters
        if params[:student_id].present?
          if params[:student_id] == 'null'
            tasks = tasks.where(student_id: nil)
          else
            tasks = tasks.where(student_id: params[:student_id])
          end
        end
        
        tasks = tasks.by_status(params[:status])
        tasks = tasks.by_priority(params[:priority])
        tasks = tasks.by_category(params[:category])
        
        if params[:due_date_from].present? || params[:due_date_to].present?
          from_date = params[:due_date_from].present? ? DateTime.parse(params[:due_date_from]) : nil
          to_date = params[:due_date_to].present? ? DateTime.parse(params[:due_date_to]) : nil
          tasks = tasks.due_between(from_date, to_date)
        end
        
        # Apply sorting
        sort_by = params[:sort_by] || 'due_date'
        sort_order = params[:sort_order] == 'desc' ? :desc : :asc
        
        tasks = case sort_by
                when 'due_date'
                  tasks.order(Arel.sql("due_date #{sort_order} NULLS LAST"))
                when 'priority'
                  tasks.order(priority: sort_order)
                when 'title'
                  tasks.order(title: sort_order)
                else
                  tasks.ordered_by_due_date
                end
        
        result = paginate(tasks)
        
        render_success(
          result[:data].map { |t| task_response(t) },
          meta: result[:meta]
        )
      end
      
      # GET /api/v1/tasks/:id
      def show
        render_success(task_response(@task))
      end
      
      # POST /api/v1/tasks
      def create
        task = current_teacher.tasks.new(task_params)
        
        # Verify student belongs to teacher if provided
        if params[:student_id].present?
          student = current_teacher.students.find_by(id: params[:student_id])
          unless student
            render_error("Student not found", code: "NOT_FOUND", status: :not_found)
            return
          end
          task.student = student
        end
        
        if task.save
          render_created(task_response(task))
        else
          render_validation_errors(task)
        end
      end
      
      # PUT /api/v1/tasks/:id
      def update
        if @task.update(task_params)
          render_success(task_response(@task))
        else
          render_validation_errors(@task)
        end
      end
      
      # DELETE /api/v1/tasks/:id
      def destroy
        @task.destroy
        render_no_content
      end
      
      # PATCH /api/v1/tasks/:id/complete
      def complete
        @task.mark_complete!
        render_success(task_response(@task))
      end
      
      private
      
      def set_task
        @task = current_teacher.tasks.find_by(id: params[:id])
        render_not_found("Task") unless @task
      end
      
      def task_params
        params.permit(
          :title,
          :description,
          :due_date,
          :priority,
          :status,
          :category
        )
      end
      
      def task_response(task)
        {
          id: task.id,
          teacher_id: task.teacher_id,
          student_id: task.student_id,
          student_name: task.student_name,
          title: task.title,
          description: task.description,
          due_date: task.due_date&.iso8601,
          priority: task.priority,
          status: task.status,
          completed_date: task.completed_date&.iso8601,
          category: task.category,
          created_at: task.created_at.iso8601,
          updated_at: task.updated_at.iso8601
        }
      end
    end
  end
end
```

### 4.7 Update Routes for Assignments & Tasks

Add to `config/routes.rb` inside the `api/v1` namespace:

```ruby
resources :assignments, only: [:index, :show, :create, :update, :destroy] do
  member do
    patch :complete
  end
end

resources :tasks, only: [:index, :show, :create, :update, :destroy] do
  member do
    patch :complete
  end
end
```

### ✅ Phase 4 Checklist

- [ ] Created Assignment model and migration
- [ ] Created Task model and migration
- [ ] Updated models with validations and scopes
- [ ] Created AssignmentsController
- [ ] Created TasksController
- [ ] Updated routes
- [ ] Run migrations
- [ ] Test endpoints

---

## Phase 5: Report Cards

### 5.1 Create Report Card Models & Migrations

```bash
rails generate model ReportCard \
  student:references \
  teacher:references \
  title:string \
  period_type:string \
  start_date:date \
  end_date:date \
  grading_system:string \
  status:string \
  overall_comments:text \
  published_at:datetime
```

```bash
rails generate model ReportCardEntry \
  report_card:references \
  subject:references \
  subject_name:string \
  letter_grade:string \
  percentage_grade:decimal \
  standards_rating:string \
  comments:text
```

Update migrations and run:

```bash
rails db:migrate
```

### 5.2 Create Report Cards Controller

Create `app/controllers/api/v1/report_cards_controller.rb` (similar pattern to other controllers)

### 5.3 Update Routes

```ruby
resources :report_cards, only: [:index, :show, :create, :update, :destroy] do
  resources :entries, controller: 'report_card_entries', only: [:create, :update, :destroy]
  member do
    post :pdf
  end
end
```

---

## Phase 6: Expenses

### 6.1 Create Expense Models & Migrations

```bash
rails generate model ExpenseCategory \
  teacher:references \
  name:string \
  description:text \
  color_code:string \
  icon:string \
  is_system_default:boolean
```

```bash
rails generate model Expense \
  teacher:references \
  student:references \
  subject:references \
  expense_category:references \
  expense_date:date \
  amount:decimal \
  currency:string \
  vendor:string \
  description:text \
  receipt_url:string \
  payment_method:string \
  is_tax_deductible:boolean \
  notes:text \
  tags:jsonb
```

### 6.2 Create Expenses Controller with Summary

The expenses controller should include the summary endpoint for generating reports.

---

## Phase 7: Lesson Plans

### 7.1 Create Lesson Plan Models & Migrations

```bash
rails generate model LessonPlan \
  teacher:references \
  subject:references \
  title:string \
  description:text \
  content:text \
  grade_level:string \
  duration_minutes:integer \
  objectives:text \
  materials_needed:text \
  activities:jsonb \
  assessment_methods:text \
  notes:text \
  attachments:jsonb \
  is_template:boolean \
  visibility:string \
  view_count:integer
```

### 7.2 Create Lesson Plans Controller with Sharing

---

## Testing Checklist

### Unit Tests

- [ ] Teacher model validations
- [ ] Student model validations
- [ ] Assignment model validations
- [ ] JWT service encoding/decoding
- [ ] All model scopes

### Integration Tests

- [ ] Authentication flow (register → login → refresh → logout)
- [ ] CRUD operations for each resource
- [ ] Authorization (teacher can only access own data)
- [ ] Filtering and pagination

### API Tests (Manual with Postman/curl)

```bash
# Register
curl -X POST http://localhost:3000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"first_name":"Test","last_name":"Teacher","email":"test@example.com","password":"password123"}'

# Login
curl -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# Get students (with token)
curl -X GET http://localhost:3000/api/v1/students \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

---

## Deployment Preparation

### Pre-Deployment Checklist

- [ ] Set production JWT_SECRET_KEY
- [ ] Configure production database
- [ ] Set up Redis for Sidekiq
- [ ] Configure AWS S3 for file uploads
- [ ] Set up email service (SendGrid, Mailgun, etc.)
- [ ] Enable SSL/HTTPS
- [ ] Configure CORS for production domains
- [ ] Set up error tracking (Sentry, Rollbar)
- [ ] Set up logging
- [ ] Configure rate limiting
- [ ] Run security audit (brakeman)

### Heroku Deployment

```bash
# Create Heroku app
heroku create homeschool-api

# Add PostgreSQL
heroku addons:create heroku-postgresql:hobby-dev

# Add Redis
heroku addons:create heroku-redis:hobby-dev

# Set environment variables
heroku config:set JWT_SECRET_KEY=your-production-secret
heroku config:set RAILS_ENV=production

# Deploy
git push heroku main

# Run migrations
heroku run rails db:migrate

# Run seeds
heroku run rails db:seed
```

---

## Quick Reference: Complete Routes

```ruby
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      # Authentication
      namespace :auth do
        post 'register', to: 'authentication#register'
        post 'login', to: 'authentication#login'
        post 'refresh', to: 'authentication#refresh'
        post 'logout', to: 'authentication#logout'
        post 'password/reset-request', to: 'passwords#reset_request'
        post 'password/reset', to: 'passwords#reset'
        post 'password/change', to: 'passwords#change'
      end
      
      # Teacher profile
      get 'teachers/me', to: 'teachers#me'
      put 'teachers/me', to: 'teachers#update'
      delete 'teachers/me', to: 'teachers#destroy'
      
      # Students
      resources :students, only: [:index, :show, :create, :update, :destroy]
      
      # Subjects
      resources :subjects, only: [:index, :show, :create, :update, :destroy]
      
      # Calendar
      namespace :calendar do
        resources :events, controller: 'calendar_events', only: [:index, :show, :create, :update, :destroy] do
          member do
            patch 'attendance/:student_id', to: 'calendar_events#update_attendance'
          end
        end
        resources :event_types, only: [:index, :create]
      end
      
      # Assignments
      resources :assignments, only: [:index, :show, :create, :update, :destroy] do
        member do
          patch :complete
        end
      end
      
      # Tasks
      resources :tasks, only: [:index, :show, :create, :update, :destroy] do
        member do
          patch :complete
        end
      end
      
      # Report Cards
      resources :report_cards, only: [:index, :show, :create, :update, :destroy] do
        resources :entries, controller: 'report_card_entries', only: [:create, :update, :destroy]
        member do
          post :pdf
        end
      end
      
      # Expenses
      resources :expenses, only: [:index, :show, :create, :update, :destroy] do
        collection do
          get :summary
          get :categories
          post :categories, to: 'expense_categories#create'
        end
      end
      
      # Lesson Plans
      resources :lesson_plans, only: [:index, :show, :create, :update, :destroy] do
        collection do
          get :public
        end
        member do
          post :copy
          post :share
        end
      end
    end
  end
  
  get '/health', to: proc { [200, {}, ['OK']] }
end
```

---

## Document Change Log

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-11-30 | Initial implementation guide |

---

**You've got this! Start with Phase 0 and 1, then alternate with frontend work to stay motivated. 🚀**
