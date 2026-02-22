# Authentication API - Postman Testing Guide

## Setup

1. Start your Rails server:
   ```bash
   bin/rails server
   ```

2. In Postman, set these defaults for all requests:
   - Base URL: `http://localhost:3000`
   - Header: `Content-Type: application/json`

---

## Endpoints

### 1. Register a New Teacher

**Request:**
- Method: `POST`
- URL: `http://localhost:3000/api/v1/auth/register`
- Headers: `Content-Type: application/json`
- Body (raw JSON):
```json
{
  "first_name": "Robert",
  "last_name": "Masters",
  "email": "robert@example.com",
  "password": "password123"
}
```

**Expected Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "5d61b928-2cf3-47a0-b14c-10dccd76b7ec",
    "first_name": "Robert",
    "last_name": "Masters",
    "email": "robert@example.com",
    "created_at": "2026-02-17T12:00:00.000Z"
  }
}
```

---

### 2. Register with Existing Email (Error Case)

**Request:**
- Method: `POST`
- URL: `http://localhost:3000/api/v1/auth/register`
- Body (raw JSON):
```json
{
  "first_name": "Another",
  "last_name": "Person",
  "email": "robert@example.com",
  "password": "password123"
}
```

**Expected Response (422 Unprocessable Content):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "email": ["has already been taken"]
    }
  }
}
```

---

### 3. Register with Missing Fields (Error Case)

**Request:**
- Method: `POST`
- URL: `http://localhost:3000/api/v1/auth/register`
- Body (raw JSON):
```json
{
  "email": "test@example.com",
  "password": "password123"
}
```

**Expected Response (422 Unprocessable Content):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "first_name": ["can't be blank"],
      "last_name": ["can't be blank"]
    }
  }
}
```

---

### 4. Register with Short Password (Error Case)

**Request:**
- Method: `POST`
- URL: `http://localhost:3000/api/v1/auth/register`
- Body (raw JSON):
```json
{
  "first_name": "Test",
  "last_name": "User",
  "email": "test@example.com",
  "password": "short"
}
```

**Expected Response (422 Unprocessable Content):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "password": ["is too short (minimum is 8 characters)"]
    }
  }
}
```

---

### 5. Login with Valid Credentials

**Request:**
- Method: `POST`
- URL: `http://localhost:3000/api/v1/auth/login`
- Headers: `Content-Type: application/json`
- Body (raw JSON):
```json
{
  "email": "robert@example.com",
  "password": "password123"
}
```

**Expected Response (200 OK):**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9.eyJ0ZWFjaGVyX2lkIjoiNWQ2MWI5MjgtMmNmMy00N2EwLWIxNGMtMTBkY2NkNzZiN2VjIiwiZXhwIjoxNzcwMTA2NDY3LCJpYXQiOjE3NzAxMDI4Njd9.r9ln0BDePF7q95Cl5FuvDErPuTvnJf9iDh-BtWAO1CU",
  "refresh_token": "eyJhbGciOiJIUzI1NiJ9.eyJ0ZWFjaGVyX2lkIjoiNWQ2MWI5MjgtMmNmMy00N2EwLWIxNGMtMTBkY2NkNzZiN2VjIiwidHlwZSI6InJlZnJlc2giLCJleHAiOjE3NzI2OTQ4NjcsImlhdCI6MTc3MDEwMjg2NywianRpIjoiZjk1MDA2MDktMWNkOC00MDZjLThlYWMtZDhlMzQxNGUxZWVjIn0.FuPiluBQ8ejjqo8bsA2nuXezvnzpoTnwdos1p4DutSQ"
}
```

**Note:** Save the `access_token` for authenticated requests and the `refresh_token` for refreshing the access token.

---

### 6. Login with Wrong Password (Error Case)

**Request:**
- Method: `POST`
- URL: `http://localhost:3000/api/v1/auth/login`
- Body (raw JSON):
```json
{
  "email": "robert@example.com",
  "password": "wrongpassword"
}
```

**Expected Response (401 Unauthorized):**
```json
{
  "success": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid email or password"
  }
}
```

---

### 7. Login with Non-Existent Email (Error Case)

**Request:**
- Method: `POST`
- URL: `http://localhost:3000/api/v1/auth/login`
- Body (raw JSON):
```json
{
  "email": "doesnotexist@example.com",
  "password": "password123"
}
```

**Expected Response (401 Unauthorized):**
```json
{
  "success": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid email or password"
  }
}
```

---

## Testing Checklist

| Test Case | Expected Status | ✓ |
|-----------|-----------------|---|
| Register new teacher | 201 Created | |
| Register with existing email | 422 Unprocessable Content | |
| Register with missing fields | 422 Unprocessable Content | |
| Register with short password | 422 Unprocessable Content | |
| Login with valid credentials | 200 OK | |
| Login with wrong password | 401 Unauthorized | |
| Login with non-existent email | 401 Unauthorized | |

---

## Tips

- Use Postman's **Environment Variables** to store `access_token` and `refresh_token` after login
- Set up a **Collection** for all auth endpoints to easily re-run tests
- Use **Tests** tab in Postman to automate response validation