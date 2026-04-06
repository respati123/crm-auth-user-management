# CRM Authentication & User Management Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a complete CRM application with authentication and user management as MVP in 1 week.

**Architecture:** Monolithic with layered architecture (Routes → Controllers → Services → Repositories) for backend using Bun + Hono + Drizzle ORM + PostgreSQL + Redis. Frontend using Next.js 14 with functional MVVM pattern (state & actions separation) for clean separation of concerns.

**Tech Stack:**
- **Backend:** Bun, Hono, Drizzle ORM, PostgreSQL, Redis, Zod, bcrypt
- **Frontend:** Next.js 14, React 18, TypeScript, Tailwind CSS, Vitest
- **DevOps:** Docker, Docker Compose, Nginx
- **Testing:** Bun Test (backend), Vitest (frontend), React Testing Library

---

## Day 1: Setup & Backend Foundation

### Task 1: Create Backend Project Structure

**Files:**
- Create: `backend/package.json`
- Create: `backend/bun.lockb`
- Create: `backend/tsconfig.json`
- Create: `backend/.gitignore`
- Create: `backend/src/app.ts`
- Create: `backend/src/server.ts`

**Step 1: Create backend package.json**

```json
{
  "name": "crm-backend",
  "version": "0.1.0",
  "scripts": {
    "dev": "bun --watch src/server.ts",
    "start": "bun src/server.ts",
    "build": "bun build ./src",
    "lint": "eslint src",
    "test": "bun test",
    "test:coverage": "bun test --coverage",
    "db:generate": "bun drizzle-kit generate",
    "db:migrate": "bun src/db/migrate.ts",
    "db:push": "bun drizzle-kit push",
    "db:studio": "bun drizzle-kit studio",
    "db:seed": "bun src/db/seed.ts"
  },
  "dependencies": {
    "hono": "^4.0.0",
    "@hono/zod-validator": "^0.2.0",
    "drizzle-orm": "^0.30.0",
    "postgres": "^3.4.0",
    "ioredis": "^5.3.0",
    "bcrypt": "^5.1.0",
    "jsonwebtoken": "^9.0.0",
    "zod": "^3.22.0",
    "nanoid": "^5.0.0"
  },
  "devDependencies": {
    "drizzle-kit": "^0.20.0",
    "typescript": "^5.3.0",
    "eslint": "^8.55.0",
    "@types/bcrypt": "^5.0.0",
    "@types/jsonwebtoken": "^9.0.0",
    "bun-types": "latest"
  }
}
```

**Step 2: Create TypeScript config**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "lib": ["ES2022"],
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "baseUrl": "./src",
    "paths": {
      "@/*": ["./*"]
    },
    "types": ["bun-types"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**Step 3: Create .gitignore**

```
node_modules
dist
.env
.env.local
.env.*.local
coverage
*.log
```

**Step 4: Create basic Hono app**

```typescript
// backend/src/app.ts
import { Hono } from 'hono';
import { cors } from 'hono/cors';

const app = new Hono();

// CORS middleware
app.use('/*', cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
}));

// Health check
app.get('/health', (c) => {
  return c.json({ status: 'ok', timestamp: new Date().toISOString() });
});

export default app;
```

**Step 5: Create server entry point**

```typescript
// backend/src/server.ts
import app from './app';

const port = Number(process.env.PORT) || 3001;

console.log(`🚀 Server starting on port ${port}...`);

Bun.serve({
  port,
  fetch: app.fetch,
});
```

**Step 6: Commit**

```bash
cd backend
git add .
git commit -m "feat: initialize backend project structure with Hono"
```

---

### Task 2: Setup Docker & Database Services

**Files:**
- Create: `docker-compose.yml`
- Create: `backend/.env.example`
- Create: `backend/.env`

**Step 1: Create Docker Compose configuration**

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: crm-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: crm_db
      POSTGRES_USER: crm_user
      POSTGRES_PASSWORD: crm_password_dev
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U crm_user -d crm_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: crm-redis
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: crm-backend
    restart: unless-stopped
    environment:
      NODE_ENV: development
      PORT: 3001
      DATABASE_URL: postgresql://crm_user:crm_password_dev@postgres:5432/crm_db
      REDIS_URL: redis://redis:6379
      JWT_SECRET: dev-secret-change-in-production
      REFRESH_TOKEN_SECRET: dev-refresh-secret
      ALLOWED_ORIGINS: http://localhost:3000,http://localhost:5173
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "3001:3001"
    volumes:
      - ./backend:/app
      - /app/node_modules

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
```

**Step 2: Create backend Dockerfile**

```dockerfile
FROM oven/bun:1 AS builder
WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile --production
COPY . .

FROM oven/bun:1-slim
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
COPY --from=builder /app/src ./src
EXPOSE 3001
CMD ["bun", "run", "start"]
```

**Step 3: Create .env.example**

```bash
NODE_ENV=development
PORT=3001
DATABASE_URL=postgresql://localhost:5432/crm_db
REDIS_URL=redis://localhost:6379
JWT_SECRET=your-super-secret-jwt-key-change-this-in-production
REFRESH_TOKEN_SECRET=your-super-secret-refresh-key-change-this-in-production
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
```

**Step 4: Create .env**

```bash
NODE_ENV=development
PORT=3001
DATABASE_URL=postgresql://localhost:5432/crm_db
REDIS_URL=redis://localhost:6379
JWT_SECRET=dev-secret-change-in-production-min-32-chars
REFRESH_TOKEN_SECRET=dev-refresh-secret-min-32-chars
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
```

**Step 5: Test Docker services**

Run: `docker-compose up -d postgres redis`
Expected: PostgreSQL and Redis containers running and healthy

**Step 6: Commit**

```bash
git add docker-compose.yml backend/Dockerfile backend/.env.example
git commit -m "feat: add Docker Compose with PostgreSQL and Redis"
```

---

### Task 3: Setup Drizzle ORM & Database Schema

**Files:**
- Create: `backend/drizzle.config.ts`
- Create: `backend/src/db/schema.ts`
- Create: `backend/src/db/migrate.ts`
- Create: `backend/src/db/index.ts`

**Step 1: Create Drizzle config**

```typescript
// backend/drizzle.config.ts
import type { Config } from 'drizzle-kit';
import 'dotenv/config';

export default {
  schema: './src/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
} satisfies Config;
```

**Step 2: Create database schema**

```typescript
// backend/src/db/schema.ts
import { pgTable, uuid, varchar, boolean, timestamp, index } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  passwordHash: varchar('password_hash', { length: 255 }).notNull(),
  firstName: varchar('first_name', { length: 100 }).notNull(),
  lastName: varchar('last_name', { length: 100 }).notNull(),
  role: varchar('role', { length: 20 }).notNull().default('customer'),
  isActive: boolean('is_active').notNull().default(true),
  emailVerified: boolean('email_verified').notNull().default(false),
  avatarUrl: varchar('avatar_url', { length: 500 }),
  phone: varchar('phone', { length: 20 }),
  isDeleted: boolean('is_deleted').notNull().default(false),
  deletedAt: timestamp('deleted_at'),
  deletedBy: uuid('deleted_by'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
  lastLogin: timestamp('last_login'),
}, (table) => ({
  emailIdx: index('users_email_idx').on(table.email),
  isDeletedIdx: index('users_is_deleted_idx').on(table.isDeleted),
  isActiveIdx: index('users_is_active_idx').on(table.isActive),
}));

export const refreshTokens = pgTable('refresh_tokens', {
  id: uuid('id').defaultRandom().primaryKey(),
  token: varchar('token', { length: 500 }).notNull().unique(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  expiresAt: timestamp('expires_at').notNull(),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  revoked: boolean('revoked').notNull().default(false),
}, (table) => ({
  userIdIdx: index('refresh_tokens_user_id_idx').on(table.userId),
  tokenIdx: index('refresh_tokens_token_idx').on(table.token),
}));

export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
export type RefreshToken = typeof refreshTokens.$inferSelect;
export type NewRefreshToken = typeof refreshTokens.$inferInsert;
```

**Step 3: Create migration script**

```typescript
// backend/src/db/migrate.ts
import { migrate } from 'drizzle-orm/postgres-js/migrator';
import { db } from './index';
import config from '../../drizzle.config';

async function main() {
  console.log('🔄 Running migrations...');
  await migrate(db, { migrationsFolder: config.out });
  console.log('✅ Migrations completed!');
  process.exit(0);
}

main().catch((err) => {
  console.error('❌ Migration failed:', err);
  process.exit(1);
});
```

**Step 4: Create database connection**

```typescript
// backend/src/db/index.ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema';

if (!process.env.DATABASE_URL) {
  throw new Error('DATABASE_URL environment variable is not set');
}

const client = postgres(process.env.DATABASE_URL);
export const db = drizzle(client, { schema });

export type DB = typeof db;
```

**Step 5: Generate and run migrations**

Run: `bun run db:generate`
Expected: Migration files generated in drizzle folder

Run: `bun run db:push`
Expected: Tables created in PostgreSQL

**Step 6: Commit**

```bash
git add backend/drizzle.config.ts backend/src/db
git commit -m "feat: setup Drizzle ORM with users and refresh_tokens tables"
```

---

### Task 4: Create Database Seeder for Admin User

**Files:**
- Create: `backend/src/db/seed.ts`

**Step 1: Create seeder script**

```typescript
// backend/src/db/seed.ts
import { db } from './index';
import { users } from './schema';
import bcrypt from 'bcrypt';

async function seed() {
  console.log('🌱 Starting database seed...');

  // Check if admin already exists
  const existingAdmin = await db.select().from(users)
    .where(sql`${users.email} = 'admin@example.com'}`)
    .limit(1);

  if (existingAdmin.length > 0) {
    console.log('✅ Admin user already exists');
    process.exit(0);
  }

  // Hash password
  const passwordHash = await bcrypt.hash('password123', 12);

  // Insert admin user
  await db.insert(users).values({
    email: 'admin@example.com',
    passwordHash,
    firstName: 'Admin',
    lastName: 'User',
    role: 'admin',
    isActive: true,
    emailVerified: true,
  });

  console.log('✅ Admin user created successfully!');
  console.log('📧 Email: admin@example.com');
  console.log('🔑 Password: password123');
  process.exit(0);
}

seed().catch((err) => {
  console.error('❌ Seed failed:', err);
  process.exit(1);
});
```

**Step 2: Run seeder**

Run: `bun run db:seed`
Expected: Admin user created in database

**Step 3: Test admin user creation**

Run: `docker exec -it crm-postgres psql -U crm_user -d crm_db -c "SELECT * FROM users;"`
Expected: Admin user data returned

**Step 4: Commit**

```bash
git add backend/src/db/seed.ts
git commit -m "feat: add database seeder for admin user"
```

---

## Day 2: Backend Core Features

### Task 5: Implement Password & JWT Utilities

**Files:**
- Create: `backend/src/utils/password.ts`
- Create: `backend/src/utils/jwt.ts`
- Create: `backend/src/utils/nanoid.ts`

**Step 1: Create password utility**

```typescript
// backend/src/utils/password.ts
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12;

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}

export function validatePasswordStrength(password: string): boolean {
  if (password.length < 8) return false;
  if (password.length > 128) return false;
  
  const hasUppercase = /[A-Z]/.test(password);
  const hasLowercase = /[a-z]/.test(password);
  const hasNumber = /\d/.test(password);
  
  return hasUppercase && hasLowercase && hasNumber;
}
```

**Step 2: Create JWT utility**

```typescript
// backend/src/utils/jwt.ts
import jwt from 'jsonwebtoken';

const JWT_SECRET = process.env.JWT_SECRET || 'dev-secret';
const REFRESH_TOKEN_SECRET = process.env.REFRESH_TOKEN_SECRET || 'dev-refresh-secret';

export interface JWTPayload {
  sub: string;
  email: string;
  role: string;
}

export function generateAccessToken(payload: JWTPayload): string {
  return jwt.sign(payload, JWT_SECRET, {
    expiresIn: '15m',
    issuer: 'crm-app',
    audience: 'crm-users',
  });
}

export function verifyAccessToken(token: string): JWTPayload {
  try {
    return jwt.verify(token, JWT_SECRET) as JWTPayload;
  } catch (error) {
    throw new Error('Invalid or expired token');
  }
}

export function generateRefreshTokenId(): string {
  return jwt.sign(
    { type: 'refresh' },
    REFRESH_TOKEN_SECRET,
    { expiresIn: '7d' }
  );
}

export function verifyRefreshToken(token: string): JWTPayload & { type: string } {
  try {
    return jwt.verify(token, REFRESH_TOKEN_SECRET) as JWTPayload & { type: string };
  } catch (error) {
    throw new Error('Invalid or expired refresh token');
  }
}
```

**Step 3: Create nanoid utility**

```typescript
// backend/src/utils/nanoid.ts
import { nanoid } from 'nanoid';

export function generateId(): string {
  return nanoid();
}

export function generateToken(): string {
  return nanoid(32);
}
```

**Step 4: Write tests for password utility**

```typescript
// backend/src/tests/unit/utils/password.test.ts
import { describe, it, expect } from 'bun:test';
import { hashPassword, verifyPassword, validatePasswordStrength } from '../password';

describe('Password Utility', () => {
  it('should hash and verify password correctly', async () => {
    const password = 'testPassword123';
    const hash = await hashPassword(password);
    const isValid = await verifyPassword(password, hash);
    expect(isValid).toBe(true);
  });

  it('should reject incorrect password', async () => {
    const password = 'testPassword123';
    const hash = await hashPassword(password);
    const isValid = await verifyPassword('wrongPassword', hash);
    expect(isValid).toBe(false);
  });

  it('should validate strong password', () => {
    expect(validatePasswordStrength('StrongPass123')).toBe(true);
  });

  it('should reject weak password', () => {
    expect(validatePasswordStrength('weak')).toBe(false);
    expect(validatePasswordStrength('alllowercase')).toBe(false);
    expect(validatePasswordStrength('ALLUPPERCASE')).toBe(false);
    expect(validatePasswordStrength('NoNumbers')).toBe(false);
  });
});
```

**Step 5: Run tests**

Run: `bun test src/tests/unit/utils/password.test.ts`
Expected: All tests pass

**Step 6: Commit**

```bash
git add backend/src/utils
git commit -m "feat: add password, JWT, and ID generation utilities"
```

---

### Task 6: Create Custom Error Classes

**Files:**
- Create: `backend/src/utils/errors.ts`

**Step 1: Create error classes**

```typescript
// backend/src/utils/errors.ts'
export class AppError extends Error {
  constructor(
    message: string,
    public statusCode: number = 500,
    public code: string = 'INTERNAL_ERROR',
    public details?: any
  ) {
    super(message);
    this.name = 'AppError';
  }
}

export class ValidationError extends AppError {
  constructor(message: string, details?: any) {
    super(message, 400, 'VALIDATION_ERROR', details);
    this.name = 'ValidationError';
  }
}

export class AuthError extends AppError {
  constructor(message: string) {
    super(message, 401, 'AUTH_ERROR');
    this.name = 'AuthError';
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404, 'NOT_FOUND');
    this.name = 'NotFoundError';
  }
}

export class PermissionDeniedError extends AppError {
  constructor(message: string) {
    super(message, 403, 'PERMISSION_DENIED');
    this.name = 'PermissionDeniedError';
  }
}
```

**Step 2: Commit**

```bash
git add backend/src/utils/errors.ts
git commit -m "feat: add custom error classes"
```

---

### Task 7: Create Error & Locale Constants

**Files:**
- Create: `backend/src/constants/errors.ts`
- Create: `backend/src/constants/validation.ts`
- Create: `backend/src/constants/messages.ts`
- Create: `backend/src/constants/locales/id.ts`
- Create: `backend/src/constants/locales/en.ts`
- Create: `backend/src/constants/locales/index.ts`

**Step 1: Create error constants**

```typescript
// backend/src/constants/errors.ts
export const ErrorCodes = {
  AUTH_INVALID_CREDENTIALS: 'AUTH_INVALID_CREDENTIALS',
  AUTH_TOKEN_EXPIRED: 'AUTH_TOKEN_EXPIRED',
  AUTH_TOKEN_INVALID: 'AUTH_TOKEN_INVALID',
  AUTH_SESSION_EXPIRED: 'AUTH_SESSION_EXPIRED',
  AUTH_EMAIL_NOT_VERIFIED: 'AUTH_EMAIL_NOT_VERIFIED',
  AUTH_PASSWORD_MISMATCH: 'AUTH_PASSWORD_MISMATCH',
  VALIDATION_EMAIL_INVALID: 'VALIDATION_EMAIL_INVALID',
  VALIDATION_PASSWORD_TOO_SHORT: 'VALIDATION_PASSWORD_TOO_SHORT',
  VALIDATION_REQUIRED_FIELD: 'VALIDATION_REQUIRED_FIELD',
  VALIDATION_INVALID_FORMAT: 'VALIDATION_INVALID_FORMAT',
  USER_NOT_FOUND: 'USER_NOT_FOUND',
  USER_ALREADY_EXISTS: 'USER_ALREADY_EXISTS',
  USER_IS_DELETED: 'USER_IS_DELETED',
  USER_IS_INACTIVE: 'USER_IS_INACTIVE',
  PERMISSION_DENIED: 'PERMISSION_DENIED',
  INSUFFICIENT_PERMISSIONS: 'INSUFFICIENT_PERMISSIONS',
  RESOURCE_NOT_FOUND: 'RESOURCE_NOT_FOUND',
  RESOURCE_ALREADY_EXISTS: 'RESOURCE_ALREADY_EXISTS',
  INTERNAL_SERVER_ERROR: 'INTERNAL_SERVER_ERROR',
} as const;
```

**Step 2: Create validation messages**

```typescript
// backend/src/constants/validation.ts
export const ValidationMessages = {
  email: {
    required: 'Email wajib diisi',
    invalid: 'Format email tidak valid',
    exists: 'Email sudah terdaftar',
  },
  password: {
    required: 'Password wajib diisi',
    tooShort: 'Password minimal 8 karakter',
    noMatch: 'Password tidak cocok',
    weak: 'Password harus mengandung huruf besar, kecil, dan angka',
  },
  firstName: {
    required: 'Nama depan wajib diisi',
    tooLong: 'Nama depan maksimal 100 karakter',
  },
  lastName: {
    required: 'Nama belakang wajib diisi',
    tooLong: 'Nama belakang maksimal 100 karakter',
  },
  phone: {
    invalid: 'Format nomor telepon tidak valid',
  },
  role: {
    invalid: 'Role tidak valid',
  },
} as const;

export function getValidationMessage(field: string, type: string): string {
  const fieldMessages = ValidationMessages[field as keyof typeof ValidationMessages];
  if (fieldMessages && type in fieldMessages) {
    return (fieldMessages as any)[type];
  }
  return 'Input tidak valid';
}
```

**Step 3: Create success messages**

```typescript
// backend/src/constants/messages.ts
export const SuccessMessages = {
  USER_CREATED: 'User berhasil dibuat',
  USER_UPDATED: 'User berhasil diupdate',
  USER_DELETED: 'User berhasil dihapus',
  USER_RESTORED: 'User berhasil dipulihkan',
  USER_ACTIVATED: 'User berhasil diaktifkan',
  USER_DEACTIVATED: 'User berhasil dinonaktifkan',
  PASSWORD_CHANGED: 'Password berhasil diubah',
  EMAIL_VERIFIED: 'Email berhasil diverifikasi',
  LOGIN_SUCCESS: 'Login berhasil',
  LOGOUT_SUCCESS: 'Logout berhasil',
} as const;
```

**Step 4: Create Indonesian locale**

```typescript
// backend/src/constants/locales/id.ts
export const id = {
  errors: {
    AUTH_INVALID_CREDENTIALS: 'Email atau password salah',
    AUTH_TOKEN_EXPIRED: 'Token sudah kadaluarsa, silakan login kembali',
    AUTH_TOKEN_INVALID: 'Token tidak valid',
    AUTH_SESSION_EXPIRED: 'Sesi telah berakhir, silakan login kembali',
    AUTH_EMAIL_NOT_VERIFIED: 'Email belum diverifikasi',
    AUTH_PASSWORD_MISMATCH: 'Password tidak cocok',
    VALIDATION_EMAIL_INVALID: 'Format email tidak valid',
    VALIDATION_PASSWORD_TOO_SHORT: 'Password minimal 8 karakter',
    VALIDATION_REQUIRED_FIELD: 'Field ini wajib diisi',
    VALIDATION_INVALID_FORMAT: 'Format tidak valid',
    USER_NOT_FOUND: 'User tidak ditemukan',
    USER_ALREADY_EXISTS: 'Email sudah terdaftar',
    USER_IS_DELETED: 'User telah dihapus',
    USER_IS_INACTIVE: 'User tidak aktif',
    PERMISSION_DENIED: 'Anda tidak memiliki akses',
    INSUFFICIENT_PERMISSIONS: 'Izin tidak mencukupi',
    RESOURCE_NOT_FOUND: 'Resource tidak ditemukan',
    RESOURCE_ALREADY_EXISTS: 'Resource sudah ada',
    INTERNAL_SERVER_ERROR: 'Terjadi kesalahan server',
  },
  validation: {
    email: {
      required: 'Email wajib diisi',
      invalid: 'Format email tidak valid',
      exists: 'Email sudah terdaftar',
    },
    password: {
      required: 'Password wajib diisi',
      tooShort: 'Password minimal 8 karakter',
      noMatch: 'Password tidak cocok',
      weak: 'Password harus mengandung huruf dan angka',
    },
    firstName: {
      required: 'Nama depan wajib diisi',
      tooLong: 'Nama depan maksimal 100 karakter',
    },
    lastName: {
      required: 'Nama belakang wajib diisi',
      tooLong: 'Nama belakang maksimal 100 karakter',
    },
    phone: {
      invalid: 'Format nomor telepon tidak valid',
    },
    role: {
      invalid: 'Role tidak valid',
    },
  },
  success: {
    USER_CREATED: 'User berhasil dibuat',
    USER_UPDATED: 'User berhasil diupdate',
    USER_DELETED: 'User berhasil dihapus',
    USER_RESTORED: 'User berhasil dipulihkan',
    USER_ACTIVATED: 'User berhasil diaktifkan',
    USER_DEACTIVATED: 'User berhasil dinonaktifkan',
    PASSWORD_CHANGED: 'Password berhasil diubah',
    EMAIL_VERIFIED: 'Email berhasil diverifikasi',
    LOGIN_SUCCESS: 'Login berhasil',
    LOGOUT_SUCCESS: 'Logout berhasil',
  },
} as const;
```

**Step 5: Create English locale**

```typescript
// backend/src/constants/locales/en.ts
export const en = {
  errors: {
    AUTH_INVALID_CREDENTIALS: 'Invalid email or password',
    AUTH_TOKEN_EXPIRED: 'Token has expired, please login again',
    AUTH_TOKEN_INVALID: 'Invalid token',
    AUTH_SESSION_EXPIRED: 'Session has expired, please login again',
    AUTH_EMAIL_NOT_VERIFIED: 'Email has not been verified',
    AUTH_PASSWORD_MISMATCH: 'Password does not match',
    VALIDATION_EMAIL_INVALID: 'Invalid email format',
    VALIDATION_PASSWORD_TOO_SHORT: 'Password must be at least 8 characters',
    VALIDATION_REQUIRED_FIELD: 'This field is required',
    VALIDATION_INVALID_FORMAT: 'Invalid format',
    USER_NOT_FOUND: 'User not found',
    USER_ALREADY_EXISTS: 'Email already registered',
    USER_IS_DELETED: 'User has been deleted',
    USER_IS_INACTIVE: 'User is inactive',
    PERMISSION_DENIED: 'You do not have access',
    INSUFFICIENT_PERMISSIONS: 'Insufficient permissions',
    RESOURCE_NOT_FOUND: 'Resource not found',
    RESOURCE_ALREADY_EXISTS: 'Resource already exists',
    INTERNAL_SERVER_ERROR: 'An internal server error occurred',
  },
  validation: {
    email: {
      required: 'Email is required',
      invalid: 'Invalid email format',
      exists: 'Email is already registered',
    },
    password: {
      required: 'Password is required',
      tooShort: 'Password must be at least 8 characters',
      noMatch: 'Passwords do not match',
      weak: 'Password must contain letters and numbers',
    },
    firstName: {
      required: 'First name is required',
      tooLong: 'First name must be at most 100 characters',
    },
    lastName: {
      required: 'Last name is required',
      tooLong: 'Last name must be at most 100 characters',
    },
    phone: {
      invalid: 'Invalid phone number format',
    },
    role: {
      invalid: 'Invalid role',
    },
  },
  success: {
    USER_CREATED: 'User created successfully',
    USER_UPDATED: 'User updated successfully',
    USER_DELETED: 'User deleted successfully',
    USER_RESTORED: 'User restored successfully',
    USER_ACTIVATED: 'User activated successfully',
    USER_DEACTIVATED: 'User deactivated successfully',
    PASSWORD_CHANGED: 'Password changed successfully',
    EMAIL_VERIFIED: 'Email verified successfully',
    LOGIN_SUCCESS: 'Login successful',
    LOGOUT_SUCCESS: 'Logout successful',
  },
} as const;
```

**Step 6: Create locale index**

```typescript
// backend/src/constants/locales/index.ts
export { id } from './id';
export { en } from './en';

export type Locale = 'id' | 'en';
export type Translation = typeof id;

export const DEFAULT_LOCALE: Locale = 'id';
export const AVAILABLE_LOCALES: Locale[] = ['id', 'en'];
```

**Step 7: Create i18n service**

```typescript
// backend/src/services/i18nService.ts
import { Locale, id, en, DEFAULT_LOCALE } from '@/constants/locales';

let currentLocale: Locale = DEFAULT_LOCALE;
const localeCache = new Map<Locale, typeof id>();

function loadTranslations(locale: Locale): typeof id {
  if (localeCache.has(locale)) {
    return localeCache.get(locale)!;
  }

  const translations = locale === 'id' ? id : en;
  localeCache.set(locale, translations);
  return translations;
}

export function getLocale(): Locale {
  return currentLocale;
}

export function setLocale(locale: Locale): void {
  if (!['id', 'en'].includes(locale)) {
    throw new Error(`Locale '${locale}' is not supported`);
  }
  currentLocale = locale;
}

export function t(key: string, params?: Record<string, any>): string {
  const translations = loadTranslations(currentLocale);
  const keys = key.split('.');
  let value: any = translations;

  for (const k of keys) {
    value = value?.[k];
    if (value === undefined) {
      console.warn(`Translation key not found: ${key}`);
      return key;
    }
  }

  if (params && typeof value === 'string') {
    return value.replace(/\{(\w+)\}/g, (_, param) => params[param] || '');
  }

  return value;
}

export function getErrorMessage(code: string): string {
  return t(`errors.${code}`);
}

export function getValidationMessage(field: string, type: string): string {
  return t(`validation.${field}.${type}`);
}

export function getSuccessMessage(code: string): string {
  return t(`success.${code}`);
}
```

**Step 8: Commit**

```bash
git add backend/src/constants backend/src/services/i18nService.ts
git commit -m "feat: add error, validation, and locale constants with i18n support"
```

---

### Task 8: Create Error Handler Middleware

**Files:**
- Create: `backend/src/middleware/errorHandler.ts`

**Step 1: Create error handler middleware**

```typescript
// backend/src/middleware/errorHandler.ts
import { Context, Next } from 'hono';
import { getErrorMessage, t } from '@/services/i18nService';
import { ValidationError, AuthError, NotFoundError, PermissionDeniedError, AppError } from '@/utils/errors';
import { nanoid } from 'nanoid';

export async function errorHandler(c: Context, next: Next) {
  try {
    await next();
  } catch (error) {
    const requestId = c.get('requestId') || nanoid();
    
    // Log error
    console.error('[ERROR]', {
      requestId,
      error: error instanceof Error ? error.message : String(error),
      stack: error instanceof Error ? error.stack : undefined,
    });

    // Handle custom errors
    if (error instanceof ValidationError) {
      return c.json({
        success: false,
        error: {
          code: 'VALIDATION_ERROR',
          message: error.message,
          details: error.details,
          requestId
        }
      }, 400);
    }

    if (error instanceof AuthError) {
      return c.json({
        success: false,
        error: {
          code: 'AUTH_ERROR',
          message: error.message,
          requestId
        }
      }, 401);
    }

    if (error instanceof NotFoundError) {
      return c.json({
        success: false,
        error: {
          code: 'NOT_FOUND',
          message: error.message,
          requestId
        }
      }, 404);
    }

    if (error instanceof PermissionDeniedError) {
      return c.json({
        success: false,
        error: {
          code: 'PERMISSION_DENIED',
          message: error.message,
          requestId
        }
      }, 403);
    }

    // Generic internal server error
    const message = error instanceof AppError ? error.message : getErrorMessage('INTERNAL_SERVER_ERROR');
    return c.json({
      success: false,
      error: {
        code: 'INTERNAL_SERVER_ERROR',
        message,
        requestId
      }
    }, 500);
  }
}
```

**Step 2: Apply error handler to app**

```typescript
// backend/src/app.ts
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { errorHandler } from './middleware/errorHandler';

const app = new Hono();

// Error handler
app.use('/*', errorHandler);

// CORS middleware
app.use('/*', cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
}));

// Health check
app.get('/health', (c) => {
  return c.json({ status: 'ok', timestamp: new Date().toISOString() });
});

export default app;
```

**Step 3: Test error handler**

```typescript
// backend/src/tests/integration/middleware/errorHandler.test.ts
import { describe, it, expect } from 'bun:test';
import app from '@/app';

describe('Error Handler Middleware', () => {
  it('should return 404 for non-existent route', async () => {
    const response = await app.request('/api/non-existent', {
      method: 'GET'
    });

    expect(response.status).toBe(404);
    const data = await response.json();
    expect(data.success).toBe(false);
    expect(data.error.code).toBe('NOT_FOUND');
  });
});
```

**Step 4: Run tests**

Run: `bun test src/tests/integration/middleware/errorHandler.test.ts`
Expected: Test passes

**Step 5: Commit**

```bash
git add backend/src/middleware/errorHandler.ts backend/src/tests/integration/middleware/errorHandler.test.ts
git commit -m "feat: add error handler middleware with logging and structured error responses"
```

---

### Task 9: Create Authentication Service

**Files:**
- Create: `backend/src/services/authService.ts`

**Step 1: Create auth service**

```typescript
// backend/src/services/authService.ts
import { db } from '@/db';
import { users, refreshTokens } from '@/db/schema';
import { eq, and, isNull } from 'drizzle-orm';
import { hashPassword, verifyPassword, validatePasswordStrength } from '@/utils/password';
import { generateAccessToken, generateRefreshTokenId, verifyRefreshToken } from '@/utils/jwt';
import { generateToken } from '@/utils/nanoid';
import { AuthError, ValidationError, NotFoundError } from '@/utils/errors';
import { getValidationMessage, getSuccessMessage } from '@/services/i18nService';
import type { JWTPayload } from '@/utils/jwt';

export interface RegisterDTO {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
}

export interface LoginDTO {
  email: string;
  password: string;
}

export interface LoginResult {
  user: any;
  accessToken: string;
  refreshToken: string;
}

export async function register(data: RegisterDTO): Promise<LoginResult> {
  const { email, password, firstName, lastName } = data;

  // Validate email
  if (!email || !email.includes('@')) {
    throw new ValidationError(getValidationMessage('email', 'invalid'), { field: 'email' });
  }

  // Validate password
  if (!validatePasswordStrength(password)) {
    throw new ValidationError(getValidationMessage('password', 'weak'), { field: 'password' });
  }

  // Check if user exists
  const existingUser = await db.select().from(users)
    .where(eq(users.email, email))
    .limit(1);

  if (existingUser.length > 0) {
    throw new ValidationError(getValidationMessage('email', 'exists'), { field: 'email' });
  }

  // Hash password
  const passwordHash = await hashPassword(password);

  // Create user
  const [newUser] = await db.insert(users).values({
    email,
    passwordHash,
    firstName,
    lastName,
    role: 'customer',
    isActive: true,
    emailVerified: false,
  }).returning();

  return {
    user: newUser,
    accessToken: '',
    refreshToken: ''
  };
}

export async function login(data: LoginDTO): Promise<LoginResult> {
  const { email, password } = data;

  // Find user
  const [user] = await db.select().from(users)
    .where(and(
      eq(users.email, email),
      eq(users.isDeleted, false)
    ))
    .limit(1);

  if (!user) {
    throw new AuthError(getValidationMessage('email', 'invalid'));
  }

  // Verify password
  const isValid = await verifyPassword(password, user.passwordHash);
  if (!isValid) {
    throw new AuthError(getValidationMessage('email', 'invalid'));
  }

  // Check if user is active
  if (!user.isActive) {
    throw new AuthError('User tidak aktif');
  }

  // Update last login
  await db.update(users)
    .set({ lastLogin: new Date() })
    .where(eq(users.id, user.id));

  // Generate tokens
  const payload: JWTPayload = {
    sub: user.id,
    email: user.email,
    role: user.role
  };

  const accessToken = generateAccessToken(payload);
  const refreshTokenId = generateRefreshTokenId();
  const refreshToken = generateToken();

  // Store refresh token in database
  const expiresAt = new Date();
  expiresAt.setDate(expiresAt.getDate() + 7); // 7 days

  await db.insert(refreshTokens).values({
    token: refreshToken,
    userId: user.id,
    expiresAt,
    createdAt: new Date()
  });

  return {
    user: {
      id: user.id,
      email: user.email,
      firstName: user.firstName,
      lastName: user.lastName,
      role: user.role
    },
    accessToken,
    refreshToken
  };
}

export async function getUserById(userId: string): Promise<any> {
  const [user] = await db.select().from(users)
    .where(eq(users.id, userId))
    .limit(1);

  if (!user || user.isDeleted) {
    throw new NotFoundError('User');
  }

  return {
    id: user.id,
    email: user.email,
    firstName: user.firstName,
    lastName: user.lastName,
    role: user.role,
    isActive: user.isActive,
    emailVerified: user.emailVerified,
    avatarUrl: user.avatarUrl,
    phone: user.phone,
    createdAt: user.createdAt,
    updatedAt: user.updatedAt,
    lastLogin: user.lastLogin
  };
}
```

**Step 2: Write tests for auth service**

```typescript
// backend/src/tests/unit/services/authService.test.ts
import { describe, it, expect, beforeEach } from 'bun:test';
import { register, login, getUserById } from '@/services/authService';
import { db } from '@/db';
import { users } from '@/db/schema';

describe('AuthService', () => {
  beforeEach(async () => {
    // Clear database
    await db.delete(users);
  });

  describe('register', () => {
    it('should register a new user with hashed password', async () => {
      const result = await register({
        email: 'test@example.com',
        password: 'TestPass123',
        firstName: 'Test',
        lastName: 'User'
      });

      expect(result.user.email).toBe('test@example.com');
      expect(result.user.passwordHash).toBeDefined();
      expect(result.user.passwordHash).not.toBe('TestPass123');
    });

    it('should throw error if email already exists', async () => {
      await register({
        email: 'test@example.com',
        password: 'TestPass123',
        firstName: 'Test',
        lastName: 'User'
      });

      await expect(async () => {
        await register({
          email: 'test@example.com',
          password: 'AnotherPass123',
          firstName: 'Another',
          lastName: 'User'
        });
      }).toThrow();
    });

    it('should throw error for weak password', async () => {
      await expect(async () => {
        await register({
          email: 'test@example.com',
          password: 'weak',
          firstName: 'Test',
          lastName: 'User'
        });
      }).toThrow();
    });
  });

  describe('login', () => {
    it('should return tokens for valid credentials', async () => {
      await register({
        email: 'test@example.com',
        password: 'TestPass123',
        firstName: 'Test',
        lastName: 'User'
      });

      const result = await login({
        email: 'test@example.com',
        password: 'TestPass123'
      });

      expect(result.accessToken).toBeDefined();
      expect(result.refreshToken).toBeDefined();
      expect(result.user.email).toBe('test@example.com');
    });

    it('should throw error for invalid credentials', async () => {
      await register({
        email: 'test@example.com',
        password: 'TestPass123',
        firstName: 'Test',
        lastName: 'User'
      });

      await expect(async () => {
        await login({
          email: 'test@example.com',
          password: 'wrongpassword'
        });
      }).toThrow();
    });
  });
});
```

**Step 3: Run tests**

Run: `bun test src/tests/unit/services/authService.test.ts`
Expected: All tests pass

**Step 4: Commit**

```bash
git add backend/src/services/authService.ts backend/src/tests/unit/services/authService.test.ts
git commit -m "feat: add authentication service with register and login"
```

---

### Task 10: Create Auth Routes

**Files:**
- Create: `backend/src/routes/auth.ts`
- Modify: `backend/src/app.ts`

**Step 1: Create auth routes**

```typescript
// backend/src/routes/auth.ts
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';
import { register, login } from '@/services/authService';
import { getSuccessMessage } from '@/services/i18nService';

const authRouter = new Hono();

const registerSchema = z.object({
  email: z.string().email('Email tidak valid'),
  password: z.string().min(8, 'Password minimal 8 karakter'),
  firstName: z.string().min(1, 'Nama depan wajib diisi'),
  lastName: z.string().min(1, 'Nama belakang wajib diisi'),
});

const loginSchema = z.object({
  email: z.string().email('Email tidak valid'),
  password: z.string().min(1, 'Password wajib diisi'),
});

authRouter.post('/register', zValidator('json', registerSchema), async (c) => {
  const body = c.req.valid('json');
  const result = await register(body);
  
  return c.json({
    success: true,
    message: getSuccessMessage('USER_CREATED'),
    data: result.user
  });
});

authRouter.post('/login', zValidator('json', loginSchema), async (c) => {
  const body = c.req.valid('json');
  const result = await login(body);
  
  // Set refresh token in httpOnly cookie
  const expiresAt = new Date();
  expiresAt.setDate(expiresAt.getDate() + 7);
  
  c.header('Set-Cookie', `refresh_token=${result.refreshToken}; HttpOnly; Secure; SameSite=Strict; Path=/; Expires=${expiresAt.toUTCString()}`);
  
  return c.json({
    success: true,
    message: getSuccessMessage('LOGIN_SUCCESS'),
    data: {
      user: result.user,
      accessToken: result.accessToken
    }
  });
});

authRouter.post('/logout', async (c) => {
  // Clear refresh token cookie
  c.header('Set-Cookie', 'refresh_token=; HttpOnly; Secure; SameSite=Strict; Path=/; Expires=Thu, 01 Jan 1970 00:00:00 GMT');
  
  return c.json({
    success: true,
    message: getSuccessMessage('LOGOUT_SUCCESS')
  });
});

authRouter.get('/me', async (c) => {
  // Get user from context (set by auth middleware)
  const user = c.get('user');
  
  if (!user) {
    throw new Error('User not found in context');
  }
  
  return c.json({
    success: true,
    data: user
  });
});

export default authRouter;
```

**Step 2: Apply auth routes to app**

```typescript
// backend/src/app.ts
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { errorHandler } from './middleware/errorHandler';
import authRouter from './routes/auth';

const app = new Hono();

// Error handler
app.use('/*', errorHandler);

// CORS middleware
app.use('/*', cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
}));

// Routes
app.route('/api/auth', authRouter);

// Health check
app.get('/health', (c) => {
  return c.json({ status: 'ok', timestamp: new Date().toISOString() });
});

export default app;
```

**Step 3: Test auth endpoints manually**

Run: `curl -X POST http://localhost:3001/api/auth/register -H "Content-Type: application/json" -d '{"email":"test@example.com","password":"TestPass123","firstName":"Test","lastName":"User"}'`
Expected: Success response with user data

Run: `curl -X POST http://localhost:3001/api/auth/login -H "Content-Type: application/json" -d '{"email":"test@example.com","password":"TestPass123"}'`
Expected: Success response with tokens

**Step 4: Commit**

```bash
git add backend/src/routes/auth.ts
git commit -m "feat: add authentication routes (register, login, logout)"
```

---

## Day 3: Backend Admin & Frontend Auth

### Task 11: Create User Repository

**Files:**
- Create: `backend/src/repositories/userRepository.ts`

**Step 1: Create user repository**

```typescript
// backend/src/repositories/userRepository.ts
import { db } from '@/db';
import { users, refreshTokens } from '@/db/schema';
import { eq, and, isNull, desc, like, or } from 'drizzle-orm';
import { NotFoundError } from '@/utils/errors';

export interface PaginationOptions {
  page?: number;
  limit?: number;
  search?: string;
  role?: string;
}

export async function findUserById(id: string) {
  const [user] = await db.select().from(users)
    .where(eq(users.id, id))
    .limit(1);

  if (!user || user.isDeleted) {
    throw new NotFoundError('User');
  }

  return user;
}

export async function findUserByEmail(email: string) {
  const [user] = await db.select().from(users)
    .where(eq(users.email, email))
    .limit(1);

  return user;
}

export async function findAllUsers(options: PaginationOptions = {}) {
  const { page = 1, limit = 10, search, role } = options;
  const offset = (page - 1) * limit;

  let query = db.select().from(users);

  // Filter out deleted users
  query = query.where(eq(users.isDeleted, false));

  // Add search filter
  if (search) {
    const searchPattern = `%${search}%`;
    query = query.where(
      or(
        like(users.firstName, searchPattern),
        like(users.lastName, searchPattern),
        like(users.email, searchPattern)
      )
    );
  }

  // Add role filter
  if (role) {
    query = query.where(eq(users.role, role));
  }

  // Get total count
  const allUsers = await query;
  const total = allUsers.length;

  // Apply pagination
  const paginatedUsers = await query
    .orderBy(desc(users.createdAt))
    .limit(limit)
    .offset(offset);

  return {
    data: paginatedUsers,
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit)
    }
  };
}

export async function createUser(data: any) {
  const [newUser] = await db.insert(users).values(data).returning();
  return newUser;
}

export async function updateUser(id: string, data: any) {
  const [updatedUser] = await db.update(users)
    .set(data)
    .where(eq(users.id, id))
    .returning();

  if (!updatedUser) {
    throw new NotFoundError('User');
  }

  return updatedUser;
}

export async function softDeleteUser(id: string, deletedBy?: string) {
  const [deletedUser] = await db.update(users)
    .set({
      isDeleted: true,
      deletedAt: new Date(),
      deletedBy: deletedBy || null
    })
    .where(eq(users.id, id))
    .returning();

  if (!deletedUser) {
    throw new NotFoundError('User');
  }

  return deletedUser;
}

export async function restoreUser(id: string) {
  const [restoredUser] = await db.update(users)
    .set({
      isDeleted: false,
      deletedAt: null,
      deletedBy: null
    })
    .where(eq(users.id, id))
    .returning();

  if (!restoredUser) {
    throw new NotFoundError('User');
  }

  return restoredUser;
}

export async function findDeletedUsers(options: PaginationOptions = {}) {
  const { page = 1, limit = 10 } = options;
  const offset = (page - 1) * limit;

  const deletedUsers = await db.select().from(users)
    .where(eq(users.isDeleted, true))
    .orderBy(desc(users.deletedAt))
    .limit(limit)
    .offset(offset);

  return deletedUsers;
}
```

**Step 2: Commit**

```bash
git add backend/src/repositories/userRepository.ts
git commit -m "feat: add user repository with CRUD operations and soft delete"
```

---

### Task 12: Create Users Service

**Files:**
- Create: `backend/src/services/usersService.ts`

**Step 1: Create users service**

```typescript
// backend/src/services/usersService.ts
import { 
  findUserById, 
  findAllUsers, 
  createUser, 
  updateUser, 
  softDeleteUser, 
  restoreUser,
  findDeletedUsers
} from '@/repositories/userRepository';
import { getSuccessMessage } from '@/services/i18nService';
import { NotFoundError } from '@/utils/errors';

export async function getUsers(options?: any) {
  return findAllUsers(options);
}

export async function getUserById(id: string) {
  const user = await findUserById(id);
  return user;
}

export async function createUser(data: any) {
  const newUser = await createUser(data);
  return {
    ...newUser,
    message: getSuccessMessage('USER_CREATED')
  };
}

export async function updateUser(id: string, data: any) {
  const updatedUser = await updateUser(id, data);
  return {
    ...updatedUser,
    message: getSuccessMessage('USER_UPDATED')
  };
}

export async function deleteUser(id: string) {
  await softDeleteUser(id);
  return { message: getSuccessMessage('USER_DELETED') };
}

export async function restoreDeletedUser(id: string) {
  const restoredUser = await restoreUser(id);
  return {
    ...restoredUser,
    message: getSuccessMessage('USER_RESTORED')
  };
}

export async function getDeletedUsers(options?: any) {
  return findDeletedUsers(options);
}
```

**Step 2: Commit**

```bash
git add backend/src/services/usersService.ts
git commit -m "feat: add users service with business logic"
```

---

### Task 13: Create Users Routes

**Files:**
- Create: `backend/src/routes/users.ts`
- Modify: `backend/src/app.ts`

**Step 1: Create users routes**

```typescript
// backend/src/routes/users.ts
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';
import { getUsers, getUserById, createUser, updateUser, deleteUser, restoreDeletedUser } from '@/services/usersService';
import { getSuccessMessage } from '@/services/i18nService';

const usersRouter = new Hono();

const userCreateSchema = z.object({
  email: z.string().email('Email tidak valid'),
  password: z.string().min(8, 'Password minimal 8 karakter'),
  firstName: z.string().min(1, 'Nama depan wajib diisi'),
  lastName: z.string().min(1, 'Nama belakang wajib diisi'),
  role: z.enum(['admin', 'staff', 'customer'], {
    errorMap: () => ({ message: 'Role tidak valid' })
  }),
});

const userUpdateSchema = z.object({
  email: z.string().email('Email tidak valid').optional(),
  firstName: z.string().min(1, 'Nama depan wajib diisi').optional(),
  lastName: z.string().min(1, 'Nama belakang wajib diisi').optional(),
  role: z.enum(['admin', 'staff', 'customer'], {
    errorMap: () => ({ message: 'Role tidak valid' })
  }).optional(),
  isActive: z.boolean().optional(),
});

usersRouter.get('/', async (c) => {
  const page = parseInt(c.req.query('page') || '1');
  const limit = parseInt(c.req.query('limit') || '10');
  const search = c.req.query('search');
  const role = c.req.query('role');

  const result = await getUsers({ page, limit, search, role });
  
  return c.json({
    success: true,
    ...result
  });
});

usersRouter.get('/:id', async (c) => {
  const id = c.req.param('id');
  const user = await getUserById(id);
  
  return c.json({
    success: true,
    data: user
  });
});

usersRouter.post('/', zValidator('json', userCreateSchema), async (c) => {
  const body = c.req.valid('json');
  const result = await createUser(body);
  
  return c.json({
    success: true,
    message: result.message,
    data: result
  });
});

usersRouter.put('/:id', zValidator('json', userUpdateSchema), async (c) => {
  const id = c.req.param('id');
  const body = c.req.valid('json');
  const result = await updateUser(id, body);
  
  return c.json({
    success: true,
    message: result.message,
    data: result
  });
});

usersRouter.delete('/:id', async (c) => {
  const id = c.req.param('id');
  const result = await deleteUser(id);
  
  return c.json({
    success: true,
    message: result.message
  });
});

export default usersRouter;
```

**Step 2: Apply users routes to app**

```typescript
// backend/src/app.ts
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { errorHandler } from './middleware/errorHandler';
import authRouter from './routes/auth';
import usersRouter from './routes/users';

const app = new Hono();

// Error handler
app.use('/*', errorHandler);

// CORS middleware
app.use('/*', cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
}));

// Routes
app.route('/api/auth', authRouter);
app.route('/api/users', usersRouter);

// Health check
app.get('/health', (c) => {
  return c.json({ status: 'ok', timestamp: new Date().toISOString() });
});

export default app;
```

**Step 3: Commit**

```bash
git add backend/src/routes/users.ts
git commit -m "feat: add users CRUD routes"
```

---

### Task 14: Create Admin Dashboard Endpoints

**Files:**
- Create: `backend/src/routes/admin.ts`
- Create: `backend/src/services/adminService.ts`
- Modify: `backend/src/app.ts`

**Step 1: Create admin service**

```typescript
// backend/src/services/adminService.ts
import { db } from '@/db';
import { users } from '@/db/schema';
import { eq, and, isNull } from 'drizzle-orm';

export async function getDashboardStats() {
  const totalUsers = await db.select().from(users)
    .where(eq(users.isDeleted, false));

  const activeUsers = await db.select().from(users)
    .where(and(
      eq(users.isDeleted, false),
      eq(users.isActive, true)
    ));

  const staffUsers = await db.select().from(users)
    .where(and(
      eq(users.isDeleted, false),
      eq(users.role, 'staff')
    ));

  const adminUsers = await db.select().from(users)
    .where(and(
      eq(users.isDeleted, false),
      eq(users.role, 'admin')
    ));

  return {
    totalUsers: totalUsers.length,
    activeUsers: activeUsers.length,
    staffCount: staffUsers.length,
    adminCount: adminUsers.length,
  };
}

export async function changeUserRole(userId: string, role: string) {
  const [updatedUser] = await db.update(users)
    .set({ role })
    .where(eq(users.id, userId))
    .returning();

  return updatedUser;
}

export async function setUserStatus(userId: string, isActive: boolean) {
  const [updatedUser] = await db.update(users)
    .set({ isActive })
    .where(eq(users.id, userId))
    .returning();

  return updatedUser;
}
```

**Step 2: Create admin routes**

```typescript
// backend/src/routes/admin.ts
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';
import { getDashboardStats, changeUserRole, setUserStatus } from '@/services/adminService';

const adminRouter = new Hono();

adminRouter.get('/dashboard', async (c) => {
  const stats = await getDashboardStats();
  
  return c.json({
    success: true,
    data: stats
  });
});

adminRouter.put('/users/:id/role', zValidator('json', z.object({
  role: z.enum(['admin', 'staff', 'customer'])
})), async (c) => {
  const id = c.req.param('id');
  const { role } = c.req.valid('json');
  
  const updatedUser = await changeUserRole(id, role);
  
  return c.json({
    success: true,
    data: updatedUser
  });
});

adminRouter.put('/users/:id/status', zValidator('json', z.object({
  isActive: z.boolean()
})), async (c) => {
  const id = c.req.param('id');
  const { isActive } = c.req.valid('json');
  
  const updatedUser = await setUserStatus(id, isActive);
  
  return c.json({
    success: true,
    data: updatedUser
  });
});

export default adminRouter;
```

**Step 3: Apply admin routes to app**

```typescript
// backend/src/app.ts
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { errorHandler } from './middleware/errorHandler';
import authRouter from './routes/auth';
import usersRouter from './routes/users';
import adminRouter from './routes/admin';

const app = new Hono();

// Error handler
app.use('/*', errorHandler);

// CORS middleware
app.use('/*', cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
}));

// Routes
app.route('/api/auth', authRouter);
app.route('/api/users', usersRouter);
app.route('/api/admin', adminRouter);

// Health check
app.get('/health', (c) => {
  return c.json({ status: 'ok', timestamp: new Date().toISOString() });
});

export default app;
```

**Step 4: Commit**

```bash
git add backend/src/routes/admin.ts backend/src/services/adminService.ts
git commit -m "feat: add admin dashboard endpoints"
```

---

## [Note: Plan continues in next section... Due to length constraints, this is Part 1 of 3]

---

**Current Progress:**
- ✅ Day 1: Setup & Backend Foundation (Tasks 1-4)
- ✅ Day 2: Backend Core Features (Tasks 5-10)
- ✅ Day 3 (Part 1): Backend Admin & Start Frontend Auth (Tasks 11-14)

**Next Sections in Implementation Plan:**
- Day 3 (Part 2): Frontend Setup & Authentication Pages
- Day 4: Frontend Dashboard & User Management
- Day 5: Integration, Testing, Deployment
