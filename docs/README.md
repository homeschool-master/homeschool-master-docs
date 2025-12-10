# Homeschool Management App - Documentation

Welcome to the complete documentation for the Homeschool Management SaaS application.

## Table of Contents

- [Overview](#overview)
- [Architecture Documentation](#architecture-documentation)
- [Getting Started](#getting-started)
- [Development Guides](#development-guides)
- [API Reference](#api-reference)
- [Deployment](#deployment)

---

## Overview

A comprehensive homeschool management platform built with Rails API backend and React Native mobile frontend.

**Tech Stack:**
- **Backend:** Ruby on Rails 7 (API mode), PostgreSQL, Redis, Sidekiq
- **Mobile:** React Native, Expo, TypeScript, React Query, Zustand
- **Authentication:** JWT with refresh tokens
- **File Storage:** AWS S3
- **Deployment:** Heroku (MVP) → AWS (Production)

**Key Features:**
- Student management
- Calendar & scheduling with recurring events
- Assignment tracking
- Task management
- Report cards with multiple grading systems
- Expense tracking & reporting
- Lesson plan sharing

---

## Architecture Documentation

### [Database Architecture](./database-architecture.md)
Complete database schema with 15 tables, relationships, indexes, and constraints.

**Contents:**
- Entity Relationship Diagram
- Table definitions (Teachers, Students, Calendar, Assignments, Tasks, Report Cards, Expenses, Lesson Plans)
- Relationships and foreign keys
- Indexes for performance
- Sample queries

### [API Specification](./api-specification.md)
RESTful API documentation with 80+ endpoints across 9 modules.

**Contents:**
- Authentication endpoints (register, login, refresh, password reset)
- CRUD operations for all resources
- Request/response formats
- Error handling
- Rate limiting
- Pagination standards
- JWT authentication flow

### [Mobile App Architecture](./mobile-architecture.md)
React Native frontend architecture and patterns.

**Contents:**
- Project structure
- Technology stack (React Native, Expo, TypeScript)
- State management (React Query + Zustand)
- Navigation (React Navigation 6)
- Component architecture
- Offline support strategy
- Security implementation
- Testing strategy
- Future web migration path

### [Rails Implementation Guide](./rails-implementation-guide.md)
Step-by-step instructions for building the Rails API backend.

**Contents:**
- Phase 0: Project setup
- Phase 1: Authentication (JWT, refresh tokens)
- Phase 2: Teachers & Students
- Phase 3: Subjects & Calendar
- Phase 4: Assignments & Tasks
- Phase 5: Report Cards
- Phase 6: Expenses
- Phase 7: Lesson Plans
- Testing checklist
- Deployment preparation

---

## Getting Started

### Prerequisites

**Backend Development:**
```bash
Ruby 3.2.2
Rails 7.1+
PostgreSQL 14+
Redis 6+
```

**Mobile Development:**
```bash
Node.js 18+ LTS
npm or yarn
Expo CLI
Xcode (for iOS, macOS only)
Android Studio (for Android)
```

### Quick Start - Backend

```bash
# Clone the API repository
git clone https://github.com/YOUR-ORG/homeschool-api.git
cd homeschool-api

# Install dependencies
bundle install

# Setup database
rails db:create db:migrate db:seed

# Start server
rails server
```

API will be available at `http://localhost:3000`

### Quick Start - Mobile

```bash
# Clone the mobile repository
git clone https://github.com/YOUR-ORG/homeschool-mobile.git
cd homeschool-mobile

# Install dependencies
npm install

# Start Expo
npx expo start

# Run on iOS
npx expo run:ios

# Run on Android
npx expo run:android
```

---

## Development Guides

### Backend Development

**Project Structure:**
```
homeschool-api/
├── app/
│   ├── controllers/api/v1/     # API controllers
│   ├── models/                 # ActiveRecord models
│   ├── services/               # Business logic (JWT, etc.)
│   └── serializers/            # JSON serialization
├── config/
│   ├── routes.rb              # API routes
│   └── initializers/          # CORS, etc.
├── db/
│   ├── migrate/               # Database migrations
│   └── seeds/                 # Seed data
└── spec/                      # RSpec tests
```

**Key Commands:**
```bash
# Generate model
rails g model Student teacher:references first_name:string

# Generate controller
rails g controller api/v1/students

# Run migrations
rails db:migrate

# Run tests
bundle exec rspec

# Rails console
rails console

# Database console
rails dbconsole
```

**Common Patterns:**
- All controllers inherit from `Api::V1::BaseController`
- Use `render_success`, `render_error` helpers
- JWT authentication via `before_action :authenticate_request`
- Pagination with Kaminari gem

### Mobile Development

**Project Structure:**
```
homeschool-mobile/
├── src/
│   ├── api/                   # API service layer
│   ├── components/            # React components
│   ├── hooks/                 # Custom hooks
│   ├── screens/               # Screen components
│   ├── navigation/            # React Navigation config
│   ├── store/                 # Zustand stores
│   ├── types/                 # TypeScript types
│   └── utils/                 # Utilities
├── assets/                    # Images, fonts
└── app.json                   # Expo configuration
```

**Key Commands:**
```bash
# Start development server
npx expo start

# Run tests
npm test

# Type check
npm run type-check

# Lint
npm run lint

# Build for iOS
eas build --platform ios

# Build for Android
eas build --platform android
```

**Common Patterns:**
- Use React Query for server state
- Use Zustand for client state (auth, UI)
- All API calls go through `/src/api/` service layer
- Components styled with NativeWind (Tailwind)
- Navigation types defined in `types/navigation.ts`

---

## API Reference

### Base URL

**Development:** `http://localhost:3000/api/v1`  
**Production:** `https://api.homeschoolapp.com/api/v1`

### Authentication

All authenticated endpoints require a JWT token in the Authorization header:

```bash
Authorization: Bearer YOUR_ACCESS_TOKEN
```

**Token Lifecycle:**
- Access tokens expire in 1 hour
- Refresh tokens expire in 30 days
- Use `/auth/refresh` to get a new access token

### Quick Reference - Key Endpoints

**Authentication:**
```
POST   /auth/register           # Create account
POST   /auth/login              # Login
POST   /auth/refresh            # Refresh access token
POST   /auth/logout             # Logout
POST   /auth/password/reset-request
POST   /auth/password/reset
```

**Students:**
```
GET    /students                # List all students
GET    /students/:id            # Get student details
POST   /students                # Create student
PUT    /students/:id            # Update student
DELETE /students/:id            # Delete student
```

**Calendar:**
```
GET    /calendar/events         # List events
GET    /calendar/events/:id     # Get event details
POST   /calendar/events         # Create event
PUT    /calendar/events/:id     # Update event
DELETE /calendar/events/:id     # Delete event
PATCH  /calendar/events/:id/attendance/:student_id
```

**Assignments:**
```
GET    /assignments             # List assignments
GET    /assignments/:id         # Get assignment
POST   /assignments             # Create assignment
PUT    /assignments/:id         # Update assignment
DELETE /assignments/:id         # Delete assignment
PATCH  /assignments/:id/complete
```

**Tasks:**
```
GET    /tasks                   # List tasks
GET    /tasks/:id               # Get task
POST   /tasks                   # Create task
PUT    /tasks/:id               # Update task
DELETE /tasks/:id               # Delete task
PATCH  /tasks/:id/complete
```

For complete API documentation, see [API Specification](./api-specification.md).

---

## Deployment

### Backend - Heroku (MVP)

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
heroku config:set RAILS_SERVE_STATIC_FILES=true

# Deploy
git push heroku main

# Run migrations
heroku run rails db:migrate

# Run seeds
heroku run rails db:seed

# Open app
heroku open
```

### Mobile - Expo EAS

```bash
# Install EAS CLI
npm install -g eas-cli

# Login to Expo
eas login

# Configure project
eas build:configure

# Build for iOS
eas build --platform ios

# Build for Android
eas build --platform android

# Submit to App Store
eas submit --platform ios

# Submit to Play Store
eas submit --platform android
```

### Environment Variables

**Backend (.env):**
```bash
DATABASE_URL=postgres://...
JWT_SECRET_KEY=your-secret-key
JWT_ACCESS_TOKEN_EXPIRY=3600
JWT_REFRESH_TOKEN_EXPIRY=2592000
REDIS_URL=redis://localhost:6379/0
AWS_ACCESS_KEY_ID=your-key
AWS_SECRET_ACCESS_KEY=your-secret
AWS_REGION=us-east-1
AWS_BUCKET=homeschool-uploads
```

**Mobile (.env):**
```bash
EXPO_PUBLIC_API_BASE_URL=https://api.homeschoolapp.com/v1
EXPO_PUBLIC_API_TIMEOUT=30000
EXPO_PUBLIC_ENABLE_BIOMETRICS=true
EXPO_PUBLIC_ENABLE_OFFLINE_MODE=true
```

---

## Additional Resources

### Documentation Files

- [Database Architecture](./database-architecture.md) - Complete schema design
- [API Specification](./api-specification.md) - All endpoints documented
- [Mobile Architecture](./mobile-architecture.md) - Frontend patterns
- [Implementation Guide](./rails-implementation-guide.md) - Build instructions

### External Links

- [Ruby on Rails Guides](https://guides.rubyonrails.org/)
- [React Native Documentation](https://reactnative.dev/)
- [Expo Documentation](https://docs.expo.dev/)
- [React Query Docs](https://tanstack.com/query/latest)
- [Zustand Documentation](https://docs.pmnd.rs/zustand/getting-started/introduction)

---

## Contributing

### Development Workflow

1. Create a new branch: `git checkout -b feature/your-feature`
2. Make your changes
3. Write/update tests
4. Commit: `git commit -m "feat: add new feature"`
5. Push: `git push origin feature/your-feature`
6. Create Pull Request

### Commit Convention

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add student profile images
fix: resolve calendar timezone issue
docs: update API documentation
chore: upgrade dependencies
refactor: extract validation logic
test: add assignment tests
```

### Code Style

**Backend:**
- Follow [Ruby Style Guide](https://rubystyle.guide/)
- Run `rubocop` before committing
- Keep controllers thin, models fat
- Use service objects for complex logic

**Mobile:**
- Follow [Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript)
- Run `eslint` before committing
- Use TypeScript for all files
- Keep components under 200 lines

---

## Support

### Getting Help

- Email: support@homeschoolapp.com
- Slack: [homeschool-dev.slack.com]
- Issues: [GitHub Issues](https://github.com/YOUR-ORG/homeschool-master/issues)

### Reporting Bugs

Please include:
1. Description of the issue
2. Steps to reproduce
3. Expected behavior
4. Actual behavior
5. Screenshots (if applicable)
6. Environment (OS, browser, device)

---

## License

MIT License - see [LICENSE](../LICENSE) file for details

---

## Changelog

### v1.0.0 (Upcoming)
- Initial MVP release
- Core features: Students, Calendar, Assignments, Tasks
- iOS and Android mobile apps
- RESTful API with JWT authentication

---

**Last Updated:** December 2025  
**Status:** In Development  
**Version:** 0.1.0 (Pre-release)
