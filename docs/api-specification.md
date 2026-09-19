# Homeschool Management App - API Specification

**Version:** 1.0  
**Date:** November 14, 2025  
**Status:** Initial Design  
**Base URL:** `https://api.myhomeschoolmaster.com/api/v1`

---

## Table of Contents
1. [Overview](#overview)
2. [Authentication](#authentication)
3. [API Conventions](#api-conventions)
4. [Error Handling](#error-handling)
5. [Rate Limiting](#rate-limiting)
6. [API Endpoints](#api-endpoints)
   - [Authentication Endpoints](#authentication-endpoints)
   - [Teacher Endpoints](#teacher-endpoints)
   - [Student Endpoints](#student-endpoints)
   - [Calendar Endpoints](#calendar-endpoints)
   - [Assignment Endpoints](#assignment-endpoints)
   - [Task Endpoints](#task-endpoints)
   - [Report Card Endpoints](#report-card-endpoints)
   - [Expense Endpoints](#expense-endpoints)
   - [Subject Endpoints](#subject-endpoints)
   - [Lesson Plan Endpoints](#lesson-plan-endpoints)
7. [Appendix](#appendix)

---

## Overview

### Purpose
This document defines the RESTful API for the Homeschool Management mobile application. The API enables teachers (homeschool parents) to manage students, calendar events, assignments, tasks, report cards, expenses, and lesson plans.

### Design Principles
- **RESTful Architecture** - Standard HTTP methods (GET, POST, PUT, DELETE)
- **JSON Format** - All requests and responses use JSON
- **Stateless** - Each request contains all necessary information
- **Versioned** - API version in URL path (`/v1`)
- **Secure** - HTTPS only, JWT authentication
- **Consistent** - Predictable naming and response structures

### Technical Stack
- **Protocol:** HTTPS only
- **Format:** JSON (application/json)
- **Authentication:** JWT (JSON Web Tokens)
- **Date/Time Format:** ISO 8601 (UTC)
- **Character Encoding:** UTF-8

---

## Authentication

### Authentication Method: JWT (JSON Web Tokens)

The API uses JWT-based authentication with two token types:

1. **Access Token** - Short-lived (1 hour), used for API requests
2. **Refresh Token** - Long-lived (30 days), used to obtain new access tokens

### Token Storage
- **Mobile Apps:** Store in secure platform storage (iOS Keychain, Android KeyStore)
- **Never** store tokens in local storage or unencrypted files
- **Web:** the API sets httponly `access_token` and `refresh_token` cookies,
  `SameSite=Lax` and host only on `api.myhomeschoolmaster.com`. The web app is
  served from `www.myhomeschoolmaster.com`, the same registrable domain, so
  these are first party cookies. See "Domains and auth cookies" in the
  [README](./README.md#domains-and-auth-cookies) before changing either host.

### Authentication Flow

```
1. User registers or logs in
2. Server returns access_token and refresh_token
3. Client stores both tokens securely
4. Client includes access_token in Authorization header for all requests
5. When access_token expires, use refresh_token to get new access_token
6. If refresh_token expires, user must log in again
```

### Including Authentication in Requests

All protected endpoints require the `Authorization` header:

```http
Authorization: Bearer {access_token}
```

**Example:**
```http
GET /api/v1/students
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Token Expiration
- **Access Token:** 1 hour (3600 seconds)
- **Refresh Token:** 30 days (2592000 seconds)

### Security Requirements
- All API requests must use HTTPS
- Passwords must be hashed using bcrypt or argon2
- Tokens should include user ID and role in payload
- Rate limiting on authentication endpoints

---

## API Conventions

### HTTP Methods

| Method | Purpose | Idempotent |
|--------|---------|------------|
| GET | Retrieve resources | Yes |
| POST | Create new resources | No |
| PUT | Update/replace entire resource | Yes |
| PATCH | Partially update resource | No |
| DELETE | Remove resources | Yes |

### Request Headers

**Required for all requests:**
```http
Content-Type: application/json
Accept: application/json
```

**Required for protected endpoints:**
```http
Authorization: Bearer {access_token}
```

### Response Format

All API responses follow this structure:

#### Success Response
```json
{
  "success": true,
  "data": {
    // Response data here
  },
  "meta": {
    // Optional metadata (pagination, counts, etc.)
  }
}
```

#### Error Response
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      // Optional additional error details
    }
  }
}
```

### Pagination

For endpoints that return lists, use query parameters:

```http
GET /api/v1/students?page=1&limit=20
```

**Pagination Parameters:**
- `page` - Page number (default: 1)
- `limit` - Items per page (default: 20, max: 100)

**Paginated Response:**
```json
{
  "success": true,
  "data": [...],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "total_pages": 8,
    "has_next": true,
    "has_prev": false
  }
}
```

### Filtering & Sorting

**Filtering:**
```http
GET /api/v1/assignments?student_id=123&status=incomplete
```

**Sorting:**
```http
GET /api/v1/expenses?sort_by=expense_date&sort_order=desc
```

### Date/Time Format

All dates and times use ISO 8601 format in UTC:

- **Date:** `2025-11-14`
- **DateTime:** `2025-11-14T15:30:00Z`
- **DateTime with timezone:** `2025-11-14T15:30:00-05:00`

---

## Error Handling

### HTTP Status Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful GET, PUT, PATCH, DELETE |
| 201 | Created | Successful POST (resource created) |
| 204 | No Content | Successful DELETE with no response body |
| 400 | Bad Request | Invalid request format or parameters |
| 401 | Unauthorized | Missing or invalid authentication token |
| 403 | Forbidden | Valid token but insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Request conflicts with existing data |
| 422 | Unprocessable Entity | Validation errors |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error |
| 503 | Service Unavailable | Temporary server unavailability |

### Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed for one or more fields",
    "details": {
      "email": ["Email is required", "Email format is invalid"],
      "password": ["Password must be at least 8 characters"]
    }
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid token |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `VALIDATION_ERROR` | 422 | Input validation failed |
| `DUPLICATE_EMAIL` | 409 | Email already registered |
| `INVALID_CREDENTIALS` | 401 | Wrong email or password |
| `TOKEN_EXPIRED` | 401 | Access token has expired |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Server error |

---

## Rate Limiting

### Limits by Endpoint Type

| Endpoint Type | Limit | Window |
|--------------|-------|--------|
| Authentication (login, register) | 5 requests | per 15 minutes |
| Password reset | 3 requests | per hour |
| Standard API requests | 1000 requests | per hour |
| File uploads | 50 requests | per hour |

### Rate Limit Headers

Responses include rate limit information:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1699564800
```

### Rate Limit Exceeded Response

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later.",
    "details": {
      "retry_after": 900
    }
  }
}
```

---

## API Endpoints

---

## Authentication Endpoints

### 1. Register New Teacher

Create a new teacher account.

**Endpoint:** `POST /auth/register`

**Authentication:** None (public endpoint)

**Request Body:**
```json
{
  "first_name": "Sarah",
  "last_name": "Johnson",
  "email": "sarah.johnson@email.com",
  "password": "SecurePassword123!",
  "phone": "555-123-4567",
  "newsletter_subscribed": true
}
```

**Validation Rules:**
- `first_name`: Required, 1-100 characters
- `last_name`: Required, 1-100 characters
- `email`: Required, valid email format, unique
- `password`: Required, minimum 8 characters, must include uppercase, lowercase, and number
- `phone`: Optional, valid phone format
- `newsletter_subscribed`: Optional, boolean (default: false)

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "teacher": {
      "id": "uuid-123",
      "first_name": "Sarah",
      "last_name": "Johnson",
      "email": "sarah.johnson@email.com",
      "phone": "555-123-4567",
      "newsletter_subscribed": true,
      "created_at": "2025-11-14T15:30:00Z"
    },
    "tokens": {
      "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "expires_in": 3600,
      "token_type": "Bearer"
    }
  }
}
```

**Error Responses:**

400 Bad Request - Invalid input
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "email": ["Email format is invalid"],
      "password": ["Password must be at least 8 characters"]
    }
  }
}
```

409 Conflict - Email already exists
```json
{
  "success": false,
  "error": {
    "code": "DUPLICATE_EMAIL",
    "message": "An account with this email already exists"
  }
}
```

---

### 2. Login

Authenticate and receive access tokens.

**Endpoint:** `POST /auth/login`

**Authentication:** None (public endpoint)

**Request Body:**
```json
{
  "email": "sarah.johnson@email.com",
  "password": "SecurePassword123!"
}
```

**Validation Rules:**
- `email`: Required, valid email format
- `password`: Required

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "teacher": {
      "id": "uuid-123",
      "first_name": "Sarah",
      "last_name": "Johnson",
      "email": "sarah.johnson@email.com",
      "phone": "555-123-4567",
      "newsletter_subscribed": true
    },
    "tokens": {
      "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "expires_in": 3600,
      "token_type": "Bearer"
    }
  }
}
```

**Error Response (401 Unauthorized):**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Invalid email or password"
  }
}
```

---

### 3. Refresh Access Token

Get a new access token using refresh token.

**Endpoint:** `POST /auth/refresh`

**Authentication:** None (uses refresh token)

**Request Body:**
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_in": 3600,
    "token_type": "Bearer"
  }
}
```

**Error Response (401 Unauthorized):**
```json
{
  "success": false,
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "Refresh token has expired. Please log in again."
  }
}
```

---

### 4. Logout

Invalidate current tokens.

**Endpoint:** `POST /auth/logout`

**Authentication:** Required (Bearer token)

**Request Body:**
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "message": "Successfully logged out"
  }
}
```

---

### 5. Request Password Reset

Request password reset email.

**Endpoint:** `POST /auth/password/reset-request`

**Authentication:** None (public endpoint)

**Request Body:**
```json
{
  "email": "sarah.johnson@email.com"
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "message": "If an account exists with this email, a password reset link has been sent."
  }
}
```

**Note:** Always return success even if email doesn't exist (security best practice)

---

### 6. Reset Password

Reset password using token from email.

**Endpoint:** `POST /auth/password/reset`

**Authentication:** None (uses reset token)

**Request Body:**
```json
{
  "reset_token": "token-from-email",
  "new_password": "NewSecurePassword123!"
}
```

**Validation Rules:**
- `reset_token`: Required, valid reset token
- `new_password`: Required, minimum 8 characters, must include uppercase, lowercase, and number

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "message": "Password successfully reset. Please log in with your new password."
  }
}
```

**Error Response (400 Bad Request):**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_TOKEN",
    "message": "Reset token is invalid or has expired"
  }
}
```

---

### 7. Change Password

Change password for authenticated user.

**Endpoint:** `POST /auth/password/change`

**Authentication:** Required (Bearer token)

**Request Body:**
```json
{
  "current_password": "SecurePassword123!",
  "new_password": "NewSecurePassword456!"
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "message": "Password successfully changed"
  }
}
```

**Error Response (401 Unauthorized):**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Current password is incorrect"
  }
}
```

---

### 8. Verify Email

Verify email address using token from verification email.

**Endpoint:** `POST /auth/email/verify`

**Authentication:** None (uses verification token)

**Request Body:**
```json
{
  "verification_token": "token-from-email"
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "message": "Email successfully verified"
  }
}
```

---

### 9. Resend Verification Email

Request a new verification email.

**Endpoint:** `POST /auth/email/resend-verification`

**Authentication:** Required (Bearer token)

**Request Body:** None

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "message": "Verification email sent"
  }
}
```

---

## Teacher Endpoints

### 1. Get Current Teacher Profile

Get authenticated teacher's profile.

**Endpoint:** `GET /teachers/me`

**Authentication:** Required

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-123",
    "first_name": "Sarah",
    "last_name": "Johnson",
    "nickname": null,
    "email": "sarah.johnson@email.com",
    "phone": "555-123-4567",
    "newsletter_subscribed": true,
    "profile_image_url": "https://storage.example.com/profiles/uuid-123.jpg",
    "created_at": "2025-01-15T10:00:00Z",
    "updated_at": "2025-11-14T15:30:00Z"
  }
}
```

---

### 2. Update Teacher Profile

Update authenticated teacher's profile. Identity fields only: this endpoint
gates on `current_password`.

`time_zone` is **not** accepted here. It moved to
`PATCH /api/v1/profile/preferences`, which has no password gate. A request
sending `time_zone` to this endpoint does not update it.

**Endpoint:** `PUT /teachers/me`

**Authentication:** Required

**Request Body:**
```json
{
  "first_name": "Sarah",
  "last_name": "Johnson",
  "nickname": "Ms. J",
  "phone": "555-123-4567",
  "newsletter_subscribed": false
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-123",
    "first_name": "Sarah",
    "last_name": "Johnson",
    "nickname": "Ms. J",
    "email": "sarah.johnson@email.com",
    "phone": "555-123-4567",
    "newsletter_subscribed": false,
    "updated_at": "2025-11-14T16:00:00Z"
  }
}
```

---

### 3. Update Teacher Preferences

Update the authenticated teacher's non sensitive settings. These are display
preferences rather than credentials, so this endpoint does **not** require
`current_password`: the client can persist a detected timezone without being
able to prompt for a password.

`time_zone` moved here off `PATCH /profile`. Sending `time_zone` to
`PATCH /profile` no longer updates it: that field is silently ignored there,
and identity fields sent alongside it still apply as normal.

**Endpoint:** `PATCH /api/v1/profile/preferences`

**Authentication:** Required (session cookie)

**Request Body:**
```json
{
  "time_zone": "Europe/Lisbon"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| time_zone | string | No | IANA identifier, for example `America/New_York`. Validated when present. Rails style names such as "Eastern Time (US & Canada)" are rejected. |

Future display and notification settings belong on this endpoint, not on the
password gated profile endpoint.

**Success Response (200 OK):**

Returns the full teacher payload. Note the two timezone fields: `time_zone` is
the raw column and is `null` until the teacher sets one, while
`effective_time_zone` applies the `America/New_York` fallback. The client uses
the null in `time_zone` to decide whether to prompt on first login.

```json
{
  "success": true,
  "data": {
    "id": "uuid-123",
    "first_name": "Sarah",
    "last_name": "Johnson",
    "email": "sarah.johnson@email.com",
    "time_zone": "Europe/Lisbon",
    "effective_time_zone": "Europe/Lisbon",
    "created_at": "2025-01-15T10:00:00Z"
  }
}
```

For a teacher who has never set a zone:
```json
{
  "success": true,
  "data": {
    "time_zone": null,
    "effective_time_zone": "America/New_York"
  }
}
```

**Error Response (422 Unprocessable Content):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "time_zone": ["is not a recognized IANA time zone"]
    }
  }
}
```

**Error Response (401 Unauthorized):** returned when no valid session cookie is
present.

---

### 4. Upload Profile Image

Upload or update teacher profile image.

**Endpoint:** `POST /teachers/me/profile-image`

**Authentication:** Required

**Content-Type:** `multipart/form-data`

**Request Body:**
```
file: [image file] (JPEG, PNG, max 5MB)
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "profile_image_url": "https://storage.example.com/profiles/uuid-123.jpg",
    "uploaded_at": "2025-11-14T16:00:00Z"
  }
}
```

---

### 5. Delete Teacher Account

Permanently delete teacher account and all associated data.

**Endpoint:** `DELETE /teachers/me`

**Authentication:** Required

**Request Body:**
```json
{
  "password": "SecurePassword123!",
  "confirmation": "DELETE"
}
```

**Success Response (204 No Content)**

---

## Student Endpoints

### 1. Get All Students

Get all students for authenticated teacher.

**Endpoint:** `GET /students`

**Authentication:** Required

**Query Parameters:**
- `is_active` (optional): Filter by active status (true/false)
- `grade_level` (optional): Filter by grade level
- `sort_by` (optional): Field to sort by (default: first_name)
- `sort_order` (optional): asc or desc (default: asc)

**Example Request:**
```http
GET /students?is_active=true&sort_by=first_name&sort_order=asc
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-456",
      "teacher_id": "uuid-123",
      "first_name": "Emma",
      "last_name": "Johnson",
      "nickname": "Em",
      "date_of_birth": "2015-03-15",
      "grade_level": "5th Grade",
      "profile_image_url": "https://storage.example.com/students/uuid-456.jpg",
      "notes": "Advanced in math, loves science",
      "is_active": true,
      "created_at": "2025-01-15T10:00:00Z",
      "updated_at": "2025-11-14T15:30:00Z"
    },
    {
      "id": "uuid-789",
      "teacher_id": "uuid-123",
      "first_name": "Jack",
      "last_name": "Johnson",
      "nickname": null,
      "date_of_birth": "2018-07-22",
      "grade_level": "2nd Grade",
      "profile_image_url": null,
      "notes": null,
      "is_active": true,
      "created_at": "2025-01-15T10:05:00Z",
      "updated_at": "2025-11-14T15:30:00Z"
    }
  ],
  "meta": {
    "total": 2
  }
}
```

---

### 2. Get Single Student

Get details for a specific student.

**Endpoint:** `GET /students/{student_id}`

**Authentication:** Required

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-456",
    "teacher_id": "uuid-123",
    "first_name": "Emma",
    "last_name": "Johnson",
    "nickname": "Em",
    "date_of_birth": "2015-03-15",
    "grade_level": "5th Grade",
    "profile_image_url": "https://storage.example.com/students/uuid-456.jpg",
    "notes": "Advanced in math, loves science",
    "is_active": true,
    "created_at": "2025-01-15T10:00:00Z",
    "updated_at": "2025-11-14T15:30:00Z"
  }
}
```

**Error Response (404 Not Found):**
```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Student not found"
  }
}
```

---

### 3. Create Student

Create a new student.

**Endpoint:** `POST /students`

**Authentication:** Required

**Request Body:**
```json
{
  "first_name": "Emma",
  "last_name": "Johnson",
  "nickname": "Em",
  "date_of_birth": "2015-03-15",
  "grade_level": "5th Grade",
  "notes": "Advanced in math, loves science"
}
```

**Validation Rules:**
- `first_name`: Required, 1-100 characters
- `last_name`: Required, 1-100 characters
- `nickname`: Optional, max 100 characters
- `date_of_birth`: Optional, valid date (YYYY-MM-DD)
- `grade_level`: Optional, max 20 characters
- `notes`: Optional, text

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-456",
    "teacher_id": "uuid-123",
    "first_name": "Emma",
    "last_name": "Johnson",
    "nickname": "Em",
    "date_of_birth": "2015-03-15",
    "grade_level": "5th Grade",
    "profile_image_url": null,
    "notes": "Advanced in math, loves science",
    "is_active": true,
    "created_at": "2025-11-14T16:00:00Z",
    "updated_at": "2025-11-14T16:00:00Z"
  }
}
```

---

### 4. Update Student

Update student information.

**Endpoint:** `PUT /students/{student_id}`

**Authentication:** Required

**Request Body:**
```json
{
  "first_name": "Emma",
  "last_name": "Johnson",
  "nickname": "Emma J",
  "date_of_birth": "2015-03-15",
  "grade_level": "6th Grade",
  "notes": "Advanced in math and science"
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-456",
    "teacher_id": "uuid-123",
    "first_name": "Emma",
    "last_name": "Johnson",
    "nickname": "Emma J",
    "date_of_birth": "2015-03-15",
    "grade_level": "6th Grade",
    "profile_image_url": "https://storage.example.com/students/uuid-456.jpg",
    "notes": "Advanced in math and science",
    "is_active": true,
    "updated_at": "2025-11-14T16:30:00Z"
  }
}
```

---

### 5. Delete Student

Soft delete or permanently delete a student.

**Endpoint:** `DELETE /students/{student_id}`

**Authentication:** Required

**Query Parameters:**
- `permanent` (optional): true for hard delete, false for soft delete (default: false)

**Success Response (204 No Content)**

---

### 6. Upload Student Profile Image

Upload or update student profile image.

**Endpoint:** `POST /students/{student_id}/profile-image`

**Authentication:** Required

**Content-Type:** `multipart/form-data`

**Request Body:**
```
file: [image file] (JPEG, PNG, max 5MB)
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "profile_image_url": "https://storage.example.com/students/uuid-456.jpg",
    "uploaded_at": "2025-11-14T16:00:00Z"
  }
}
```

---

## Calendar Endpoints

### 1. Get Calendar Events

Get calendar events for authenticated teacher.

**Endpoint:** `GET /calendar/events`

**Authentication:** Required

**Query Parameters:**
- `start_date` (required): Start of date range (ISO 8601)
- `end_date` (required): End of date range (ISO 8601)
- `student_id` (optional): Filter by student
- `event_type_id` (optional): Filter by event type
- `include_recurring` (optional): Include recurring events (default: true)
- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 50)

**How the date range is interpreted:**

A bare date such as `2026-09-17` names one of the **teacher's local days**, not
a UTC day. `start_date` widens to local midnight and `end_date` to local
23:59:59 in the teacher's `time_zone`, falling back to `America/New_York` when
they have not set one, and both convert to UTC for the query. So a single day
query for the 17th returns the events that teacher sees on the 17th, including
an 8:00 PM Eastern event whose stored UTC timestamp is on the 18th.

A value that already carries a time component is used **as sent**, with no
widening. An explicit offset is honored rather than reinterpreted in the
teacher's zone, so `2026-09-17T00:00:00-04:00` means that instant. A time
without an offset is read as UTC.

Range matching uses overlap semantics: an event is returned when any part of it
falls inside the window, so a genuine multi day event appears on every day it
spans. Stored timestamps are always UTC and are never rewritten by this query.

**Example Request:**
```http
GET /calendar/events?start_date=2025-11-01&end_date=2025-11-30&student_id=uuid-456
```

A single local day, for a teacher in `America/New_York`:
```http
GET /calendar/events?start_date=2026-09-17&end_date=2026-09-17
```
resolves to the UTC window `2026-09-17T04:00:00Z` through
`2026-09-18T03:59:59Z`. The same request from a teacher in `Asia/Tokyo`
resolves to `2026-09-16T15:00:00Z` through `2026-09-17T14:59:59Z`.

**Error Response (422 Unprocessable Content):** returned when `start_date` or
`end_date` is missing or cannot be parsed.
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "start_date and end_date are required and must be valid dates"
  }
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-event-1",
      "teacher_id": "uuid-123",
      "event_type_id": "uuid-type-1",
      "event_type_name": "Lesson",
      "title": "Math - Fractions",
      "description": "Introduction to fractions with visual aids",
      "location": "Home classroom",
      "start_time": "2025-11-15T10:00:00Z",
      "end_time": "2025-11-15T11:00:00Z",
      "all_day": false,
      "is_recurring": true,
      "recurrence_rule": "FREQ=WEEKLY;BYDAY=MO,WE,FR",
      "recurrence_end_date": "2025-12-20",
      "parent_event_id": null,
      "color_code": "#4CAF50",
      "reminder_minutes": 30,
      "attendee_ids": ["uuid-456"],
      "created_at": "2025-11-01T10:00:00Z",
      "updated_at": "2025-11-01T10:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 50,
    "total": 1
  }
}
```

---

> **Attendees are ids, not records.** A calendar response carries
> `attendee_ids` only. A parent blocking subjects across several children can
> reach hundreds of events in a month, and nesting the full student record on
> every event repeats the same handful of students hundreds of times. Clients
> already load the students list separately and resolve names and colours from
> it. Note that the student index returns active students only, so an event can
> reference a student who has since been soft deleted: clients should render
> those as a former student rather than dropping them.

### 2. Get Single Calendar Event

Get details for a specific event.

**Endpoint:** `GET /calendar/events/{event_id}`

**Authentication:** Required

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-event-1",
    "teacher_id": "uuid-123",
    "event_type_id": "uuid-type-1",
    "event_type_name": "Lesson",
    "title": "Math - Fractions",
    "description": "Introduction to fractions with visual aids",
    "location": "Home classroom",
    "start_time": "2025-11-15T10:00:00Z",
    "end_time": "2025-11-15T11:00:00Z",
    "all_day": false,
    "is_recurring": true,
    "recurrence_rule": "FREQ=WEEKLY;BYDAY=MO,WE,FR",
    "recurrence_end_date": "2025-12-20",
    "parent_event_id": null,
    "color_code": "#4CAF50",
    "reminder_minutes": 30,
    "attendee_ids": ["uuid-456"],
    "created_at": "2025-11-01T10:00:00Z",
    "updated_at": "2025-11-01T10:00:00Z"
  }
}
```

---

### 3. Create Calendar Event

Create a new calendar event.

**Endpoint:** `POST /calendar/events`

**Authentication:** Required

**Request Body:**
```json
{
  "event_type_id": "uuid-type-1",
  "title": "Math - Fractions",
  "description": "Introduction to fractions with visual aids",
  "location": "Home classroom",
  "start_time": "2025-11-15T10:00:00Z",
  "end_time": "2025-11-15T11:00:00Z",
  "all_day": false,
  "is_recurring": true,
  "recurrence_rule": "FREQ=WEEKLY;BYDAY=MO,WE,FR",
  "recurrence_end_date": "2025-12-20",
  "color_code": "#4CAF50",
  "reminder_minutes": 30,
  "student_ids": ["uuid-456", "uuid-789"]
}
```

**Validation Rules:**
- `title`: Required, max 255 characters
- `start_time`: Required, valid ISO 8601 datetime
- `end_time`: Optional, must be after start_time
- `all_day`: Optional, boolean (default: false)
- `is_recurring`: Optional, boolean (default: false)
- `recurrence_rule`: Required if is_recurring=true, valid RRULE format
- `student_ids`: Optional, array of valid student IDs

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-event-1",
    "teacher_id": "uuid-123",
    "event_type_id": "uuid-type-1",
    "title": "Math - Fractions",
    "description": "Introduction to fractions with visual aids",
    "location": "Home classroom",
    "start_time": "2025-11-15T10:00:00Z",
    "end_time": "2025-11-15T11:00:00Z",
    "all_day": false,
    "is_recurring": true,
    "recurrence_rule": "FREQ=WEEKLY;BYDAY=MO,WE,FR",
    "recurrence_end_date": "2025-12-20",
    "color_code": "#4CAF50",
    "reminder_minutes": 30,
    "attendee_ids": ["uuid-456"],
    "created_at": "2025-11-14T16:00:00Z"
  }
}
```

---

### 4. Update Calendar Event

Update an existing calendar event.

**Endpoint:** `PUT /calendar/events/{event_id}`

**Authentication:** Required

**Request Body:**
```json
{
  "title": "Math - Advanced Fractions",
  "description": "Continuing fractions lesson",
  "start_time": "2025-11-15T10:30:00Z",
  "end_time": "2025-11-15T11:30:00Z",
  "reminder_minutes": 15,
  "student_ids": ["uuid-456"]
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-event-1",
    "title": "Math - Advanced Fractions",
    "updated_at": "2025-11-14T16:30:00Z"
  }
}
```

---

### 5. Delete Calendar Event

Delete a calendar event (with option to delete recurring series).

**Endpoint:** `DELETE /calendar/events/{event_id}`

**Authentication:** Required

**Query Parameters:**
- `delete_series` (optional): Delete entire recurring series (default: false)

**Success Response (204 No Content)**

---

### 6. Update Event Attendance

Update attendance status for a student in an event.

**Endpoint:** `PATCH /calendar/events/{event_id}/attendance/{student_id}`

**Authentication:** Required

**Request Body:**
```json
{
  "attendance_status": "attended"
}
```

**Validation Rules:**
- `attendance_status`: Required, one of: pending, attended, absent, excused

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "event_id": "uuid-event-1",
    "student_id": "uuid-456",
    "attendance_status": "attended",
    "updated_at": "2025-11-15T11:30:00Z"
  }
}
```

---

### 7. Get Event Types

Get all event types available.

**Endpoint:** `GET /calendar/event-types`

**Authentication:** Required

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-type-1",
      "name": "Lesson",
      "color_code": "#4CAF50",
      "icon": "book",
      "description": "Regular lesson or teaching session",
      "is_system_default": true
    },
    {
      "id": "uuid-type-2",
      "name": "Field Trip",
      "color_code": "#2196F3",
      "icon": "bus",
      "description": "Educational field trip or outing",
      "is_system_default": true
    }
  ]
}
```

---

### 8. Create Custom Event Type

Create a custom event type.

**Endpoint:** `POST /calendar/event-types`

**Authentication:** Required

**Request Body:**
```json
{
  "name": "Music Practice",
  "color_code": "#9C27B0",
  "icon": "music",
  "description": "Daily music practice session"
}
```

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-type-custom-1",
    "name": "Music Practice",
    "color_code": "#9C27B0",
    "icon": "music",
    "description": "Daily music practice session",
    "is_system_default": false
  }
}
```

---

## Assignment Endpoints

Built. An assignment is a piece of work in a subject, given to one or more
students, with its own due date. It does not create a calendar event and does
not touch `calendar_events`.

**Not built, and deliberately so:** there is no `student_id` on the assignment,
no `calendar_event_id`, no `completion_status`, no `assigned_date`, no
`attachments`, and no `grade` or `points_earned` column on the assignment
itself. Earlier versions of this section and of the database architecture doc
described all of those. They cannot all be right: an assignment given to three
students cannot carry one score, which is the whole point of grading per
student.

**How the two tables relate.** One row in `assignment_grades` per student per
assignment. The row existing means the student has the work. `points_earned`
null means it has not been marked yet. There is no separate join table: the
grade row is the assignment of the work and the score for it.

**How a score is stored.** Points earned against points possible, both
decimals. `points_possible` lives on the assignment, because it is a property
of the work; `points_earned` lives on the grade, because it is per student.
Percentages and letters are derived on read and never stored: a percentage
cannot be turned back into points, and a letter cannot be turned back into
either, so storing those would throw away what the teacher entered. A family
that grades out of 100 sets `points_possible` to 100 and enters the
percentage; one that grades pass or fail sets it to 1 and enters 1 or 0.

---

### 1. Get All Assignments

**Endpoint:** `GET /assignments`

**Authentication:** Required

**Query Parameters:**
- `subject_id` (optional): only that subject's work
- `due_from`, `due_to` (optional): `YYYY-MM-DD`, inclusive at both ends

An unparseable date is a 422 rather than a filter dropped quietly.

**Ordering:** due date ascending, undated last, then creation order.

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-assignment-1",
      "teacher_id": "uuid-123",
      "subject_id": "uuid-subject-math",
      "title": "Chapter 4 problems",
      "description": "Odd numbered questions only",
      "due_date": "2026-09-20",
      "points_possible": "20.0",
      "weight": "1.0",
      "grades": [
        {
          "id": "uuid-grade-1",
          "assignment_id": "uuid-assignment-1",
          "student_id": "uuid-student-eliza",
          "points_earned": "18.0",
          "percentage": "90.0",
          "graded": true,
          "graded_at": "2026-09-21T14:02:00Z",
          "notes": null
        },
        {
          "id": "uuid-grade-2",
          "assignment_id": "uuid-assignment-1",
          "student_id": "uuid-student-samuel",
          "points_earned": null,
          "percentage": null,
          "graded": false,
          "graded_at": null,
          "notes": null
        }
      ],
      "created_at": "2026-09-18T10:00:00Z"
    }
  ]
}
```

Grades are nested rather than reduced to ids, unlike calendar event attendees.
There the ids kept a month of several hundred events small; here the grades are
the substance of the record and a family has a handful of students.

---

### 2. Get Single Assignment

**Endpoint:** `GET /assignments/{assignment_id}`

**Authentication:** Required

Same payload as a row of the index. 404 for another teacher's assignment.

---

### 3. Create Assignment

**Endpoint:** `POST /assignments`

**Authentication:** Required

**Request Body:**
```json
{
  "subject_id": "uuid-subject-math",
  "title": "Chapter 4 problems",
  "description": "Odd numbered questions only",
  "due_date": "2026-09-20",
  "points_possible": 20,
  "weight": 1,
  "student_ids": ["uuid-student-eliza", "uuid-student-samuel"]
}
```

**Accepted fields:** `subject_id`, `title`, `description`, `due_date`,
`points_possible`, `weight`, `student_ids`. `teacher_id` is ignored.

**Validation Rules:**
- `subject_id`: required, must be one of the teacher's own subjects
- `title`: required, max 255 characters
- `points_possible`: required, greater than zero. Defaults to 100
- `weight`: required, zero or greater. Defaults to 1
- `due_date`: optional date. Work with no due date appears in no report period
- `student_ids`: optional. Every id must be one of the teacher's own students,
  and a list containing one that is not rejects the whole request and creates
  nothing

Each id in `student_ids` becomes an unmarked grade row.

---

### 4. Update Assignment

**Endpoint:** `PATCH /assignments/{assignment_id}`

**Authentication:** Required

Same fields as create. A submitted `student_ids` array replaces the assigned
set: students already on the assignment keep the score recorded against them,
and **a student removed from the list has their grade row deleted, score and
all**. Unassigning is destructive by design, the alternative being an orphan
score for work the student is no longer doing.

Omitting `student_ids` leaves the assigned set alone. Note that an empty array
only survives a JSON body: form encoding drops it, so a form encoded empty
array reads as "no change" rather than "remove everyone".

---

### 5. Delete Assignment

**Endpoint:** `DELETE /assignments/{assignment_id}`

**Authentication:** Required

A hard delete, taking its grade rows with it.

**Success Response (204 No Content)**

---

### 6. Get Grades for an Assignment

**Endpoint:** `GET /assignments/{assignment_id}/grades`

**Authentication:** Required

The same grade objects nested in the assignment payload.

---

### 7. Record a Score

**Endpoint:** `PATCH /assignments/{assignment_id}/grades/{grade_id}`

**Authentication:** Required

**Request Body:**
```json
{ "points_earned": 18, "notes": "Neat working" }
```

**Accepted fields:** `points_earned`, `notes`. Which students have the
assignment is set through the assignment's `student_ids`, so there is one way
to do each thing: this endpoint only records marks.

- `points_earned` null puts the work back to unmarked and clears `graded_at`,
  which takes it out of every average again
- `points_earned` 0 is a real score of zero and does count
- A score above `points_possible` is accepted: that is extra credit, and
  capping it silently would lose marks
- `graded_at` is set by the server when a score is first entered and is not
  moved by a later correction

---

### 8. Student Progress, the weighted roll up

**Endpoint:** `GET /students/{student_id}/progress?from=&to=`

**Authentication:** Required

The auto-calculated weighted grades the marketing copy promises, per subject
over a period.

**Query Parameters:** `from` and `to`, both **required**, `YYYY-MM-DD`,
inclusive. A roll up with no period is not a report, and defaulting to all time
would quietly answer a different question. `to` before `from` is a 422.

**The calculation:**

```
percentage = sum(weight * points_earned / points_possible) / sum(weight)
```

A weighted mean of each assignment's own percentage, not of raw points. The two
differ: adding points up means a 100 point exam already outweighs a 10 point
quiz ten to one whether the teacher meant it to or not. Weighting percentages
separates what the work is out of from how much it counts.

**What is included:**
- Assignments with a `due_date` inside the period. Undated work belongs to no
  period and is excluded
- Only grade rows belonging to this student
- Only marked work. An assignment the student has but that is unmarked is
  excluded rather than scored zero, so a report part way through a term
  reflects what has actually been marked. `assigned_count`, `graded_count` and
  `ungraded_count` are all reported so the reader can see what the figure covers
- Work weighted 0 contributes to neither side of the sum

**No grade rather than zero.** When nothing is marked, or every weight in a
subject is 0, `percentage` and `letter` come back null. Dividing either case
would invent a grade of 0% for a student who simply has not been marked.

**Letters:** 90/80/70/60 on the weighted percentage, derived on read. Per
teacher scales are not built.

**Overall** applies the same formula across every subject at once rather than
averaging the subject averages: there is no "how much does this subject count"
anywhere in the schema, and inventing one here would be a second weighting
concept the teacher never set.

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "student_id": "uuid-student-eliza",
    "from": "2026-09-01",
    "to": "2026-09-30",
    "subjects": [
      {
        "subject_id": "uuid-subject-math",
        "subject_name": "Math",
        "assigned_count": 4,
        "graded_count": 3,
        "ungraded_count": 1,
        "points_earned": "98.0",
        "points_possible": "130.0",
        "percentage": "75.0",
        "letter": "C"
      }
    ],
    "overall": {
      "assigned_count": 5,
      "graded_count": 4,
      "ungraded_count": 1,
      "points_earned": "193.0",
      "points_possible": "230.0",
      "percentage": "79.0",
      "letter": "C"
    }
  }
}
```

---

## Task Endpoints

Built. A task is the teacher's own to-do item: "Submit internet
reimbursement", "Email co-op leader for winter schedule". It carries a title,
an optional description, an optional due date and a completed state, and it
belongs to the teacher alone. Every query is scoped to the authenticated
teacher: another teacher's task is not found rather than forbidden.

**Not built, and deliberately so:** tasks have no `student_id`, `priority`,
`status` or `category`. An earlier version of this section described all four,
along with pagination, six query parameters and a `student_name` field. None of
it shipped. Tasks are not attached to students, subjects, assignments or
calendar events.

**How completion is stored:** one nullable `completed_at` timestamp. Null means
not done. The payload carries both `completed`, the boolean a checkbox binds
to, and `completed_at`, the instant it was ticked. There is one column behind
the two fields, so they cannot disagree, and completing an already completed
task does not move the timestamp.

---

### 1. Get All Tasks

**Endpoint:** `GET /tasks`

**Authentication:** Required

**Query Parameters:**
- `completed` (optional): `true` or `false`. Omitted returns everything
- `due_by` (optional): `YYYY-MM-DD`. Tasks due on or before that date,
  inclusive. Undated tasks are excluded, since they are not due by any date

Both are optional and combine. `completed=false` is the dashboard panel's
query, `completed=false&due_by=<today>` is the overdue question, and no
parameters at all is the full list. There is no separate overdue endpoint and
no pagination.

A value that cannot be honored is a 422 rather than a filter dropped quietly:
a panel silently showing every task would look like it was working.

**Ordering:** due date ascending with undated tasks last, then creation order
for ties. A task with no due date is not due soon, so it sits at the end of a
list the dashboard reads from the top.

**Example Request:**
```http
GET /tasks?completed=false&due_by=2026-09-20
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-task-1",
      "teacher_id": "uuid-123",
      "title": "Export report cards",
      "description": null,
      "due_date": "2026-09-10",
      "completed": false,
      "completed_at": null,
      "created_at": "2026-09-18T10:00:00Z"
    },
    {
      "id": "uuid-task-2",
      "teacher_id": "uuid-123",
      "title": "Email co-op leader for winter schedule",
      "description": "Ask about the January start date",
      "due_date": "2026-09-20",
      "completed": false,
      "completed_at": null,
      "created_at": "2026-09-18T10:05:00Z"
    }
  ]
}
```

**Error Response (422 Unprocessable Content):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "completed": ["must be true or false"]
    }
  }
}
```

---

### 2. Get Single Task

**Endpoint:** `GET /tasks/{task_id}`

**Authentication:** Required

**Success Response (200 OK):** the task payload, as in Get All Tasks.

**Error Response (404 Not Found):**
```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Task not found"
  }
}
```

---

### 3. Create Task

**Endpoint:** `POST /tasks`

**Authentication:** Required

**Request Body:**
```json
{
  "title": "Submit internet reimbursement",
  "description": "Attach the September bill",
  "due_date": "2026-09-30"
}
```

**Accepted fields:** `title`, `description`, `due_date`, `completed`. Anything
else is ignored, including `teacher_id` and `completed_at`: the task is always
created against the authenticated teacher, and the timestamp is only ever set
through `completed`.

**Validation Rules:**
- `title`: required, max 255 characters. Duplicates are allowed, unlike subjects
- `description`: optional text. A blank string is stored as null
- `due_date`: optional date, `YYYY-MM-DD`. A blank string clears it
- `completed`: optional boolean, defaults to false

**Success Response (201 Created):** the created task payload.

---

### 4. Update Task

**Endpoint:** `PATCH /tasks/{task_id}`

**Authentication:** Required

Same accepted fields as create. Ticking and unticking the checkbox are both
plain updates:

```json
{ "completed": true }
```

```json
{ "completed": false }
```

There is no `/tasks/{id}/complete` member route. One endpoint handles both
directions, which a checkbox needs, and `completed: true` on an already
completed task leaves the original `completed_at` where it is.

**Success Response (200 OK):** the full task payload.

---

### 5. Delete Task

**Endpoint:** `DELETE /tasks/{task_id}`

**Authentication:** Required

A hard delete, unlike students and subjects: the row is removed. Nothing
references a task, and a to-do the teacher deleted is meant to be gone rather
than hidden.

**Success Response (204 No Content)**

---

## Report Card Endpoints

**Not built.** The section that stood here described `period_type`,
`grading_system`, a draft/finalized/published status, denormalized
`subject_name`, and separate `letter_grade`, `percentage_grade` and
`standards_rating` columns, none of which exists.

What does exist is the live calculation behind them: see **Student Progress**
under Assignment Endpoints, which returns the per subject weighted roll up over
any date range. A persisted report card is a snapshot of that plus a title,
notes and an issued timestamp, and it is the next slice of this work rather
than something already shipped.

---

## Expense Endpoints

### 1. Get All Expenses

Get expenses for authenticated teacher.

**Endpoint:** `GET /expenses`

**Authentication:** Required

**Query Parameters:**
- `student_id` (optional): Filter by student (null for general expenses)
- `subject_id` (optional): Filter by subject
- `category_id` (optional): Filter by category
- `expense_date_from` (optional): Expenses after this date
- `expense_date_to` (optional): Expenses before this date
- `is_tax_deductible` (optional): Filter by tax deductible status
- `tags` (optional): Filter by tags (comma-separated)
- `sort_by` (optional): Field to sort by (default: expense_date)
- `sort_order` (optional): asc or desc (default: desc)
- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 20)

**Example Request:**
```http
GET /expenses?student_id=uuid-456&expense_date_from=2025-01-01&expense_date_to=2025-01-31
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-expense-1",
      "teacher_id": "uuid-123",
      "student_id": "uuid-456",
      "student_name": "Emma Johnson",
      "subject_id": "uuid-subject-1",
      "subject_name": "Mathematics",
      "category_id": "uuid-cat-1",
      "category_name": "Curriculum",
      "expense_date": "2025-01-15",
      "amount": 45.99,
      "currency": "USD",
      "vendor": "Amazon",
      "description": "Singapore Math Workbook Level 5",
      "receipt_url": "https://storage.example.com/receipts/receipt-123.pdf",
      "payment_method": "Credit Card",
      "is_tax_deductible": true,
      "notes": "Annual curriculum purchase",
      "tags": ["curriculum", "math", "annual"],
      "created_at": "2025-01-15T20:00:00Z",
      "updated_at": "2025-01-15T20:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 1,
    "total_amount": 45.99
  }
}
```

---

### 2. Get Single Expense

Get details for a specific expense.

**Endpoint:** `GET /expenses/{expense_id}`

**Authentication:** Required

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-expense-1",
    "teacher_id": "uuid-123",
    "student_id": "uuid-456",
    "student_name": "Emma Johnson",
    "subject_id": "uuid-subject-1",
    "subject_name": "Mathematics",
    "category_id": "uuid-cat-1",
    "category_name": "Curriculum",
    "expense_date": "2025-01-15",
    "amount": 45.99,
    "currency": "USD",
    "vendor": "Amazon",
    "description": "Singapore Math Workbook Level 5",
    "receipt_url": "https://storage.example.com/receipts/receipt-123.pdf",
    "payment_method": "Credit Card",
    "is_tax_deductible": true,
    "notes": "Annual curriculum purchase",
    "tags": ["curriculum", "math", "annual"],
    "created_at": "2025-01-15T20:00:00Z",
    "updated_at": "2025-01-15T20:00:00Z"
  }
}
```

---

### 3. Create Expense

Create a new expense.

**Endpoint:** `POST /expenses`

**Authentication:** Required

**Request Body:**
```json
{
  "student_id": "uuid-456",
  "subject_id": "uuid-subject-1",
  "category_id": "uuid-cat-1",
  "expense_date": "2025-01-15",
  "amount": 45.99,
  "vendor": "Amazon",
  "description": "Singapore Math Workbook Level 5",
  "payment_method": "Credit Card",
  "is_tax_deductible": true,
  "notes": "Annual curriculum purchase",
  "tags": ["curriculum", "math", "annual"]
}
```

**Validation Rules:**
- `expense_date`: Required, valid date (YYYY-MM-DD)
- `amount`: Required, positive decimal
- `description`: Required, text
- `student_id`: Optional, null for general expenses
- `subject_id`: Optional, valid subject ID
- `category_id`: Optional, valid category ID
- `vendor`: Optional, max 255 characters
- `currency`: Optional, 3-letter code (default: USD)
- `is_tax_deductible`: Optional, boolean (default: true)
- `tags`: Optional, array of strings

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-expense-1",
    "teacher_id": "uuid-123",
    "student_id": "uuid-456",
    "expense_date": "2025-01-15",
    "amount": 45.99,
    "description": "Singapore Math Workbook Level 5",
    "created_at": "2025-11-14T16:00:00Z"
  }
}
```

---

### 4. Update Expense

Update an existing expense.

**Endpoint:** `PUT /expenses/{expense_id}`

**Authentication:** Required

**Request Body:**
```json
{
  "amount": 49.99,
  "description": "Singapore Math Workbook Level 5 + Answer Key",
  "notes": "Added answer key to order"
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-expense-1",
    "amount": 49.99,
    "description": "Singapore Math Workbook Level 5 + Answer Key",
    "notes": "Added answer key to order",
    "updated_at": "2025-11-14T16:30:00Z"
  }
}
```

---

### 5. Delete Expense

Delete an expense.

**Endpoint:** `DELETE /expenses/{expense_id}`

**Authentication:** Required

**Success Response (204 No Content)**

---

### 6. Upload Expense Receipt

Upload a receipt image/document for an expense.

**Endpoint:** `POST /expenses/{expense_id}/receipt`

**Authentication:** Required

**Content-Type:** `multipart/form-data`

**Request Body:**
```
file: [file] (PDF, JPEG, PNG, max 10MB)
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "receipt_url": "https://storage.example.com/receipts/receipt-123.pdf",
    "uploaded_at": "2025-11-14T16:00:00Z"
  }
}
```

---

### 7. Get Expense Summary Report

Generate expense summary for a date range with various filters.

**Endpoint:** `GET /expenses/summary`

**Authentication:** Required

**Query Parameters:**
- `student_id` (optional): Filter by student
- `subject_id` (optional): Filter by subject
- `category_id` (optional): Filter by category
- `expense_date_from` (required): Start date
- `expense_date_to` (required): End date
- `group_by` (optional): Group results by (student, subject, category, month)

**Example Request:**
```http
GET /expenses/summary?expense_date_from=2025-01-01&expense_date_to=2025-12-31&group_by=student
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "summary": {
      "total_amount": 2450.75,
      "total_expenses": 47,
      "date_range": {
        "from": "2025-01-01",
        "to": "2025-12-31"
      },
      "grouped_by": "student",
      "groups": [
        {
          "student_id": "uuid-456",
          "student_name": "Emma Johnson",
          "total_amount": 1285.50,
          "expense_count": 25,
          "by_category": [
            {
              "category_name": "Curriculum",
              "total_amount": 675.00,
              "expense_count": 8
            },
            {
              "category_name": "Supplies",
              "total_amount": 410.50,
              "expense_count": 12
            }
          ]
        },
        {
          "student_id": "uuid-789",
          "student_name": "Jack Johnson",
          "total_amount": 865.25,
          "expense_count": 18
        },
        {
          "student_id": null,
          "student_name": "General Expenses",
          "total_amount": 300.00,
          "expense_count": 4
        }
      ]
    }
  }
}
```

---

### 8. Get Expense Categories

Get all expense categories.

**Endpoint:** `GET /expenses/categories`

**Authentication:** Required

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-cat-1",
      "teacher_id": "uuid-123",
      "name": "Curriculum",
      "description": "Textbooks, workbooks, and curriculum materials",
      "color_code": "#4CAF50",
      "icon": "book",
      "is_system_default": true
    },
    {
      "id": "uuid-cat-2",
      "teacher_id": "uuid-123",
      "name": "Field Trips",
      "description": "Educational outings and experiences",
      "color_code": "#2196F3",
      "icon": "bus",
      "is_system_default": true
    }
  ]
}
```

---

### 9. Create Expense Category

Create a custom expense category.

**Endpoint:** `POST /expenses/categories`

**Authentication:** Required

**Request Body:**
```json
{
  "name": "Art Supplies",
  "description": "Paints, brushes, canvas, and art materials",
  "color_code": "#9C27B0",
  "icon": "palette"
}
```

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-cat-custom-1",
    "teacher_id": "uuid-123",
    "name": "Art Supplies",
    "description": "Paints, brushes, canvas, and art materials",
    "color_code": "#9C27B0",
    "icon": "palette",
    "is_system_default": false
  }
}
```

---

## Subject Endpoints

Built. Subjects belong to a teacher and carry a name, an optional colour and
description, and an active flag. Every query is scoped to the authenticated
teacher: another teacher's subject is not found rather than forbidden.

**Name uniqueness:** a name is unique per teacher and compared case
insensitively, so one teacher cannot hold both "Math" and "math". Two different
teachers may each have a subject called Math. A removed subject releases its
name, so the same name can be used again afterwards. The rule is enforced by a
model validation and by a matching partial unique index on
`(teacher_id, lower(name)) WHERE is_active`.

---

### 1. Get All Subjects

Get the authenticated teacher's active subjects, ordered by name.

**Endpoint:** `GET /subjects`

**Authentication:** Required

Removed subjects are excluded. There are no query parameters: filtering and
sorting are not implemented for this collection.

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-subject-1",
      "teacher_id": "uuid-123",
      "name": "Mathematics",
      "color": "#d97b0a",
      "description": "Arithmetic, algebra, geometry and problem solving",
      "is_active": true,
      "created_at": "2026-09-18T10:00:00Z"
    },
    {
      "id": "uuid-subject-2",
      "teacher_id": "uuid-123",
      "name": "Science",
      "color": "#16a34a",
      "description": null,
      "is_active": true,
      "created_at": "2026-09-18T10:05:00Z"
    }
  ]
}
```

---

### 2. Get Single Subject

**Endpoint:** `GET /subjects/{subject_id}`

**Authentication:** Required

A removed subject is still readable by id, which matches students: only the
collection hides them.

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-subject-1",
    "teacher_id": "uuid-123",
    "name": "Mathematics",
    "color": "#d97b0a",
    "description": "Arithmetic, algebra, geometry and problem solving",
    "is_active": true,
    "created_at": "2026-09-18T10:00:00Z"
  }
}
```

**Error Response (404 Not Found):**
```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Subject not found"
  }
}
```

---

### 3. Create Subject

**Endpoint:** `POST /subjects`

**Authentication:** Required

**Request Body:**
```json
{
  "name": "Mathematics",
  "color": "#d97b0a",
  "description": "Arithmetic, algebra, geometry and problem solving"
}
```

**Accepted fields:** `name`, `color`, `description`. Anything else is ignored,
including `teacher_id` and `is_active`: the subject is always created against
the authenticated teacher and always active.

**Validation Rules:**
- `name`: required, max 100 characters, unique per teacher, case insensitive
- `color`: optional, max 20 characters. Not format checked, matching students
- `description`: optional text. A blank string is stored as null

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-subject-1",
    "teacher_id": "uuid-123",
    "name": "Mathematics",
    "color": "#d97b0a",
    "description": "Arithmetic, algebra, geometry and problem solving",
    "is_active": true,
    "created_at": "2026-09-18T16:00:00Z"
  }
}
```

**Error Response (422 Unprocessable Content):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "name": ["has already been taken"]
    }
  }
}
```

---

### 4. Update Subject

**Endpoint:** `PATCH /subjects/{subject_id}`

**Authentication:** Required

Same accepted fields as create. `is_active` is not among them, so this endpoint
cannot be used to restore a removed subject.

**Request Body:**
```json
{
  "name": "Advanced Mathematics",
  "color": "#7c3aed"
}
```

**Success Response (200 OK):** the full subject payload, as in Get Single
Subject.

**Error Response (422 Unprocessable Content):** same shape as create, including
a rename onto a name the teacher already uses.

---

### 5. Delete Subject

**Endpoint:** `DELETE /subjects/{subject_id}`

**Authentication:** Required

Soft delete: `is_active` is flipped to false and the row survives, matching
students. The subject leaves the index, stays readable by id, and releases its
name for reuse.

**Success Response (204 No Content)**

---

## Lesson Plan Endpoints

### 1. Get My Lesson Plans

Get lesson plans created by authenticated teacher.

**Endpoint:** `GET /lesson-plans`

**Authentication:** Required

**Query Parameters:**
- `subject_id` (optional): Filter by subject
- `grade_level` (optional): Filter by grade level
- `is_template` (optional): Filter templates only
- `visibility` (optional): Filter by visibility (private, public)
- `sort_by` (optional): Field to sort by (default: created_at)
- `sort_order` (optional): asc or desc (default: desc)
- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 20)

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-lp-1",
      "teacher_id": "uuid-123",
      "subject_id": "uuid-subject-1",
      "subject_name": "Mathematics",
      "title": "Introduction to Fractions",
      "description": "Visual introduction to fraction concepts using pizza slices",
      "grade_level": "3rd-4th Grade",
      "duration_minutes": 45,
      "objectives": "Students will understand fraction notation and identify halves, thirds, and fourths",
      "materials_needed": "Paper plates, markers, scissors",
      "is_template": false,
      "visibility": "private",
      "view_count": 0,
      "created_at": "2025-11-01T10:00:00Z",
      "updated_at": "2025-11-14T15:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

---

### 2. Get Single Lesson Plan

Get details for a specific lesson plan.

**Endpoint:** `GET /lesson-plans/{lesson_plan_id}`

**Authentication:** Required

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-lp-1",
    "teacher_id": "uuid-123",
    "teacher_name": "Sarah Johnson",
    "subject_id": "uuid-subject-1",
    "subject_name": "Mathematics",
    "title": "Introduction to Fractions",
    "description": "Visual introduction to fraction concepts using pizza slices",
    "content": "Full lesson plan content here...",
    "grade_level": "3rd-4th Grade",
    "duration_minutes": 45,
    "objectives": "Students will understand fraction notation and identify halves, thirds, and fourths",
    "materials_needed": "Paper plates, markers, scissors",
    "activities": [
      {
        "name": "Pizza Fraction Activity",
        "duration": 20,
        "description": "Students cut paper plates to create fraction representations"
      }
    ],
    "assessment_methods": "Observe student understanding during activities, check worksheet completion",
    "notes": "Works well with hands-on learners",
    "attachments": [
      {
        "name": "fraction-worksheet.pdf",
        "url": "https://storage.example.com/lesson-plans/worksheet.pdf",
        "type": "pdf"
      }
    ],
    "is_template": false,
    "visibility": "private",
    "view_count": 0,
    "created_at": "2025-11-01T10:00:00Z",
    "updated_at": "2025-11-14T15:00:00Z"
  }
}
```

---

### 3. Create Lesson Plan

Create a new lesson plan.

**Endpoint:** `POST /lesson-plans`

**Authentication:** Required

**Request Body:**
```json
{
  "subject_id": "uuid-subject-1",
  "title": "Introduction to Fractions",
  "description": "Visual introduction to fraction concepts using pizza slices",
  "content": "Full lesson plan content here...",
  "grade_level": "3rd-4th Grade",
  "duration_minutes": 45,
  "objectives": "Students will understand fraction notation and identify halves, thirds, and fourths",
  "materials_needed": "Paper plates, markers, scissors",
  "activities": [
    {
      "name": "Pizza Fraction Activity",
      "duration": 20,
      "description": "Students cut paper plates to create fraction representations"
    }
  ],
  "assessment_methods": "Observe student understanding during activities",
  "notes": "Works well with hands-on learners",
  "is_template": false,
  "visibility": "private"
}
```

**Validation Rules:**
- `title`: Required, max 255 characters
- `subject_id`: Optional, valid subject ID
- `description`: Optional, text
- `content`: Optional, text
- `grade_level`: Optional, max 50 characters
- `duration_minutes`: Optional, positive integer
- `visibility`: Optional, one of: private, public (default: private)

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-lp-1",
    "teacher_id": "uuid-123",
    "title": "Introduction to Fractions",
    "visibility": "private",
    "created_at": "2025-11-14T16:00:00Z"
  }
}
```

---

### 4. Update Lesson Plan

Update an existing lesson plan.

**Endpoint:** `PUT /lesson-plans/{lesson_plan_id}`

**Authentication:** Required

**Request Body:**
```json
{
  "title": "Introduction to Fractions (Updated)",
  "description": "Enhanced visual introduction to fraction concepts",
  "visibility": "public"
}
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-lp-1",
    "title": "Introduction to Fractions (Updated)",
    "visibility": "public",
    "updated_at": "2025-11-14T16:30:00Z"
  }
}
```

---

### 5. Delete Lesson Plan

Delete a lesson plan.

**Endpoint:** `DELETE /lesson-plans/{lesson_plan_id}`

**Authentication:** Required

**Success Response (204 No Content)**

---

### 6. Search Public Lesson Plans

Search publicly shared lesson plans from other teachers.

**Endpoint:** `GET /lesson-plans/public`

**Authentication:** Required

**Query Parameters:**
- `search` (optional): Search in title and description
- `subject_id` (optional): Filter by subject
- `grade_level` (optional): Filter by grade level
- `sort_by` (optional): Field to sort by (default: view_count)
- `sort_order` (optional): asc or desc (default: desc)
- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 20)

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-lp-public-1",
      "teacher_id": "uuid-other-teacher",
      "teacher_name": "Jennifer Smith",
      "subject_id": "uuid-subject-1",
      "subject_name": "Mathematics",
      "title": "Fun with Fractions",
      "description": "Engaging fraction lesson using everyday objects",
      "grade_level": "3rd-5th Grade",
      "duration_minutes": 60,
      "view_count": 145,
      "created_at": "2025-10-15T10:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

---

### 7. Copy Public Lesson Plan

Copy a public lesson plan to your own library.

**Endpoint:** `POST /lesson-plans/{lesson_plan_id}/copy`

**Authentication:** Required

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-lp-copy-1",
    "teacher_id": "uuid-123",
    "title": "Fun with Fractions (Copy)",
    "visibility": "private",
    "created_at": "2025-11-14T16:00:00Z"
  }
}
```

---

### 8. Share Lesson Plan with Specific Teacher

Share a lesson plan privately with another teacher (future feature - V2.0).

**Endpoint:** `POST /lesson-plans/{lesson_plan_id}/share`

**Authentication:** Required

**Request Body:**
```json
{
  "shared_with_teacher_email": "friend@email.com",
  "can_edit": false
}
```

**Success Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "share_id": "uuid-share-1",
    "lesson_plan_id": "uuid-lp-1",
    "shared_with_teacher_email": "friend@email.com",
    "can_edit": false,
    "shared_date": "2025-11-14T16:00:00Z"
  }
}
```

---

### 9. Upload Lesson Plan Attachment

Upload a file attachment to a lesson plan.

**Endpoint:** `POST /lesson-plans/{lesson_plan_id}/attachments`

**Authentication:** Required

**Content-Type:** `multipart/form-data`

**Request Body:**
```
file: [file] (PDF, DOCX, images, max 10MB)
```

**Success Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "attachment": {
      "name": "worksheet.pdf",
      "url": "https://storage.example.com/lesson-plans/worksheet.pdf",
      "type": "pdf",
      "size": 245678,
      "uploaded_at": "2025-11-14T16:00:00Z"
    }
  }
}
```

---

## Appendix

### A. Common Response Examples

#### Successful Creation
```json
{
  "success": true,
  "data": {
    "id": "uuid-123",
    "created_at": "2025-11-14T16:00:00Z"
  }
}
```

#### Validation Error
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "email": ["Email is required"],
      "password": ["Password must be at least 8 characters"]
    }
  }
}
```

#### Not Found Error
```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Resource not found"
  }
}
```

#### Unauthorized Error
```json
{
  "success": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Authentication required"
  }
}
```

---

### B. Date/Time Examples

**ISO 8601 Format:**
```
2025-11-14T15:30:00Z           # UTC time
2025-11-14T15:30:00-05:00      # EST time
2025-11-14                     # Date only
```

---

### C. RRULE Examples for Recurring Events

**Weekly on Mondays, Wednesdays, Fridays:**
```
FREQ=WEEKLY;BYDAY=MO,WE,FR
```

**Daily for 30 occurrences:**
```
FREQ=DAILY;COUNT=30
```

**Monthly on the 15th:**
```
FREQ=MONTHLY;BYMONTHDAY=15
```

**Every other week:**
```
FREQ=WEEKLY;INTERVAL=2
```

---

### D. Grading System Examples

**Letter Grade:**
```json
{
  "letter_grade": "A",
  "percentage_grade": null,
  "standards_rating": null
}
```

**Percentage:**
```json
{
  "letter_grade": null,
  "percentage_grade": 92.5,
  "standards_rating": null
}
```

**Standards-Based:**
```json
{
  "letter_grade": null,
  "percentage_grade": null,
  "standards_rating": "Exceeds Expectations"
}
```

---

### E. JWT Token Payload Example

```json
{
  "sub": "uuid-123",
  "email": "sarah.johnson@email.com",
  "role": "teacher",
  "iat": 1699564800,
  "exp": 1699568400
}
```

**Payload Fields:**
- `sub`: Subject (teacher ID)
- `email`: Teacher email
- `role`: User role (always "teacher" in MVP)
- `iat`: Issued at timestamp
- `exp`: Expiration timestamp

---

### F. File Upload Constraints

| File Type | Max Size | Allowed Extensions |
|-----------|----------|-------------------|
| Profile Images | 5 MB | .jpg, .jpeg, .png, .gif |
| Receipts | 10 MB | .pdf, .jpg, .jpeg, .png |
| Attachments | 10 MB | .pdf, .docx, .doc, .jpg, .jpeg, .png |

---

### G. Security Best Practices

1. **Always use HTTPS** - Never send credentials over HTTP
2. **Store tokens securely** - Use iOS Keychain or Android KeyStore
3. **Implement token refresh** - Don't ask users to log in repeatedly
4. **Validate all inputs** - Never trust client-side data
5. **Hash passwords** - Use bcrypt or argon2 with proper salt
6. **Rate limit authentication** - Prevent brute force attacks
7. **Implement CSRF protection** - For web-based interfaces
8. **Log security events** - Track failed login attempts
9. **Expire sessions** - Force re-authentication after inactivity
10. **Sanitize file uploads** - Validate file types and scan for malware

---

### H. Pagination Example

**Request:**
```http
GET /students?page=2&limit=10
```

**Response:**
```json
{
  "success": true,
  "data": [...],
  "meta": {
    "page": 2,
    "limit": 10,
    "total": 45,
    "total_pages": 5,
    "has_next": true,
    "has_prev": true
  }
}
```

---

## Document Change Log

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-11-14 | Initial API specification document |

---

## Approval & Sign-off

**Prepared by:** API Architecture Team  
**Review Date:** 2025-11-14  
**Status:** Draft - Pending Approval

---

**Next Steps:**
1. Review and approve API specification
2. Set up development environment
3. Implement authentication endpoints first
4. Build core CRUD endpoints
5. Implement file upload functionality
6. Add search and filtering capabilities
7. Implement expense reporting
8. Build lesson plan sharing features
9. Create comprehensive API tests
10. Document deployment procedures
