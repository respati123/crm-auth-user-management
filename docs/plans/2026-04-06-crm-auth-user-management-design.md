# CRM Authentication & User Management - Design Document

**Date:** 2026-04-06
**Author:** Respati
**Status:** Approved
**Target Timeline:** 1 Week (Aggressive MVP)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Overview](#system-overview)
3. [Technical Architecture](#technical-architecture)
4. [Database Schema](#database-schema)
5. [API Endpoints](#api-endpoints)
6. [Authentication Flow](#authentication-flow)
7. [Frontend Architecture](#frontend-architecture)
8. [Error Handling Strategy](#error-handling-strategy)
9. [Testing Strategy](#testing-strategy)
10. [Security Considerations](#security-considerations)
11. [Deployment Strategy](#deployment-strategy)
12. [Implementation Timeline](#implementation-timeline)

---

## Executive Summary

This document outlines the design for a CRM application focused on Authentication and User Management as the MVP. The system follows a modern, scalable architecture using Bun + Hono for the backend and Next.js with a functional MVVM pattern for the frontend.

**Key Features (MVP):**
- User authentication (Login, Register, Logout, Password Reset)
- User management with CRUD operations
- Role-based access control (Admin, Staff, Customer)
- Soft delete functionality
- Admin dashboard with user statistics
- Multi-language support (Bahasa Indonesia & English)

**Tech Stack:**
- **Backend:** Bun + Hono + Drizzle ORM + PostgreSQL + Redis
- **Frontend:** Next.js 14 + TypeScript + Tailwind CSS
- **Deployment:** Docker + Docker Compose + Nginx
- **Testing:** Bun Test (Backend), Vitest (Frontend)

---

## System Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (Next.js)                 │
│  - Login/Register Pages                               │
│  - Admin Dashboard                                    │
│  - User Management UI                                 │
└─────────────────┬───────────────────────────────────────┘
                  │ HTTP/HTTPS
                  │
┌─────────────────▼───────────────────────────────────────┐
│              Backend (Bun + Hono)                     │
│  ┌───────────────────────────────────────────────┐    │
│  │  Routes Layer                                │    │
│  │  - /api/auth/*                              │    │
│  │  - /api/users/*                             │    │
│  │  - /api/admin/*                             │    │
│  └───────────────────┬───────────────────────────┘    │
│  ┌───────────────────▼───────────────────────────┐    │
│  │  Controllers Layer                           │    │
│  │  - AuthController                           │    │
│  │  - UsersController                          │    │
│  │  - AdminController                          │    │
│  └───────────────────┬───────────────────────────┘    │
│  ┌───────────────────▼───────────────────────────┐    │
│  │  Services Layer                             │    │
│  │  - AuthService (JWT, Password, etc.)        │    │
│  │  - UserService (CRUD, validation)           │    │
│  │  - AdminService (permissions, reports)       │    │
│  └───────────────────┬───────────────────────────┘    │
│  ┌───────────────────▼───────────────────────────┐    │
│  │  Repositories Layer                          │    │
│  │  - UserRepository                           │    │
│  └───────────────────┬───────────────────────────┘    │
└───────────────────┬─────────────────────────────────────┘
                  │ Drizzle ORM
                  │
┌───────────────────▼─────────────────────────────────────┐
│              PostgreSQL Database                        │
│  - users table                                       │
│  - refresh_tokens table (for Redis fallback)            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   Redis                               │
│  - Session storage                                   │
│  - Refresh tokens                                    │
│  - Rate limiting                                     │
└─────────────────────────────────────────────────────────┘
```

---

## Technical Architecture

### Layered Architecture (Backend)

**Principles:**
- **Separation of Concerns:** Each layer has a specific responsibility
- **Dependency Injection:** Services depend on interfaces, not implementations
- **Single Responsibility:** Each class/function has one reason to change

**Layers:**

1. **Routes Layer:** HTTP endpoints, request validation, middleware
2. **Controllers Layer:** Business logic orchestration, request/response handling
3. **Services Layer:** Core business logic, transactions, external service integration
4. **Repositories Layer:** Data access abstraction, SQL query execution

### Functional MVVM Pattern (Frontend)

**Principles:**
- **No Classes:** All functions and hooks
- **State & Actions Separation:** ViewModels return `{ state, actions }`
- **Pure Views:** Components are pure presentation, receive state & actions as props
- **Type Safety:** TypeScript enforces correct structure

**Pattern Structure:**
```typescript
// ViewModel returns
{
  state: {
    // Data only - pure state
    users: User[],
    loading: boolean,
    error: string | null,
    searchQuery: string,
    selectedRole?: 'admin' | 'staff' | 'customer',
    pagination: { page, limit, total }
  },
  actions: {
    // Functions only - pure functions
    loadUsers: () => Promise<void>,
    handleSearch: (query: string) => void,
    handleDelete: (userId: string) => Promise<void>,
    handleRestore: (userId: string) => Promise<void>
  }
}
```

---

## Database Schema

### Tables

#### users table

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  role VARCHAR(20) NOT NULL DEFAULT 'customer' CHECK (role IN ('admin', 'staff', 'customer')),
  is_active BOOLEAN NOT NULL DEFAULT true,
  email_verified BOOLEAN NOT NULL DEFAULT false,
  avatar_url VARCHAR(500),
  phone VARCHAR(20),
  is_deleted BOOLEAN NOT NULL DEFAULT false,
  deleted_at TIMESTAMP,
  deleted_by UUID REFERENCES users(id),
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  last_login TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_is_deleted ON users(is_deleted);
CREATE INDEX idx_users_is_active ON users(is_active);
```

#### refresh_tokens table

```sql
CREATE TABLE refresh_tokens (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  token VARCHAR(500) UNIQUE NOT NULL,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  revoked BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_token ON refresh_tokens(token);
```

### Soft Delete Strategy

**Implementation:**
- `is_deleted = false` → User is active
- `is_deleted = true` → User is soft-deleted
- `deleted_at` → Timestamp of deletion
- `deleted_by` → ID of user who performed the deletion

**Benefits:**
- Data preservation
- Ability to restore deleted users
- Audit trail
- Compliance with data retention policies

---

## API Endpoints

### Auth Endpoints

```
POST   /api/auth/register              - Register new user
POST   /api/auth/login                 - Login with email/password
POST   /api/auth/logout                - Logout (revoke refresh token)
POST   /api/auth/refresh               - Refresh access token
POST   /api/auth/forgot-password        - Request password reset
POST   /api/auth/reset-password         - Reset password with token
GET    /api/auth/me                    - Get current user info
```

### User Endpoints (Authenticated)

```
GET    /api/users                      - List users (admin/staff only, exclude soft deleted)
GET    /api/users/:id                 - Get user by ID (admin/staff/own profile)
POST   /api/users                      - Create new user (admin only)
PUT    /api/users/:id                 - Update user (admin/staff/own profile)
DELETE /api/users/:id                 - Soft delete user (admin only)
```

### Admin Endpoints (Admin Only)

```
GET    /api/admin/dashboard            - Dashboard stats
GET    /api/admin/users/stats          - User statistics
PUT    /api/admin/users/:id/role      - Change user role
PUT    /api/admin/users/:id/status    - Activate/deactivate user
GET    /api/admin/users/deleted        - List soft-deleted users
PUT    /api/admin/users/:id/restore   - Restore soft-deleted user
```

### Response Format

**Success Response:**
```json
{
  "success": true,
  "message": "User berhasil dibuat",
  "data": { ... },
  "locale": "id"
}
```

**Error Response:**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Format email tidak valid",
    "details": { ... },
    "requestId": "uuid-for-tracking"
  },
  "timestamp": "2024-01-01T00:00:00Z"
}
```

---

## Authentication Flow

### Login Flow

1. **User** submits email and password
2. **Backend** validates input (email format, password presence)
3. **Backend** finds user by email (check !is_deleted)
4. **Backend** verifies password (bcrypt compare)
5. **Backend** checks user is active
6. **Backend** generates JWT access token (15 min expiry)
7. **Backend** generates refresh token (random string)
8. **Backend** stores refresh token in Redis with expiry
9. **Backend** updates last_login timestamp
10. **Backend** returns access token and refresh token
11. **Frontend** stores access token in memory
12. **Frontend** stores refresh token in httpOnly cookie (set by backend)
13. **Frontend** redirects to dashboard

### Token Refresh Flow

1. **Frontend** detects access token expired (401 response)
2. **Frontend** sends refresh token to /api/auth/refresh
3. **Backend** validates refresh token in Redis
4. **Backend** checks token not expired and not revoked
5. **Backend** generates new access token (JWT)
6. **Backend** optionally rotates refresh token
7. **Backend** returns new access token and refresh token
8. **Frontend** updates access token in memory
9. **Frontend** retries original request

### Logout Flow

1. **Frontend** sends logout request to /api/auth/logout
2. **Backend** deletes refresh token from Redis
3. **Backend** optionally marks token as revoked in DB
4. **Backend** clears all user sessions from Redis
5. **Frontend** clears access token from memory
6. **Frontend** redirects to login page

### JWT Payload Structure

```json
{
  "sub": "user_id",
  "email": "user@example.com",
  "role": "admin",
  "iat": 1234567890,
  "exp": 1234568790
}
```

### Redis Storage Pattern

```
Key: refresh_token:<token_id>
Value: { user_id, expires_at, created_at, device_info }
TTL: expires_at (auto-delete)

Key: session:<user_id>:<session_id>
Value: { last_activity, ip_address }
TTL: 24 hours
```

---

## Frontend Architecture

### Project Structure

```
frontend/
├── app/
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   ├── (dashboard)/
│   │   ├── admin/dashboard/page.tsx
│   │   ├── admin/users/page.tsx
│   │   └── layout.tsx
│   ├── layout.tsx
│   └── page.tsx
├── models/
│   ├── types.ts
│   ├── user.ts
│   └── enums.ts
├── view-models/
│   ├── useAuth.ts
│   ├── useUsers.ts
│   └── useDashboard.ts
├── views/
│   ├── auth/LoginView.tsx
│   ├── dashboard/UsersView.tsx
│   └── layout/SidebarView.tsx
├── services/
│   ├── api/authService.ts
│   └── http/httpClient.ts
├── constants/
│   ├── locales/id.ts
│   ├── locales/en.ts
│   └── i18n.ts
└── components/
    ├── users/UserTable.tsx
    └── layout/Sidebar.tsx
```

### MVVM Implementation Example

**ViewModel (Custom Hook):**
```typescript
export const useUsers = (usersService: UsersService): {
  state: UsersState;
  actions: UsersActions;
} => {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const actions = {
    loadUsers: async () => {
      setLoading(true);
      try {
        const response = await usersService.getUsers();
        setUsers(response.data);
      } catch (err) {
        setError('Failed to load users');
      } finally {
        setLoading(false);
      }
    },

    handleDelete: async (userId: string) => {
      await usersService.deleteUser(userId);
      await actions.loadUsers();
    }
  };

  return {
    state: { users, loading, error },
    actions
  };
};
```

**View (Pure Presentation):**
```typescript
export const UsersView = ({ state, actions }: UsersViewProps) => {
  return (
    <div>
      {state.loading && <Spinner />}
      {state.error && <ErrorAlert message={state.error} />}
      <UserTable users={state.users} onDelete={actions.handleDelete} />
    </div>
  );
};
```

---

## Error Handling Strategy

### Backend Error Handling

**Error Types:**
- `ValidationError` - Input validation errors (400)
- `AuthError` - Authentication errors (401)
- `NotFoundError` - Resource not found (404)
- `PermissionDeniedError` - Access denied (403)
- `InternalError` - Unexpected errors (500)

**Error Constants:**
```typescript
export const ErrorCodes = {
  AUTH_INVALID_CREDENTIALS: 'AUTH_INVALID_CREDENTIALS',
  AUTH_TOKEN_EXPIRED: 'AUTH_TOKEN_EXPIRED',
  VALIDATION_EMAIL_INVALID: 'VALIDATION_EMAIL_INVALID',
  USER_NOT_FOUND: 'USER_NOT_FOUND',
  // ... more codes
} as const;
```

### Error Messages with i18n

**Structure:**
```typescript
// locales/id.ts
export const id = {
  errors: {
    AUTH_INVALID_CREDENTIALS: 'Email atau password salah',
    AUTH_TOKEN_EXPIRED: 'Token sudah kadaluarsa',
    // ... more messages
  },
  validation: {
    email: {
      required: 'Email wajib diisi',
      invalid: 'Format email tidak valid',
    }
    // ... more validations
  }
} as const;
```

### Frontend Error Handling

**ViewModel Error Handling:**
```typescript
const actions = {
  handleDelete: async (userId: string) => {
    try {
      await usersService.deleteUser(userId);
      await actions.loadUsers();
    } catch (error) {
      setError(getErrorMessage(error));
    }
  }
};
```

---

## Testing Strategy

### Backend Testing

**Tools:**
- Bun Test (built-in)
- Testcontainers (for PostgreSQL in tests)
- Fake-Redis (for Redis mocking)

**Test Types:**
- **Unit Tests:** Services, Repositories, Utils
- **Integration Tests:** Routes, Controllers with test database
- **E2E Tests:** Full authentication and user management flows

**Coverage Target:** 30-40% (basic testing for MVP)

### Frontend Testing

**Tools:**
- Vitest (unit & integration)
- React Testing Library (component testing)
- Manual testing (E2E skipped for MVP)

**Test Types:**
- **Unit Tests:** ViewModels, Services, Utils
- **Component Tests:** Views, Components with React Testing Library
- **Manual Testing:** Full user flows

**Coverage Target:** 30-40% (basic testing for MVP)

---

## Security Considerations

### Authentication Security

**Password Requirements:**
- Minimum 8 characters
- Must contain uppercase, lowercase, and numbers
- Hashed with bcrypt (cost: 12)

**JWT Configuration:**
- Access token expiry: 15 minutes
- Refresh token expiry: 7 days
- Strong secrets (min 32 characters)
- Refresh tokens stored in Redis

### Authorization Security

**Role-Based Access Control (RBAC):**
- `admin` - Full access
- `staff` - Read/write users, no admin operations
- `customer` - Read own data only

### Input Validation

**Backend:**
- Zod schema validation for all endpoints
- Parameterized queries (Drizzle ORM prevents SQL injection)
- Sanitization of user inputs

**Frontend:**
- Client-side validation (duplicate with backend)
- React's built-in XSS protection
- DOMPurify for any HTML content

### Rate Limiting

**Endpoints:**
- Auth endpoints: 5 requests / 15 minutes (per IP)
- API endpoints: 100 requests / 1 minute (per user/IP)

### Security Headers

- CORS: Configured for allowed origins
- X-Frame-Options: DENY
- X-Content-Type-Options: nosniff
- X-XSS-Protection: 1; mode=block
- CSP: Strict policy
- HSTS: Enabled in production

### Secure Storage

**Frontend:**
- Access token: In-memory only (not localStorage)
- Refresh token: HttpOnly cookie (set by backend)
- No sensitive data in localStorage
- Environment variables for secrets

---

## Deployment Strategy

### Docker Configuration

**Backend Dockerfile:**
```dockerfile
FROM oven/bun:1-slim
WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --production
COPY . .
EXPOSE 3001
CMD ["bun", "run", "start"]
```

**Frontend Dockerfile:**
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
EXPOSE 3000
CMD ["node", "server.js"]
```

### Docker Compose

**Services:**
- PostgreSQL (database)
- Redis (cache & sessions)
- Backend (Bun + Hono)
- Frontend (Next.js)
- Nginx (reverse proxy & SSL)

### Deployment Environment

**For MVP (Development/Staging):**
- Single VPS (2GB RAM, 1CPU)
- Docker Compose for easy deployment
- Nginx for reverse proxy
- Let's Encrypt for free SSL certificates

**For Production:**
- Cloud provider (AWS, GCP, Azure)
- Managed services (RDS, ElastiCache)
- Load balancer for scalability
- Auto scaling for high traffic

---

## Implementation Timeline

### Aggressive 1-Week Schedule

**Day 1: Setup & Backend Foundation**
- Backend & Frontend project setup
- Docker, PostgreSQL, Redis setup
- Drizzle schema (users table)
- Database seeding (admin user)

**Day 2: Backend Core Features**
- Authentication system (JWT, Password hashing)
- Login/Register/Forgot Password endpoints
- User CRUD operations (Create, Read, Update, Delete)
- Soft delete implementation

**Day 3: Backend Admin & Frontend Auth**
- Admin dashboard endpoints (basic stats)
- User management endpoints (list, filter, pagination)
- Frontend: Next.js setup
- Login page + Register page

**Day 4: Frontend Dashboard**
- Admin Dashboard UI (basic stats cards)
- User Management UI (table with search & filter)
- User create/edit/delete functionality
- Restore soft-deleted users

**Day 5: Integration, Testing, Deployment**
- Error handling integration
- Basic unit tests (auth & users services)
- Manual testing (auth flow, user management)
- Docker & Docker Compose setup
- Basic deployment setup

### Trade-offs for 1-Week Timeline

**Scope Reduction:**
- Comprehensive testing: Only basic tests (30-40% coverage)
- E2E testing: Skipped, manual testing only
- Complete i18n: May start with Bahasa Indonesia only
- Advanced security: Rate limiting may be simplified
- Production-ready deployment: Development deployment first

**Post-MVP Work (Week 2+):**
- Comprehensive testing (80%+ coverage)
- E2E testing with Playwright
- Complete i18n (English + Indonesian)
- Advanced security (proper rate limiting)
- Production deployment optimization
- Refactoring & code quality improvements
- Documentation & README

---

## Conclusion

This design provides a comprehensive blueprint for building a CRM Authentication & User Management system as an MVP. The architecture is designed to be:

- **Modern:** Using latest technologies (Bun, Hono, Next.js 14)
- **Scalable:** Layered architecture and modular design
- **Maintainable:** Functional MVVM pattern and clean separation of concerns
- **Secure:** Comprehensive security measures at all layers
- **International:** Multi-language support from the start
- **Production-ready:** Docker deployment and CI/CD pipeline

The aggressive 1-week timeline is achievable with the scope reductions and trade-offs outlined above. Quality improvements and additional features can be added in subsequent iterations.

---

**Next Steps:**
1. Review and approve this design document
2. Create detailed implementation plan
3. Begin Day 1 implementation tasks

---

*Document Version: 1.0*
*Last Updated: 2026-04-06*
