# Architecture Overview

## High-Level Architecture

Sage.ai follows a **Layered + Domain Architecture (Clean Lite)** pattern with complete backend/frontend separation.

```
┌─────────────────────────────────────────────────────────────┐
│                     React SPA Frontend                       │
│         (Vite 5 + React 18.3 + TypeScript)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Main App   │  │   Landing    │  │ Bitcoin      │      │
│  │              │  │    Page      │  │  Fallen      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                           ▲
                           │ REST API + SSE
                           │
┌─────────────────────────────────────────────────────────────┐
│              Nest.js Backend (Layered + Domain)              │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Presentation Layer                     │    │
│  │  (Controllers, DTOs, Guards, Swagger Decorators)   │    │
│  └────────────────────────────────────────────────────┘    │
│                           ▲                                 │
│                           │                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Application Layer                      │    │
│  │  (Use Cases, Services, Business Logic)             │    │
│  └────────────────────────────────────────────────────┘    │
│                           ▲                                 │
│                           │                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Domain Layer                           │    │
│  │  (Entities, Value Objects, Interfaces)             │    │
│  └────────────────────────────────────────────────────┘    │
│                           ▲                                 │
│                           │                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Infrastructure Layer                   │    │
│  │  (Prisma, Redis, Claude SDK, External APIs)        │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                           ▲
                           │
┌─────────────────────────────────────────────────────────────┐
│                    External Services                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Claude   │  │CoinGecko │  │ Discord  │  │  Redis   │   │
│  │   API    │  │   API    │  │ Webhook  │  │  Cache   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Key Architectural Decisions

### 1. Complete Backend/Frontend Separation

**Why**:
- Independent scaling
- Technology flexibility
- Clear API boundaries
- Better development workflow

**Trade-offs**:
- Separate deployment pipelines
- Type synchronization via OpenAPI
- CORS configuration needed

### 2. Layered + Domain Architecture (Clean Lite)

**Why**:
- Domain logic independence
- Easy testing (mock adapters)
- Technology replacement flexibility
- Clear separation of concerns
- Simple and pragmatic for MVP speed

**Structure**:
```
src/
├── modules/                 # Feature modules
│   ├── auth/
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── auth.module.ts
│   │   ├── dto/
│   │   └── guards/
│   ├── chat/
│   ├── ai-agents/
│   ├── market/
│   ├── portfolio/
│   ├── notifications/
│   ├── scheduler/
│   └── jobs/
├── common/                  # Shared utilities
│   ├── decorators/
│   ├── filters/
│   ├── interceptors/
│   └── pipes/
├── prisma/                  # Database
│   ├── schema.prisma
│   └── migrations/
└── config/                  # Configuration
```

### 3. Nest.js + Prisma (not Django or FastAPI)

**Why Nest.js**:
- TypeScript native: Full type safety across stack
- Modular architecture: Easy to scale to microservices
- Built-in SSE: Streaming support for Claude responses
- Decorator-based: Clean, readable code
- Enterprise-ready: Dependency injection, testing utilities

**Why Prisma**:
- Type-safe queries: Auto-generated types from schema
- Intuitive migrations: Simple CLI commands
- Great DX: Prisma Studio for debugging
- Performance: Optimized query engine

**Trade-offs**:
- Learning curve for developers new to Nest.js
- Prisma cold starts (mitigated by connection pooling)

### 4. BullMQ + @nestjs/schedule (not Celery)

**Why**:
- Native TypeScript: No Python runtime needed
- Nest.js integration: First-class support
- Redis-backed: Reliable job persistence
- Built-in retry: Automatic job retry with backoff

**Usage**:
- 15-minute market analysis (@nestjs/schedule)
- Memory extraction jobs (BullMQ)
- Alert distribution (BullMQ)

## Multi-Agent AI System

```
[User Question]
      ↓
[Manager Agent] ─── Intent classification (Haiku 4)
      ↓
[Analyst Agent] ─── Fetch facts (price, news) (Haiku 4)
      ↓
[Persona Agent] ─── Warren Buffett interpretation (Sonnet 4)
      ↓
[Risk Agent] ────── Validate & fact-check (Haiku 4)
      ↓
[Final Response]
```

**Goal**: <1% hallucination rate through multi-agent cross-validation

## Data Flow

### Chat Flow
```
User Input → Nest.js Controller → Chat Service → Claude Multi-Agent → SSE Stream → React UI
                    ↓
             Prisma Repository → PostgreSQL
                    ↓
             User Profile Update (inferred from conversation)
```

### Shadow Portfolio Flow
```
Claude Response → Signal Extraction → Portfolio Service → Prisma Repository
                                              ↓
                                        PostgreSQL + Redis Cache
                                              ↓
                                    ROI Calculation (daily cron)
```

### Proactive Analysis Flow
```
@nestjs/schedule (15분) → Market Analysis Job → CoinGecko + CryptoPanic + Fear & Greed
                                  ↓
                       Context Generation + Threshold Check
                                  ↓
                       Discord Webhook + PWA Push (if alert needed)
```

## Security & Compliance

### Authentication
- Auth.js (@auth/core): Google OAuth 2.0
- JWT-based auth: Access Token (1h) + Refresh Token (30d)
- CSRF protection: SameSite cookies

### Audit Log
- Separate `audit_log` table (append-only)
- Tracks: user actions, data changes, API calls
- Fields: table_name, record_id, action, old_values, new_values, user_id, timestamp

### API Security
- Nest.js Guards: @UseGuards(AuthGuard)
- Rate limiting: @nestjs/throttler
- CORS: @nestjs/cors (whitelist only)
- Helmet: Security headers

## Performance Targets

| Metric | Target |
|--------|--------|
| Chat response (first token) | <2 seconds |
| Context load (20 messages) | <0.5 seconds |
| Proactive analysis interval | 15 minutes |
| Concurrent users (MVP) | 1,000 |
| Hallucination rate | <1% |

## Infrastructure

### Development Environment
- Nest.js dev server (watch mode)
- PostgreSQL local (Docker)
- Redis local (Docker)
- Hot reload for frontend (Vite)

### Production (AWS)
- **Backend**: ECS Fargate (Docker)
  - Nest.js app container
  - BullMQ worker container
- **Database**: RDS PostgreSQL 16 (db.t3.micro)
- **Cache**: ElastiCache Redis 7.x (cache.t3.micro)
- **Frontend**: S3 + CloudFront (3 SPAs)
- **CI/CD**: GitHub Actions

### Auto-Scaling Configuration
- ECS auto-scaling: min 2, max 10 tasks
- CPU target tracking: 70%
- Scale-out cooldown: 60s
- Scale-in cooldown: 300s

### Deployment Strategy
- **dev branch** → dev environment
- **main branch** → production environment
- No staging (rapid iteration for MVP)

---

**Last Updated**: 2025년 12월 23일
**Architecture Version**: 4.0 - Nest.js + Prisma + Layered Domain
