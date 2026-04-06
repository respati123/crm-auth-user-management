# CRM - Authentication & User Management

CRM Application dengan fitur Authentication dan User Management sebagai MVP.

## Tech Stack

### Backend
- **Runtime:** Bun
- **Framework:** Hono
- **Database:** PostgreSQL
- **ORM:** Drizzle ORM
- **Cache:** Redis
- **Authentication:** JWT + Refresh Tokens
- **Validation:** Zod
- **Testing:** Bun Test

### Frontend
- **Framework:** Next.js 14 (App Router)
- **Language:** TypeScript
- **Styling:** Tailwind CSS
- **State Management:** Custom Hooks (Functional MVVM)
- **Testing:** Vitest + React Testing Library

### DevOps
- **Containerization:** Docker
- **Orchestration:** Docker Compose
- **Reverse Proxy:** Nginx
- **CI/CD:** GitHub Actions

## Getting Started

### Prerequisites
- Docker & Docker Compose
- Bun (v1+) for backend
- Node.js (v18+) for frontend

### Clone Repository
```bash
git clone https://github.com/respati123/crm-auth-user-management.git
cd crm-auth-user-management
```

### Setup Environment Variables

**Backend (.env):**
```bash
NODE_ENV=development
PORT=3001
DATABASE_URL=postgresql://localhost:5432/crm_db
REDIS_URL=redis://localhost:6379
JWT_SECRET=your-secret-min-32-chars
REFRESH_TOKEN_SECRET=your-secret-min-32-chars
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
```

**Frontend (.env.local):**
```bash
NEXT_PUBLIC_API_URL=http://localhost:3001
```

### Start Services

**Docker (PostgreSQL + Redis):**
```bash
docker-compose up -d
```

**Backend:**
```bash
cd backend
bun install
bun run dev
```

**Frontend:**
```bash
cd frontend
npm install
npm run dev
```

## Project Structure

```
crm-auth-user-management/
├── backend/              # Bun + Hono API
├── frontend/             # Next.js 14 App
├── nginx/                # Nginx config
├── docker-compose.yml     # Docker services
└── .github/             # CI/CD workflows
```

## Features (MVP)

- ✅ User Authentication (Login, Register, Logout)
- ✅ User Management (CRUD operations)
- ✅ Role-Based Access Control (Admin, Staff, Customer)
- ✅ Soft Delete (restore deleted users)
- ✅ Admin Dashboard (basic statistics)
- ✅ Multi-Language Support (Bahasa Indonesia & English)
- ✅ Rate Limiting
- ✅ Input Validation
- ✅ Error Handling

## Implementation Plan

Lihat dokumentasi lengkap di:
`docs/plans/2026-04-06-crm-implementation-plan.md`

## Timeline

**1 Week (Aggressive MVP)**
- Day 1: Setup & Backend Foundation
- Day 2: Backend Core Features
- Day 3: Backend Admin & Frontend Auth
- Day 4: Frontend Dashboard
- Day 5: Integration, Testing, Deployment

## License

MIT
