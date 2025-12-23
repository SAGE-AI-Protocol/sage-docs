# MVP Timeline (12 Weeks)

## Overview

**Goal**: Launch beta with 3 core features in 12 weeks
- Chat with Wallet Buffett AI
- Shadow Portfolio tracking
- Proactive market analysis (15-minute intervals)

**Tech Stack**: Nest.js 10.x + Prisma 5.x + React 18.3 + Vite 5

---

## Month 1 (Week 1-4): Foundation

### Week 1-2: Infrastructure

#### AWS Setup
- [ ] AWS account setup + Terraform modules
- [ ] ECS Fargate cluster (dev + prod)
- [ ] RDS PostgreSQL 16 (db.t3.micro)
- [ ] ElastiCache Redis 7.x (cache.t3.micro)
- [ ] S3 buckets + CloudFront distributions

#### Backend Foundation
- [ ] Nest.js 10.x project setup
- [ ] Layered + Domain folder structure
- [ ] Prisma 5.x schema (User, Chat, Message, Profile)
- [ ] Prisma migrations setup
- [ ] Auth.js (@auth/core) Google OAuth
- [ ] @nestjs/swagger configuration

#### Frontend Foundation
- [ ] Vite 5 + React 18.3 projects (3 apps)
- [ ] Tailwind CSS + shadcn/ui
- [ ] TanStack Query 5.x setup
- [ ] Zustand 4.x store setup

### Week 3-4: Hook Sites + Basic Backend

#### Sites
- [ ] WhyBitcoinFallen.com (Vite + React)
- [ ] Sage.ai landing page
- [ ] Email collection + waitlist

#### Backend Core
- [ ] Module structure (auth, chat, market, portfolio)
- [ ] Common utilities (guards, interceptors, filters)
- [ ] Error handling + logging
- [ ] Health check endpoints

---

## Month 2 (Week 5-8): Core Features

### Week 5-6: AI Chat Integration

#### AI Integration
- [ ] @anthropic-ai/sdk setup
- [ ] Multi-agent orchestration
  - [ ] Manager Agent (intent routing - Haiku 4)
  - [ ] Analyst Agent (fact gathering - Haiku 4)
  - [ ] Persona Agent (Buffett philosophy - Sonnet 4)
  - [ ] Risk Agent (validation - Haiku 4)
- [ ] System prompts (Wallet Buffett persona)
- [ ] Tool definitions (market data, calculations)

#### Backend Chat
- [ ] Chat service (business logic)
- [ ] Message repository (Prisma)
- [ ] SSE streaming (@nestjs/sse)
- [ ] Context management (20-message window)
- [ ] User profile inference (from conversation)

#### External APIs
- [ ] CoinGecko adapter (price data)
- [ ] Alternative.me adapter (Fear & Greed)
- [ ] Redis caching (5/10/30 min TTL)

### Week 7-8: Frontend Chat + Portfolio Backend

#### Frontend Chat
- [ ] Chat UI (message bubbles, input)
- [ ] SSE client (EventSource)
- [ ] Streaming message display
- [ ] Markdown rendering
- [ ] Code syntax highlighting

#### Backend Portfolio
- [ ] Shadow trade repository (Prisma)
- [ ] Portfolio service (ROI calculation)
- [ ] Signal extraction from Claude responses
- [ ] Daily ROI cron (@nestjs/schedule)
- [ ] Benchmark comparison (BTC, ETH)

#### Testing
- [ ] Unit tests (domain layer)
- [ ] Integration tests (API endpoints)
- [ ] Hallucination rate measurement

---

## Month 3 (Week 9-12): Polish & Launch

### Week 9-10: Portfolio UI + Proactive Analysis

#### Frontend Portfolio
- [ ] Portfolio page
- [ ] Performance chart (Recharts)
- [ ] [담아보기] button in chat
- [ ] Trade history table
- [ ] ROI vs benchmark display

#### Background Jobs
- [ ] BullMQ 5.x + Redis configuration
- [ ] @nestjs/schedule for 15-minute cron
- [ ] BullMQ worker for async jobs

#### Market Analysis
- [ ] 15-minute market analysis job
- [ ] Context generation (volatility, sentiment, news)
- [ ] Alert threshold logic
- [ ] Discord webhook integration
- [ ] PWA push notification integration

### Week 11-12: Deep Linking + Deployment

#### Deep Linking
- [ ] `/chat/new?context=market_alert` routing
- [ ] Pre-filled context in chat input
- [ ] Analytics tracking

#### Deployment
- [ ] Docker multi-stage builds
- [ ] ECS task definitions (Nest.js + BullMQ worker)
- [ ] GitHub Actions CI/CD
- [ ] Environment variables management (AWS Secrets Manager)
- [ ] Health checks
- [ ] Monitoring (CloudWatch + Sentry)

#### Testing & QA
- [ ] Integration tests (full flow)
- [ ] Performance tests (2-second response target)
- [ ] Security audit (OWASP top 10)
- [ ] Load testing (1,000 concurrent users)

#### Beta Launch
- [ ] 10-20 beta testers recruited
- [ ] Feedback collection system
- [ ] Bug tracking (GitHub Issues)
- [ ] Documentation finalization

---

## Success Criteria (Week 12)

| Metric | Target |
|--------|--------|
| Beta testers | 10-20 users |
| WhyBitcoinFallen visitors | 1,000+/day |
| Sage.ai MAU | 500+ |
| Hallucination rate | <1% |
| Chat response time | <2 seconds |
| Shadow trades tracked | 10+ trades |
| System uptime | 99%+ |

---

## Post-MVP (Phase 2)

### Q1 2026
- Multi-chat support
- Multilingual (Korean, English, Japanese, Chinese)
- Advanced portfolio features (custom benchmarks)
- Community features (Discord integration)

### Q2 2026
- Paid subscription tiers (Pro $19.99, Premium $49.99)
- Advanced analytics
- Mobile app (React Native)
- API for developers

---

**Last Updated**: 2025년 12월 23일
**Timeline**: 12 weeks (3 months) → Beta launch
**Stack**: Nest.js 10.x + Prisma 5.x + React 18.3 + TypeScript
