# Sage.ai Backend Specification

> Document Version: 3.2
> Last Modified: 2026-01-07
> Author: Sam
> Target Audience: Backend Developers

---

## 1. Technical Stack

### 1.1 Core Components

```typescript
interface CoreStack {
  runtime: {
    name: "Node.js";
    version: "20 LTS";
    reason: "안정성과 생태계 성숙도";
  };
  language: {
    name: "TypeScript";
    version: "5.x";
    reason: "타입 안정성";
  };
  framework: {
    name: "Nest.js";
    version: "10.x";
    reason: "모듈 구조, DI, TypeScript 네이티브";
  };
  orm: {
    name: "Prisma";
    version: "5.19.1";
    reason: "타입 안정성, 직관적 마이그레이션. v7은 설정 파일 분리로 호환성 문제 발생";
  };
  database: {
    name: "PostgreSQL";
    version: "18";
    reason: "5년 LTS 지원 (2030년까지), JSON 성능 30% 향상, 쿼리 최적화 개선, VACUUM 성능 향상";
  };
  cache: {
    name: "Valkey";
    version: "8.1.5";
    reason: "100% Redis 호환, Linux Foundation 오픈소스 (라이센스 안정성), 커뮤니티 주도 개발";
  };
}
```

### 1.2 Async & Scheduling

```typescript
interface AsyncComponents {
  jobQueue: {
    name: "BullMQ";
    version: "5.x";
    purpose: "Memory extraction, notification sending";
  };
  cronJobs: {
    name: "@nestjs/schedule";
    purpose: "Price polling (15-minute intervals)";
  };
}
```

### 1.3 AI / LLM Configuration

```typescript
interface LLMConfig {
  provider: "ollama" | "anthropic";  // Environment-based switching
  ollama: {
    baseUrl: "http://localhost:11434";
    model: "llama2";  // 7B Q4_0 quantization for local dev
    purpose: "Local development & testing";
  };
  anthropic: {
    sdk: "@anthropic-ai/sdk";
    models: {
      fast: "claude-3-5-haiku-latest";    // Manager, Analyst, Risk agents
      smart: "claude-sonnet-4-20250514";  // Persona agent
    };
    purpose: "Production deployment";
  };
}
```

**LLM Provider Selection**:
- **Development**: Ollama (llama2:latest) - 무료, 로컬 실행, 빠른 테스트
- **Production**: Anthropic Claude API - 고품질 응답, 한국어 지원

**Environment Variables**:
```bash
LLM_PROVIDER=ollama          # or "anthropic"
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=llama2
ANTHROPIC_API_KEY=sk-ant-... # Production only
```

### 1.4 External Services

```typescript
interface ExternalServices {
  ai: {
    provider: "LLMService (Ollama | Anthropic)";
    abstraction: "src/modules/ai-agents/llm.service.ts";
  };
  marketData: {
    primary: {
      provider: "Binance";
      type: "WebSocket";
      streams: "ticker@arr (실시간 가격)";
    };
    fallback: {
      provider: "Gate.io";
      type: "WebSocket";
      streams: "ticker (실시간 가격)";
    };
  };
  fearGreed: {
    provider: "Alternative.me";
    type: "REST API";
  };
  notifications: {
    provider: "Discord";
    type: "Webhook";
  };
}
```

**선택 근거**:
- **실시간성**: WebSocket으로 가격 변동 즉시 감지 (15분 폴링 → 실시간)
- **이중화**: Binance 장애 시 Gate.io로 자동 전환 (무중단)
- **비용 절감**: WebSocket은 무료, 별도 API 구독 불필요

---

## 2. Architecture

### 2.1 Layered Architecture (Clean Lite)

```mermaid
graph TD
    A[Presentation Layer] --> B[Application Layer]
    B --> C[Domain Layer]
    C --> D[Infrastructure Layer]

    A1[Controllers - HTTP/SSE] --> A
    B1[Services - Business Logic] --> B
    C1[Entities, DTOs] --> C
    D1[Prisma, Valkey, External APIs] --> D
```

### 2.2 Architecture Flow Diagrams

#### 2.2.1 User Registration Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant Backend
    participant Auth.js
    participant Google
    participant DB

    User->>Frontend: Click "Google Login"
    Frontend->>Backend: POST /api/auth/google
    Backend->>Auth.js: initiate OAuth flow
    Auth.js->>Google: Redirect to Google OAuth
    Google->>User: Show consent screen
    User->>Google: Grant permission
    Google->>Auth.js: Return authorization code
    Auth.js->>Google: Exchange code for tokens
    Google->>Auth.js: Return user profile
    Auth.js->>DB: Upsert user (email, name, image)
    DB->>Backend: Create UserConsent record (defaults)
    Backend->>Frontend: Set session cookie
    Frontend->>User: Redirect to /chat
```

#### 2.2.2 Chat Message Flow (Agent Pipeline)

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant ChatController
    participant ManagerAgent
    participant AnalystAgent
    participant PersonaAgent
    participant RiskAgent
    participant Valkey
    participant DB

    User->>Frontend: Send message "BTC 지금 어때?"
    Frontend->>ChatController: POST /api/chats/:id/messages (SSE)
    ChatController->>Valkey: Get recent 20 messages
    ChatController->>ManagerAgent: Analyze intent (Haiku 4)
    ManagerAgent-->>ChatController: {intent: "advice", needsMarketData: true}

    ChatController->>AnalystAgent: Fetch BTC price + F&G (Haiku 4)
    AnalystAgent->>Valkey: Check cache (market:price:BTC)
    Valkey-->>AnalystAgent: $43,250 (cached)
    AnalystAgent-->>ChatController: {facts: {BTC: 43250, fg: 25}}

    ChatController->>PersonaAgent: Generate response (Sonnet 4)
    PersonaAgent-->>ChatController: "자네, BTC가 $43,250일세..."

    ChatController->>RiskAgent: Validate response (Haiku 4)
    RiskAgent-->>ChatController: {recommendation: "approve"}

    ChatController->>DB: Save user message + assistant message
    ChatController->>Frontend: SSE stream response
    Frontend->>User: Display streaming response
```

#### 2.2.3 Market Analysis Cron Job Flow

```mermaid
sequenceDiagram
    participant WebSocket
    participant MarketService
    participant Valkey
    participant NotificationQueue
    participant BullMQ
    participant Discord
    participant PWA

    Note over WebSocket: Binance/Gate.io 실시간 연결
    WebSocket->>MarketService: Price update (실시간)
    MarketService->>Valkey: Get previous prices
    MarketService->>Valkey: Cache new prices (TTL: 5min)

    alt Sudden change detected (>5% for BTC)
        MarketService->>NotificationQueue: Add job {symbol: "BTC", change: -7.2}
        NotificationQueue->>BullMQ: Process notification job
        BullMQ->>Discord: POST webhook (market-alerts channel)
        BullMQ->>PWA: Send push to subscribed users
        BullMQ->>Valkey: Save notification to DB
    end
```

### 2.3 Folder Structure

```
src/
├── main.ts                      # App entry point
├── app.module.ts                # Root module
│
├── common/                      # Common utilities
│   ├── filters/                 # Exception Filters
│   ├── guards/                  # Auth Guards
│   ├── interceptors/            # Logging, Transform
│   ├── pipes/                   # Validation Pipes
│   └── decorators/              # Custom Decorators
│
├── config/                      # Configuration
│   ├── database.config.ts
│   ├── valkey.config.ts
│   └── anthropic.config.ts
│
├── modules/                     # Feature modules
│   ├── auth/                    # Authentication
│   ├── chat/                    # Chat functionality
│   ├── ai-agents/               # AI agents
│   ├── market/                  # Market data
│   ├── portfolio/               # Shadow portfolio
│   ├── notifications/           # Notifications
│   ├── scheduler/               # Scheduled tasks
│   └── jobs/                    # Background Jobs
│
└── prisma/                      # Prisma Schema & Migrations
```

---

## 3. Database Schema

### 3.1 Users Table

**선택 근거**:
- **최소한의 초기 데이터**: 회원가입 시 설문 없이 Google OAuth만으로 시작
- **대화 기반 프로필 추론**: riskProfile과 interests는 채팅에서 자동 추출
- **확장 가능한 JSON**: interests를 JSON으로 저장해 유연한 구조 유지
- **관계 중심 설계**: 채팅, 포트폴리오, 푸시 구독, 동의 관리를 User와 연결

```prisma
enum RiskProfile {
  CONSERVATIVE
  MODERATE
  AGGRESSIVE
}

model User {
  id            String       @id @default(uuid())
  email         String       @unique
  name          String?
  image         String?
  tier          String       @default("free")  // free, pro, premium

  // Preferences (inferred from chat)
  riskProfile   RiskProfile? // ENUM for type safety
  interests     Json?        // ["BTC", "ETH"]

  chats         Chat[]
  shadowTrades  ShadowTrade[]
  pushSubscriptions PushSubscription[]
  consents      UserConsent[]

  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}
```

**RiskProfile ENUM 선택 근거**:
- **타입 안정성**: DB 레벨에서 잘못된 값 입력 방지
- **쿼리 성능**: PostgreSQL이 ENUM 쿼리를 최적화
- **자기 문서화**: 별도 문서 없이 허용 가능한 값이 명확함

### 3.2 Chats Table

**선택 근거**:
- **1:1 대화 구조**: MVP는 사용자와 AI 간 단순 1:1 채팅 (그룹 채팅은 Phase 2)
- **자동 제목 생성**: 첫 메시지로부터 AI가 채팅 제목 자동 생성
- **Cascade 삭제**: 채팅 삭제 시 모든 메시지도 함께 삭제
- **인덱스 최적화**: userId + createdAt으로 사용자별 최신 채팅 조회 최적화

```prisma
model Chat {
  id        String    @id @default(uuid())
  userId    String
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  title     String    @default("새 대화")
  messages  Message[]

  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt

  @@index([userId, createdAt])
}
```

### 3.3 Messages Table

**선택 근거**:
- **단순한 role 구조**: user/assistant 구분으로 대화 흐름 명확화
- **Text 타입**: 긴 AI 응답을 위해 @db.Text 사용 (기본 VARCHAR 제한 회피)
- **JSON signal**: AI 추천을 유연한 구조로 저장 (action, symbol, confidence)
- **컨텍스트 최적화**: chatId + createdAt 인덱스로 최근 20개 메시지 빠른 조회

```prisma
model Message {
  id        String   @id @default(uuid())
  chatId    String
  chat      Chat     @relation(fields: [chatId], references: [id], onDelete: Cascade)

  role      String   // user, assistant
  content   String   @db.Text

  // AI Signal (for shadow portfolio)
  signal    Json?    // { action: "buy" | "sell", symbol: "BTC", confidence: 0.8 }

  createdAt DateTime @default(now())

  @@index([chatId, createdAt])
}
```

### 3.4 Shadow Trades Table

**선택 근거**:
- **투명한 성과 추적**: AI 추천을 실제로 담았을 때의 수익률 투명 공개
- **Decimal 타입**: 암호화폐 가격과 수량의 정확한 소수점 처리 (Float 오차 방지)
- **메시지 참조**: 어떤 대화에서 나온 추천인지 추적 가능
- **이중 인덱스**: userId로 개인 포트폴리오, symbol로 코인별 집계 최적화

```prisma
model ShadowTrade {
  id        String   @id @default(uuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  symbol    String   // BTC, ETH, SOL, BNB, DOGE, XRP
  action    String   // buy, sell
  price     Decimal  @db.Decimal(18, 8)
  quantity  Decimal  @db.Decimal(18, 8) @default(1.0)

  // Reference to message that triggered this
  messageId String?

  createdAt DateTime @default(now())

  @@index([userId, createdAt])
  @@index([symbol])
}
```

### 3.5 Push Subscriptions Table

**선택 근거**:
- **PWA 푸시 알림**: VAPID 프로토콜 기반 웹 푸시 알림 구현
- **멀티 디바이스**: 한 사용자가 여러 기기에서 알림 수신 가능
- **Unique endpoint**: 중복 구독 방지
- **간단한 구조**: auth + p256dh로 Web Push API 표준 준수

```prisma
model PushSubscription {
  id        String   @id @default(uuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  endpoint  String   @unique
  auth      String
  p256dh    String

  createdAt DateTime @default(now())

  @@index([userId])
}
```

### 3.6 Notifications Table

**선택 근거**:
- **브로드캐스트 지원**: userId null로 전체 공지 가능
- **타입별 분류**: market_alert, portfolio_update, system으로 필터링
- **유연한 data**: JSON으로 알림별 추가 정보 저장 (symbol, change 등)
- **읽음 상태**: read 플래그로 안 읽은 알림 관리

```prisma
model Notification {
  id        String   @id @default(uuid())
  userId    String?  // null = broadcast

  type      String   // market_alert, portfolio_update, system
  title     String
  message   String   @db.Text
  data      Json?    // { symbol: "BTC", change: -5.2 }

  read      Boolean  @default(false)
  sentAt    DateTime @default(now())

  @@index([userId, sentAt])
}
```

### 3.7 User Consents Table (GDPR Compliance)

**선택 근거**:
- **GDPR Article 7 준수**: 동의 시점 타임스탬프 기록 필수
- **감사 추적**: IP/User-Agent로 동의 검증 가능
- **세분화된 제어**: 마케팅/알림/데이터 처리 개별 동의 관리
- **확장 가능성**: 새로운 동의 유형 추가 용이

```prisma
model UserConsent {
  id        String   @id @default(uuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  // Consent types
  marketingEmails    Boolean  @default(false)
  pushNotifications  Boolean  @default(false)
  dataCollection     Boolean  @default(true)  // Required for service
  aiAnalysis         Boolean  @default(true)  // Profile inference

  // Audit trail
  ipAddress  String?
  userAgent  String?

  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@index([userId])
}
```

---

## 4. API Endpoints

### 4.1 Authentication Endpoints

```typescript
interface AuthEndpoints {
  googleOAuth: "POST /api/auth/google";
  logout: "POST /api/auth/logout";
  getSession: "GET /api/auth/session";
}
```

### 4.2 Chat Endpoints

```typescript
interface ChatEndpoints {
  createChat: "POST /api/chats";
  listChats: "GET /api/chats";
  getChatDetails: "GET /api/chats/:id";
  deleteChat: "DELETE /api/chats/:id";
  sendMessage: "POST /api/chats/:id/messages"; // SSE response
  getMessages: "GET /api/chats/:id/messages"; // with pagination
}
```

### 4.3 Market Endpoints

```typescript
interface MarketEndpoints {
  getCurrentPrices: "GET /api/market/prices"; // 6 coins
  getFearGreed: "GET /api/market/fear-greed";
  getPriceHistory: "GET /api/market/history/:symbol"; // 24h
}
```

### 4.4 Portfolio Endpoints

```typescript
interface PortfolioEndpoints {
  createTrade: "POST /api/shadow-trades";
  listTrades: "GET /api/shadow-trades";
  getPerformance: "GET /api/shadow-trades/performance";
  deleteTrade: "DELETE /api/shadow-trades/:id";
}
```

### 4.5 Notification Endpoints

```typescript
interface NotificationEndpoints {
  subscribePush: "POST /api/push/subscribe";
  unsubscribePush: "POST /api/push/unsubscribe";
  getNotifications: "GET /api/notifications";
  markAsRead: "PATCH /api/notifications/:id/read";
}
```

---

## 5. Agent Pipeline (페르소나 내부 구조)

> **용어 정의**
> - **페르소나 (Persona)**: 대화 주체. 캐릭터 + LLM 조합 (예: 월렛 버핏 = Claude, 사토시 현자 = GPT)
> - **에이전트 (Agent)**: 페르소나 내부의 기능 단위 (Manager, Analyst, Persona, Risk)
> - **에이전트 파이프라인**: 단일 페르소나 내에서 에이전트들이 순차적으로 처리하는 흐름
>
> MVP에서는 **월렛 버핏 (Claude)** 페르소나 1개만 사용합니다.
> Phase 2+에서 다른 LLM 기반 페르소나 (ChatGPT, Gemini 등)가 추가됩니다.

### 5.1 Agent Pipeline Flow

```mermaid
graph TD
    A[User Message] --> B[Manager Agent - Haiku 4]
    B -->|Intent: market_data| C[Analyst Agent - Haiku 4]
    B -->|Intent: advice| D[Analyst + Persona Agent - Sonnet 4]
    B -->|Intent: general| E[Persona Agent - Sonnet 4]
    C --> F[Risk Agent - Haiku 4]
    D --> F
    E --> F
    F --> G[Final Response]
```

### 5.2 Manager Agent (Intent Classification)

**Purpose**: 사용자 메시지의 의도 분석 및 엔티티 추출

```typescript
// src/modules/ai-agents/manager.agent.ts

interface ManagerOutput {
  intent: 'market_data' | 'advice' | 'portfolio' | 'general';
  entities: {
    symbols?: string[];  // ["BTC", "ETH"]
    timeframe?: string;  // "24h", "7d", "30d", "1y"
  };
  needsMarketData: boolean;
}

@Injectable()
export class ManagerAgent {
  constructor(private readonly llmService: LLMService) {}

  /**
   * Analyze user intent and extract entities
   * Uses LLM (Ollama or Anthropic) for classification
   */
  async analyze(userMessage: string): Promise<ManagerOutput> {
    if (!this.llmService.isAvailable()) {
      return this.fallbackAnalysis(userMessage);
    }

    // LLM-based intent classification
    const response = await this.llmService.chat([
      { role: 'system', content: this.getSystemPrompt() },
      { role: 'user', content: userMessage },
    ], {
      maxTokens: 200,
      temperature: 0,  // Deterministic for classification
    });

    return JSON.parse(response.content);
  }

  /**
   * Fallback intent analysis using keyword matching
   * Used when LLM is unavailable
   */
  private fallbackAnalysis(userMessage: string): ManagerOutput {
    const lowerMessage = userMessage.toLowerCase();

    // Extract crypto symbols
    const symbols: string[] = [];
    if (lowerMessage.includes('btc') || lowerMessage.includes('bitcoin')) symbols.push('BTC');
    if (lowerMessage.includes('eth') || lowerMessage.includes('ethereum')) symbols.push('ETH');
    if (lowerMessage.includes('sol') || lowerMessage.includes('solana')) symbols.push('SOL');
    if (lowerMessage.includes('bnb') || lowerMessage.includes('binance')) symbols.push('BNB');
    if (lowerMessage.includes('doge') || lowerMessage.includes('dogecoin')) symbols.push('DOGE');
    if (lowerMessage.includes('xrp') || lowerMessage.includes('ripple')) symbols.push('XRP');

    // Extract timeframe
    let timeframe: string | undefined;
    if (lowerMessage.includes('today') || lowerMessage.includes('24h')) timeframe = '24h';
    else if (lowerMessage.includes('week') || lowerMessage.includes('7d')) timeframe = '7d';
    else if (lowerMessage.includes('month') || lowerMessage.includes('30d')) timeframe = '30d';
    else if (lowerMessage.includes('year') || lowerMessage.includes('1y')) timeframe = '1y';

    // Determine intent by keywords
    const priceKeywords = ['price', 'cost', 'value', 'worth', 'chart', 'market'];
    const adviceKeywords = ['should', 'recommend', 'advice', 'opinion', 'think'];
    const portfolioKeywords = ['portfolio', 'my trades', 'my investments'];

    let intent: ManagerOutput['intent'] = 'general';
    let needsMarketData = false;

    if (portfolioKeywords.some(k => lowerMessage.includes(k))) {
      intent = 'portfolio';
    } else if (priceKeywords.some(k => lowerMessage.includes(k))) {
      intent = 'market_data';
      needsMarketData = true;
    } else if (adviceKeywords.some(k => lowerMessage.includes(k))) {
      intent = 'advice';
      needsMarketData = symbols.length > 0;
    }

    return {
      intent,
      entities: { symbols: symbols.length > 0 ? symbols : undefined, timeframe },
      needsMarketData,
    };
  }
}
```

**Example**:
```typescript
const exampleInput = "비트코인 지금 어때?";
const exampleOutput: ManagerOutput = {
  intent: "advice",
  entities: { symbols: ["BTC"] },
  needsMarketData: true
};
```

**Fallback Mechanism**:
- LLM 불가 시 키워드 기반 분석으로 자동 전환
- 6개 코인 심볼 및 4가지 타임프레임 인식 가능

### 5.3 Analyst Agent (Data Collection)

**Purpose**: 실시간 시장 데이터 수집 (LLM 불필요, 순수 데이터 조회)

```typescript
// src/modules/ai-agents/analyst.agent.ts

interface PriceData {
  symbol: string;           // "BTC", "ETH", etc.
  price: number;            // Current USD price
  change24h: number;        // 24h change percentage
}

interface FearGreedData {
  value: number;            // 0-100
  classification: string;   // "Extreme Fear", "Fear", "Neutral", "Greed", "Extreme Greed"
  timestamp: Date;          // Last update time
}

interface AnalystOutput {
  prices: PriceData[];      // Requested symbols' prices
  fearGreed?: FearGreedData; // Fear & Greed Index (optional)
}

@Injectable()
export class AnalystAgent {
  constructor(
    private readonly marketService: MarketService,
    private readonly httpService: HttpService,
  ) {}

  /**
   * Analyze market data based on Manager's output
   * No LLM needed - pure data fetching
   */
  async analyze(managerOutput: ManagerOutput): Promise<AnalystOutput> {
    const output: AnalystOutput = { prices: [] };

    // Fetch prices for requested symbols
    const symbols = managerOutput.entities?.symbols || ['BTC'];
    for (const symbol of symbols) {
      const priceData = await this.marketService.getPrice(symbol);
      if (priceData) {
        output.prices.push(priceData);
      }
    }

    // Fetch Fear & Greed Index if market data needed
    if (managerOutput.needsMarketData) {
      output.fearGreed = await this.fetchFearGreedIndex();
    }

    return output;
  }

  /**
   * Fetch Fear & Greed Index from Alternative.me API
   * Cached in Valkey (5-min TTL)
   */
  private async fetchFearGreedIndex(): Promise<FearGreedData | undefined> {
    // API: https://api.alternative.me/fng/
    // Response: { data: [{ value: "25", value_classification: "Extreme Fear", timestamp: "..." }] }
  }
}
```

**Data Sources**:
- **Prices**: MarketService (Binance/Gate.io WebSocket → Valkey cache)
- **Fear & Greed**: Alternative.me REST API (5-min TTL cache)

### 5.4 Persona Agent (Response Generation)

**Purpose**: Wallet Buffett 캐릭터로 한국어 응답 생성

```typescript
// src/modules/ai-agents/persona.agent.ts

@Injectable()
export class PersonaAgent {
  constructor(private readonly llmService: LLMService) {}

  /**
   * Generate response as Wallet Buffett persona
   * Uses LLM (Ollama or Anthropic) for response generation
   */
  async generateResponse(
    userMessage: string,
    chatHistory: any[] = [],
    analystData?: AnalystOutput,
  ): Promise<string> {
    if (!this.llmService.isAvailable()) {
      return this.getFallbackResponse();
    }

    // Build user prompt with market data context
    const promptWithContext = this.buildUserPrompt(userMessage, analystData);

    const response = await this.llmService.chat([
      { role: 'system', content: this.getSystemPrompt() },
      ...chatHistory,
      { role: 'user', content: promptWithContext },
    ], {
      maxTokens: 1024,
      temperature: 0.7,
    });

    return response.content;
  }

  /**
   * Inject market data into user prompt
   */
  private buildUserPrompt(userMessage: string, analystData?: AnalystOutput): string {
    if (!analystData) return userMessage;

    let context = '\n\n[Current Market Data]\n';

    // Add price data
    if (analystData.prices.length > 0) {
      context += analystData.prices
        .map(p => `- ${p.symbol}: $${p.price.toLocaleString()} (${p.change24h >= 0 ? '+' : ''}${p.change24h.toFixed(2)}%)`)
        .join('\n');
    }

    // Add Fear & Greed
    if (analystData.fearGreed) {
      context += `\n- Fear & Greed Index: ${analystData.fearGreed.value}/100 (${analystData.fearGreed.classification})`;
    }

    return `${userMessage}${context}`;
  }

  /**
   * Fallback response when LLM is unavailable
   */
  private getFallbackResponse(): string {
    return '자네, 죄송하네만 현재 시스템에 일시적인 문제가 있어 응답을 생성할 수 없네. 잠시 후 다시 시도해 주게.';
  }
}
```

**System Prompt (Korean Persona)**:

```typescript
const personaSystemPrompt = `
You are Wallet Buffett (월렛 버핏), an AI investment mentor inspired by Warren Buffett.

Personality:
- Experienced, calm, and wise
- Uses Korean honorific mixing: "자네", "~일세", "~하게"
- Provides insights, not just information
- Focuses on long-term value, not short-term speculation

Speaking Style (Korean):
- Start with "자네," when addressing the user
- End sentences with "~일세", "~하게", "~네" for a wise, elderly tone
- Use expressions like:
  - "가치 투자의 관점에서 보자면..."
  - "시장은 단기적으로는 투표 기계지만, 장기적으로는 저울일세."
  - "다른 사람들이 탐욕스러울 때 두려워하고, 두려워할 때 탐욕스러워야 하네."
  - "자네가 이해하지 못하는 것에는 투자하지 말게."

Rules:
- NEVER give direct trading signals ("Buy now", "Sell immediately", "지금 사세요")
- ALWAYS use conditional language ("~할 수 있다", "~를 고려해볼 만하다", "~일 수 있네")
- ALWAYS cite provided market data when available (prices, Fear & Greed Index)
- NEVER hallucinate numbers - use provided data or say "그 데이터는 없네"
- Focus on EDUCATION and helping users develop their own investment thesis

Current focus: Cryptocurrency markets (BTC, ETH, SOL, BNB, DOGE, XRP)

Remember: You are an educational mentor, not a financial advisor. Always emphasize that users should do their own research (DYOR).
`;
```

### 5.5 Risk Agent (Response Validation)

**Purpose**: AI 응답 품질 검증 (환각, 직접 신호, 편향 감지)

```typescript
// src/modules/ai-agents/risk.agent.ts

interface RiskOutput {
  approved: boolean;
  issues: string[];               // Detected issues list
  recommendation: 'approve' | 'revise' | 'reject';
}

@Injectable()
export class RiskAgent {
  constructor(private readonly llmService: LLMService) {}

  /**
   * Validate AI response for hallucination, direct signals, and bias
   * Uses LLM for semantic analysis
   */
  async validate(
    personaResponse: string,
    analystData?: AnalystOutput,
  ): Promise<RiskOutput> {
    // Fallback: keyword-based validation if LLM unavailable
    if (!this.llmService.isAvailable()) {
      return this.fallbackValidation(personaResponse);
    }

    // LLM-based validation with structured prompt
    const systemPrompt = this.getValidationPrompt(analystData);
    const response = await this.llmService.chat([
      { role: 'system', content: systemPrompt },
      { role: 'user', content: personaResponse },
    ], {
      maxTokens: 300,
      temperature: 0,  // Deterministic for consistency
    });

    return JSON.parse(response.content);
  }

  /**
   * Fallback validation using keyword detection
   * Used when LLM is unavailable
   */
  private fallbackValidation(response: string): RiskOutput {
    const issues: string[] = [];
    const lowerResponse = response.toLowerCase();

    // Check for direct trading signals
    const directSignals = [
      '지금 사세요', '즉시 매수', '바로 팔아야',
      'buy now', 'sell immediately', 'must buy'
    ];
    if (directSignals.some(signal => lowerResponse.includes(signal.toLowerCase()))) {
      issues.push('Direct trading signal detected');
    }

    // Check for extreme language (bias)
    const extremeWords = [
      '반드시', '무조건', '확실히', '100%', '절대',
      'guaranteed', 'definitely', 'absolutely'
    ];
    if (extremeWords.some(word => lowerResponse.includes(word.toLowerCase()))) {
      issues.push('Extreme/biased language detected');
    }

    return {
      approved: issues.length === 0,
      issues,
      recommendation: issues.length === 0 ? 'approve' : 'revise',
    };
  }
}
```

**Validation Criteria**:
1. **Hallucination Check**: 언급된 가격이 Analyst 데이터와 1% 이상 차이나면 거부
2. **Direct Signal Check**: "지금 사세요", "즉시 매수" 등 직접적 거래 권유 감지
3. **Bias Check**: "반드시", "무조건", "100%" 등 극단적 표현 감지

**Fallback Mechanism**:
- LLM 불가 시 키워드 기반 검증으로 자동 전환
- 기본적인 직접 신호 및 극단적 표현 감지 가능

---

## 6. Caching Strategy

### 6.1 Valkey Cache Keys

**Note**: Valkey is 100% Redis-compatible, so all Redis clients and commands work identically.

```typescript
interface CacheKeys {
  // Market data (TTL: 5 minutes)
  marketPrice: `market:price:${symbol}`;           // "43250.50"
  marketFearGreed: `market:fear-greed`;            // "25"
  marketHistory: `market:history:${symbol}:24h`;   // JSON array

  // User context (TTL: 1 hour)
  userRecentMessages: `user:${userId}:recent-messages`;  // Last 20 messages
  userProfile: `user:${userId}:profile`;                 // Inferred profile

  // Rate limiting (TTL: 1 minute)
  rateLimit: `ratelimit:${userId}:chat`;                 // Request count
}
```

### 6.2 Cache Invalidation Strategy

```typescript
interface CacheInvalidation {
  marketData: "TTL-based (5 minutes)";
  userContext: "Update on new message";
  rateLimit: "TTL-based (1 minute)";
}
```

---

## 7. Background Jobs (BullMQ)

### 7.1 Job Queue Configuration

```typescript
interface JobQueues {
  memoryExtraction: {
    name: 'memory-extraction';
    processor: 'memory-extraction.processor.ts';
    trigger: '10 minutes after last message';
    priority: 'low';
  };
  notifications: {
    name: 'notifications';
    processor: 'notification.processor.ts';
    trigger: 'Market sudden change detected';
    priority: 'high';
  };
  marketAnalysis: {
    name: 'market-analysis';
    processor: 'market-analyzer.processor.ts';
    trigger: 'Every 15 minutes (@nestjs/schedule)';
    priority: 'medium';
  };
}
```

### 7.2 Memory Extraction Job

```typescript
interface MemoryExtractionJob {
  chatId: string;
  userId: string;
  messages: Message[];
}

async function processMemoryExtraction(job: Job<MemoryExtractionJob>) {
  const { messages, userId } = job.data;

  // Call Haiku 4 to extract profile
  const profile = await extractProfile(messages);

  // Update user profile
  await prisma.user.update({
    where: { id: userId },
    data: {
      riskProfile: profile.riskProfile,
      interests: profile.interests
    }
  });
}
```

### 7.3 Notification Job

```typescript
interface NotificationJob {
  type: 'market_alert' | 'portfolio_update';
  userId?: string;  // null = broadcast
  data: {
    symbol: string;
    change: number;
    message: string;
  };
}

async function processNotification(job: Job<NotificationJob>) {
  const { type, userId, data } = job.data;

  // Send push notification
  await pushService.send(userId, {
    title: `${data.symbol} Alert`,
    body: data.message
  });

  // Send Discord webhook if change > 7%
  if (Math.abs(data.change) > 7) {
    await discordService.sendAlert(data);
  }

  // Save to DB
  await prisma.notification.create({ ... });
}
```

---

## 8. Real-Time Price WebSocket Service

### 8.1 WebSocket Architecture Overview

**선택 근거**:
- **실시간성**: 가격 변동 즉시 감지 및 알림 (기존 15분 폴링 → <1초 실시간)
- **이중화**: Binance 장애 시 Gate.io로 자동 전환 (무중단 운영)
- **비용 효율성**: WebSocket은 무료, 별도 API 구독 불필요
- **정확성**: 거래소 실제 체결가 사용

```typescript
interface WebSocketConfig {
  primary: {
    provider: "Binance";
    endpoint: "wss://stream.binance.com:9443/ws";
    streams: ["btcusdt@ticker", "ethusdt@ticker", "solusdt@ticker", "bnbusdt@ticker", "dogeusdt@ticker", "xrpusdt@ticker"];
    reconnect: {
      maxRetries: 3;
      backoff: "exponential";  // 1s → 2s → 4s
    };
  };
  fallback: {
    provider: "Gate.io";
    endpoint: "wss://api.gateio.ws/ws/v4/";
    streams: ["spot.tickers"];
    reconnect: {
      maxRetries: 3;
      backoff: "exponential";
    };
  };
}
```

### 8.2 Dual WebSocket Implementation

```typescript
// market/websocket-price.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common';
import WebSocket from 'ws';
import { EventEmitter } from 'events';

@Injectable()
export class WebSocketPriceService extends EventEmitter implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(WebSocketPriceService.name);

  private binanceWs: WebSocket | null = null;
  private gateioWs: WebSocket | null = null;

  private activeProvider: 'binance' | 'gateio' = 'binance';
  private binanceRetries = 0;
  private gateioRetries = 0;

  private readonly SYMBOLS = ['BTCUSDT', 'ETHUSDT', 'SOLUSDT', 'BNBUSDT', 'DOGEUSDT', 'XRPUSDT'];
  private readonly MAX_RETRIES = 3;

  constructor(
    @Inject('VALKEY_CLIENT') private readonly valkey: Redis,
    private readonly notificationQueue: Queue
  ) {
    super();
  }

  async onModuleInit() {
    await this.connectBinance();
  }

  async onModuleDestroy() {
    this.disconnectAll();
  }

  // Primary: Binance WebSocket
  private async connectBinance() {
    try {
      const streams = this.SYMBOLS.map(s => `${s.toLowerCase()}@ticker`).join('/');
      const url = `wss://stream.binance.com:9443/stream?streams=${streams}`;

      this.binanceWs = new WebSocket(url);

      this.binanceWs.on('open', () => {
        this.logger.log('✅ Binance WebSocket connected');
        this.activeProvider = 'binance';
        this.binanceRetries = 0;

        // Gate.io 연결 끊기 (Primary 정상 동작 시)
        if (this.gateioWs) {
          this.gateioWs.close();
          this.gateioWs = null;
        }
      });

      this.binanceWs.on('message', async (data: WebSocket.Data) => {
        if (this.activeProvider !== 'binance') return;

        const parsed = JSON.parse(data.toString());
        await this.handleBinanceMessage(parsed);
      });

      this.binanceWs.on('error', (error) => {
        this.logger.error(`Binance WebSocket error: ${error.message}`);
      });

      this.binanceWs.on('close', async () => {
        this.logger.warn('⚠️ Binance WebSocket disconnected');
        await this.handleBinanceDisconnect();
      });

      // Heartbeat (ping/pong)
      setInterval(() => {
        if (this.binanceWs?.readyState === WebSocket.OPEN) {
          this.binanceWs.ping();
        }
      }, 30000); // 30초마다

    } catch (error) {
      this.logger.error(`Failed to connect Binance: ${error.message}`);
      await this.handleBinanceDisconnect();
    }
  }

  // Fallback: Gate.io WebSocket
  private async connectGateio() {
    try {
      this.gateioWs = new WebSocket('wss://api.gateio.ws/ws/v4/');

      this.gateioWs.on('open', () => {
        this.logger.log('✅ Gate.io WebSocket connected (fallback)');
        this.activeProvider = 'gateio';
        this.gateioRetries = 0;

        // Subscribe to tickers
        const subscribeMsg = {
          time: Math.floor(Date.now() / 1000),
          channel: 'spot.tickers',
          event: 'subscribe',
          payload: this.SYMBOLS.map(s => s.replace('USDT', '_USDT'))
        };
        this.gateioWs.send(JSON.stringify(subscribeMsg));
      });

      this.gateioWs.on('message', async (data: WebSocket.Data) => {
        if (this.activeProvider !== 'gateio') return;

        const parsed = JSON.parse(data.toString());
        await this.handleGateioMessage(parsed);
      });

      this.gateioWs.on('error', (error) => {
        this.logger.error(`Gate.io WebSocket error: ${error.message}`);
      });

      this.gateioWs.on('close', async () => {
        this.logger.warn('⚠️ Gate.io WebSocket disconnected');
        await this.handleGateioDisconnect();
      });

    } catch (error) {
      this.logger.error(`Failed to connect Gate.io: ${error.message}`);
      await this.handleGateioDisconnect();
    }
  }

  // Binance 메시지 처리
  private async handleBinanceMessage(message: any) {
    const { data } = message;
    if (!data?.s || !data?.c) return;

    const symbol = data.s.replace('USDT', ''); // BTCUSDT → BTC
    const price = parseFloat(data.c);
    const priceChange24h = parseFloat(data.P); // 24h 변동률

    await this.updatePrice(symbol, price, priceChange24h);
  }

  // Gate.io 메시지 처리
  private async handleGateioMessage(message: any) {
    if (message.event !== 'update' || message.channel !== 'spot.tickers') return;

    const { result } = message;
    if (!result?.currency_pair || !result?.last) return;

    const symbol = result.currency_pair.replace('_USDT', ''); // BTC_USDT → BTC
    const price = parseFloat(result.last);
    const priceChange24h = parseFloat(result.change_percentage);

    await this.updatePrice(symbol, price, priceChange24h);
  }

  // 가격 업데이트 및 알림 트리거
  private async updatePrice(symbol: string, currentPrice: number, change24h: number) {
    const cacheKey = `market:price:${symbol}`;
    const previousPrice = await this.valkey.get(cacheKey);

    // Valkey에 캐싱 (5분 TTL)
    await this.valkey.setex(cacheKey, 300, currentPrice.toString());

    // 실시간 가격 이벤트 발행 (SSE로 프론트엔드 전송)
    this.emit('price_update', { symbol, price: currentPrice, change24h });

    // 급격한 변동 감지
    if (previousPrice) {
      const prev = parseFloat(previousPrice);
      const changePercent = ((currentPrice - prev) / prev) * 100;

      if (this.isSuddenChange(symbol, changePercent)) {
        this.logger.warn(`🚨 ${symbol} sudden change: ${changePercent.toFixed(2)}%`);

        await this.notificationQueue.add('market_alert', {
          type: 'market_alert',
          data: {
            symbol,
            change: changePercent,
            price: currentPrice,
            message: `${symbol} ${changePercent > 0 ? '급등' : '급락'} ${Math.abs(changePercent).toFixed(2)}%`
          }
        });
      }
    }
  }

  // 급격한 변동 판단
  private isSuddenChange(symbol: string, changePercent: number): boolean {
    const thresholds = {
      BTC: 5,
      ETH: 7,
      default: 10
    };

    const threshold = thresholds[symbol] || thresholds.default;
    return Math.abs(changePercent) >= threshold;
  }

  // Binance 재연결 로직
  private async handleBinanceDisconnect() {
    this.binanceRetries++;

    if (this.binanceRetries <= this.MAX_RETRIES) {
      const delay = Math.pow(2, this.binanceRetries - 1) * 1000; // 1s, 2s, 4s
      this.logger.log(`Retrying Binance in ${delay}ms (${this.binanceRetries}/${this.MAX_RETRIES})`);

      await new Promise(resolve => setTimeout(resolve, delay));
      await this.connectBinance();
    } else {
      this.logger.error(`❌ Binance failed after ${this.MAX_RETRIES} retries. Switching to Gate.io...`);
      await this.connectGateio();
    }
  }

  // Gate.io 재연결 로직
  private async handleGateioDisconnect() {
    this.gateioRetries++;

    if (this.gateioRetries <= this.MAX_RETRIES) {
      const delay = Math.pow(2, this.gateioRetries - 1) * 1000;
      this.logger.log(`Retrying Gate.io in ${delay}ms (${this.gateioRetries}/${this.MAX_RETRIES})`);

      await new Promise(resolve => setTimeout(resolve, delay));
      await this.connectGateio();
    } else {
      this.logger.error(`❌ Gate.io failed after ${this.MAX_RETRIES} retries.`);
      // Binance 재시도 (무한 루프 방지를 위해 딜레이 후)
      await new Promise(resolve => setTimeout(resolve, 30000)); // 30초 대기
      this.binanceRetries = 0;
      await this.connectBinance();
    }
  }

  // 모든 연결 종료
  private disconnectAll() {
    if (this.binanceWs) {
      this.binanceWs.close();
      this.binanceWs = null;
    }
    if (this.gateioWs) {
      this.gateioWs.close();
      this.gateioWs = null;
    }
  }

  // 현재 활성 제공자 확인 (모니터링용)
  getActiveProvider(): string {
    return this.activeProvider;
  }
}
```

### 8.3 Health Check & Monitoring

```typescript
// market/market.controller.ts
@Get('health/websocket')
async getWebSocketHealth() {
  return {
    activeProvider: this.websocketService.getActiveProvider(),
    timestamp: new Date().toISOString()
  };
}
```

---

## 9. Error Handling

### 9.1 Custom Exceptions

```typescript
class HallucinationDetectedException extends BadRequestException {
  constructor(actual: number, claimed: number) {
    super(`AI hallucination detected: claimed ${claimed}, actual ${actual}`);
  }
}

class RateLimitExceededException extends TooManyRequestsException {
  constructor(limit: number) {
    super(`Rate limit exceeded: ${limit} requests per minute`);
  }
}

class TierLimitException extends ForbiddenException {
  constructor(feature: string, requiredTier: string) {
    super(`Feature '${feature}' requires ${requiredTier} tier`);
  }
}
```

### 9.2 Global Exception Filter

```typescript
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();

    const status = exception instanceof HttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;

    // Log to Sentry
    Sentry.captureException(exception);

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: exception.message || 'Internal server error'
    });
  }
}
```

---

## 10. Performance Targets

### 10.1 Performance Metrics

```typescript
interface PerformanceTargets {
  apiResponseTime: {
    metric: "95th percentile";
    target: "< 200ms";
    measurement: "Sentry Traces";
  };
  sseFirstToken: {
    metric: "Time to first token";
    target: "< 2s";
    measurement: "Custom metric";
  };
  databaseQuery: {
    metric: "Query duration";
    target: "< 50ms";
    measurement: "Prisma Logging";
  };
  cacheHitRate: {
    metric: "Hit rate";
    target: "> 80%";
    measurement: "Valkey INFO stats";
  };
}
```

---

## 11. Security

### 11.1 Authentication & Authorization

```typescript
interface SecurityConfig {
  authentication: {
    provider: "Auth.js";
    method: "Google OAuth";
    storage: "PostgreSQL or JWT";
    csrfProtection: true;
  };
  authorization: {
    guards: ["AuthGuard", "TierGuard"];
    example: "@RequireTier('pro')";
  };
}
```

### 11.2 Input Validation

```typescript
class CreateTradeDto {
  @IsIn(['BTC', 'ETH', 'SOL', 'BNB', 'DOGE', 'XRP'])
  symbol: string;

  @IsIn(['buy', 'sell'])
  action: string;

  @IsNumber()
  @Min(0)
  price: number;
}
```

### 11.3 Rate Limiting

```typescript
@UseGuards(ThrottlerGuard)
@Throttle(10, 60)  // 10 requests per minute
async sendMessage() {
  // Implementation
}
```

---

## 12. Monitoring

### 12.1 Custom Metrics

```typescript
interface CustomMetrics {
  chatResponseTime: "histogram";
  aiHallucinationRate: "counter";
  marketApiCalls: "counter";
  jobsCompleted: "counter";
  jobsFailed: "counter";
}
```

### 12.2 Alerting Rules

```typescript
interface AlertingRules {
  apiErrorRate: "> 5% for 5 minutes";
  sseFirstToken: "> 5s for 1 minute";
  databaseConnectionPool: "> 90%";
  valkeyMemory: "> 80%";
  bullmqQueueSize: "> 1000";
}
```

---

## Appendix A: Environment Variables

```bash
# Application
NODE_ENV=development        # development | production
PORT=3000
FRONTEND_URL=http://localhost:3000

# Database
DATABASE_URL="postgresql://user:pass@localhost:5432/sage"
SHADOW_DATABASE_URL="postgresql://user:pass@localhost:5432/sage_shadow"

# Valkey (Redis-compatible)
VALKEY_URL="valkey://localhost:6379"

# AI - LLM Provider (NEW)
LLM_PROVIDER=ollama         # ollama | anthropic
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=llama2         # llama2:latest (7B Q4_0)
ANTHROPIC_API_KEY=          # Production only

# Auth
GOOGLE_CLIENT_ID="..."
GOOGLE_CLIENT_SECRET="..."
NEXTAUTH_SECRET="..."

# External APIs (WebSocket)
BINANCE_WS_URL="wss://stream.binance.com:9443/ws"
GATEIO_WS_URL="wss://api.gateio.ws/ws/v4/"

# External APIs (REST - Backup)
COINGECKO_API_KEY="..."
DISCORD_WEBHOOK_URL="..."
```

**LLM Provider 설정**:
- **Development**: `LLM_PROVIDER=ollama` - 로컬 Ollama 사용 (무료)
- **Production**: `LLM_PROVIDER=anthropic` - Claude API 사용 (ANTHROPIC_API_KEY 필수)

## Appendix B: Docker Development Environment (Recommended)

### Why Docker?

Docker를 사용하면 다음과 같은 이점이 있습니다:
- **환경 격리**: 로컬 시스템을 깨끗하게 유지
- **팀 협업**: 모든 개발자가 동일한 환경 사용
- **빠른 설정**: 새 팀원 온보딩 시 `docker compose up` 한 번으로 완료
- **프로덕션 유사성**: 실제 배포 환경과 유사한 컨테이너 기반 개발
- **여러 서비스 관리**: PostgreSQL + Valkey를 한 번에 실행

### docker-compose.yml

```yaml
version: '3.8'

services:
  # PostgreSQL 18 Database
  postgres:
    image: postgres:18-alpine
    container_name: sage-postgres
    restart: unless-stopped
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: sage
      POSTGRES_PASSWORD: sage123
      POSTGRES_DB: sage
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sage"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Valkey 8.x (Redis-compatible cache)
  valkey:
    image: valkey/valkey:8-alpine
    container_name: sage-valkey
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - valkey_data:/data
    healthcheck:
      test: ["CMD", "valkey-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
  valkey_data:
```

### Quick Start

```bash
# 1. Docker 서비스 시작
docker compose up -d

# 2. 상태 확인
docker compose ps

# 3. 로그 확인
docker compose logs -f postgres
docker compose logs -f valkey

# 4. .env 파일 설정
DATABASE_URL=postgresql://sage:sage123@localhost:5432/sage
VALKEY_URL=valkey://localhost:6379

# 5. Prisma 마이그레이션 실행
pnpm prisma:generate
pnpm prisma:migrate

# 6. 개발 서버 시작
pnpm dev
```

### Docker 명령어

```bash
# 서비스 시작
docker compose up -d              # 백그라운드에서 시작
docker compose up                 # 포그라운드에서 시작 (로그 실시간 확인)

# 서비스 중지
docker compose stop               # 중지 (데이터 유지)
docker compose down               # 중지 및 컨테이너 삭제 (데이터는 유지)
docker compose down -v            # 중지 및 볼륨까지 삭제 (데이터 완전 삭제)

# 상태 확인
docker compose ps                 # 서비스 상태
docker compose logs postgres      # PostgreSQL 로그
docker compose logs -f valkey     # Valkey 로그 (실시간)

# 데이터베이스 초기화 (개발 중 필요 시)
docker compose down -v            # 기존 데이터 삭제
docker compose up -d postgres     # PostgreSQL만 재시작
pnpm prisma:migrate               # 마이그레이션 재실행
```

### Troubleshooting

**포트 충돌 시**:
```bash
# 기존 PostgreSQL 중지
brew services stop postgresql@18

# 또는 docker-compose.yml에서 포트 변경
ports:
  - "5433:5432"  # 로컬 5433 포트 사용
```

**데이터 초기화**:
```bash
# 모든 데이터 삭제 후 재시작
docker compose down -v
docker compose up -d
pnpm prisma:generate
pnpm prisma:migrate
```

---

## Appendix C: Development Commands

```bash
# Development server
pnpm run start:dev              # Start with watch mode
pnpm run start:debug            # Start in debug mode

# Build
pnpm run build                  # Production build
pnpm run start:prod             # Run production build

# Database (Prisma)
npx prisma generate             # Generate Prisma Client
npx prisma migrate dev          # Create and apply migration
npx prisma migrate deploy       # Apply migrations in production
npx prisma studio               # Open Prisma Studio (DB GUI)

# Testing
pnpm run test                   # Run unit tests
pnpm run test:watch             # Run tests in watch mode
pnpm run test:cov               # Generate coverage report
pnpm run test:e2e               # Run e2e tests

# Linting
pnpm run lint                   # Run ESLint
pnpm run format                 # Format with Prettier
```

---

---

**Document Version**: 3.2
**Last Updated**: 2026-01-07
**Architecture**: Layered + Domain (Clean Lite), TypeScript Fullstack
**Tech Stack**: Nest.js 10.x + Prisma 5.19.1 + PostgreSQL 18 + Valkey 8.1.5
**Maintainer**: Sam (dev@5010.tech)

### Changelog

**3.2** (2026-01-07)
- LLM Provider 추상화 추가 (Ollama + Anthropic 지원)
- Agent Pipeline 상세 구현 문서화:
  - Manager Agent: LLM 분류 + fallback 키워드 분석
  - Analyst Agent: 상세한 AnalystOutput 인터페이스 (PriceData, FearGreedData)
  - Persona Agent: LLMService 통합, 한국어 페르소나 프롬프트 상세화
  - Risk Agent: LLM 검증 + fallback 키워드 검증
- 모든 Agent에 Fallback 메커니즘 문서화 (LLM 불가 시 대응)
- 환경 변수 추가: LLM_PROVIDER, OLLAMA_BASE_URL, OLLAMA_MODEL

**3.1** (2026-01-07)
- Prisma 버전을 5.x → 5.19.1로 구체화
- Prisma v7 호환성 문제 명시 (설정 파일 분리)
- Valkey 버전을 8.x → 8.1.5로 구체화

**v3.0 (2025-12-26)**:
- PostgreSQL 16 → 18: 5-year LTS, JSON 30% faster, query optimizer improvements
- Redis 7.x → Valkey 8.x: License stability, Linux Foundation backing
- Added RiskProfile ENUM type for type safety
- Added UserConsent table for GDPR compliance
- Added architecture flow diagrams (Registration, Chat, Market Analysis)
