# 시스템 아키텍처 설계 가이드

Go 프로젝트의 아키텍처 설계를 위한 의사결정 가이드입니다.

## 📋 아키텍처 결정 템플릿 (ADR - Architecture Decision Record)

### ADR-001: [결정 제목]

**상황 (Context)**
- 현재 상황과 문제점 설명
- 해결해야 할 과제

**결정 (Decision)**
- 선택한 아키텍처/기술/패턴
- 구체적인 구현 방법

**근거 (Rationale)**
- 이 결정을 내린 이유
- 고려했던 대안들
- 장단점 분석

**결과 (Consequences)**
- 이 결정의 영향
- 향후 고려사항

---

## 🏗 아키텍처 패턴 가이드

### 1. 레이어드 아키텍처 (Layered Architecture)

```
┌─────────────────────────────────────┐
│           Presentation Layer        │ ← HTTP Handlers, CLI Commands
├─────────────────────────────────────┤
│           Application Layer         │ ← Services, Use Cases
├─────────────────────────────────────┤
│             Domain Layer            │ ← Business Logic, Entities
├─────────────────────────────────────┤
│          Infrastructure Layer       │ ← Database, External APIs
└─────────────────────────────────────┘
```

**언제 사용하나?**
- 중소 규모 웹 애플리케이션
- 명확한 계층 구조가 필요한 경우
- 팀이 전통적인 MVC 패턴에 익숙한 경우

**Go 구현 예시:**
```go
// cmd/server/main.go - Presentation Entry Point
func main() {
    cfg := config.Load()
    app := app.New(cfg)
    app.Run()
}

// internal/handler/ - Presentation Layer
type UserHandler struct {
    userService service.UserService
}

// internal/service/ - Application Layer  
type userService struct {
    userRepo domain.UserRepository
}

// internal/domain/ - Domain Layer
type User struct {
    ID    string
    Email string
}

// internal/repository/ - Infrastructure Layer
type userRepository struct {
    db *sql.DB
}
```

### 2. 헥사고날 아키텍처 (Hexagonal Architecture)

```
                    ┌─────────────────┐
                    │   외부 시스템      │
                    │  (Web, CLI,     │
                    │   Database)     │
                    └─────────┬───────┘
                              │
                    ┌─────────▼───────┐
                    │      Port       │ ← 인터페이스 정의
                    ├─────────────────┤
                    │   Application   │ ← 비즈니스 로직
                    │      Core       │
                    ├─────────────────┤
                    │    Adapter      │ ← 구현체
                    └─────────────────┘
```

**언제 사용하나?**
- 복잡한 비즈니스 로직
- 다양한 외부 시스템 연동
- 테스트 용이성이 중요한 경우

**Go 구현 예시:**
```go
// domain/ports/user.go - Port (인터페이스)
type UserRepository interface {
    Save(ctx context.Context, user *User) error
    FindByEmail(ctx context.Context, email string) (*User, error)
}

type UserService interface {
    CreateUser(ctx context.Context, req CreateUserRequest) (*User, error)
}

// domain/user.go - Core Domain
type User struct {
    id    UserID
    email Email
    name  Name
}

func (u *User) UpdateEmail(newEmail Email) error {
    // 비즈니스 규칙 검증
    if err := u.validateEmailChange(newEmail); err != nil {
        return err
    }
    u.email = newEmail
    return nil
}

// adapters/repository/postgres_user.go - Adapter
type postgresUserRepository struct {
    db *sql.DB
}

func (r *postgresUserRepository) Save(ctx context.Context, user *User) error {
    // PostgreSQL 구현
}

// adapters/service/user_service.go - Application Service
type userService struct {
    userRepo ports.UserRepository
    emailSvc ports.EmailService
}
```

### 3. 마이크로서비스 아키텍처

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  User Service │    │ Order Service│    │ Payment Svc  │
│              │    │              │    │              │
│ ┌──────────┐ │    │ ┌──────────┐ │    │ ┌──────────┐ │
│ │    DB    │ │    │ │    DB    │ │    │ │    DB    │ │
│ └──────────┘ │    │ └──────────┘ │    │ └──────────┘ │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼───────┐
                    │  API Gateway │
                    └──────────────┘
```

**언제 사용하나?**
- 대규모 시스템
- 독립적인 배포가 필요한 경우
- 팀이 분리되어 있는 경우

**Go 구현 고려사항:**
```go
// 서비스 간 통신
type UserServiceClient interface {
    GetUser(ctx context.Context, userID string) (*User, error)
}

// gRPC 클라이언트
type grpcUserClient struct {
    conn *grpc.ClientConn
}

// HTTP 클라이언트
type httpUserClient struct {
    client *http.Client
    baseURL string
}

// 서비스 디스커버리
type ServiceDiscovery interface {
    Discover(serviceName string) ([]string, error)
}
```

## 🔧 기술 스택 결정 가이드

### 데이터베이스 선택

| 요구사항 | 추천 | 이유 |
|---------|------|------|
| ACID 트랜잭션 필요 | PostgreSQL | 강력한 ACID 지원, JSON 지원 |
| 높은 성능, 단순 스키마 | MySQL | 성능 최적화, 단순함 |
| 스키마 유연성 필요 | MongoDB | NoSQL, 스키마 유연성 |
| 캐싱 | Redis | 메모리 기반, 다양한 데이터 구조 |
| 시계열 데이터 | InfluxDB/TimescaleDB | 시계열 최적화 |

**Go 데이터베이스 접근 패턴:**
```go
// database/connection.go
type DB struct {
    primary *sql.DB
    replica *sql.DB
}

func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error) {
    return db.replica.QueryContext(ctx, query, args...)
}

func (db *DB) ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error) {
    return db.primary.ExecContext(ctx, query, args...)
}

// repository 패턴
type Repository[T any] interface {
    Create(ctx context.Context, entity T) error
    GetByID(ctx context.Context, id string) (T, error)
    Update(ctx context.Context, entity T) error
    Delete(ctx context.Context, id string) error
}
```

### 웹 프레임워크 선택

| 프레임워크 | 적합한 경우 | 특징 |
|-----------|------------|------|
| net/http (표준) | 단순한 API, 최대 제어 | 의존성 없음, 최고 성능 |
| Gin | 빠른 개발, REST API | 높은 성능, 미들웨어 풍부 |
| Echo | 미들웨어 중심 | 단순함, 좋은 성능 |
| Fiber | Express.js 스타일 | 매우 빠름, Express 유사 |
| Gorilla Mux | 표준 라이브러리 확장 | 표준과 유사, 라우팅 기능 |

**표준 라이브러리 사용 예시:**
```go
// internal/server/server.go
type Server struct {
    router *http.ServeMux
    server *http.Server
}

func (s *Server) SetupRoutes() {
    s.router.HandleFunc("/api/users", s.handleUsers)
    s.router.HandleFunc("/api/users/", s.handleUserByID)
    s.router.HandleFunc("/health", s.handleHealth)
}

func (s *Server) handleUsers(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        s.listUsers(w, r)
    case http.MethodPost:
        s.createUser(w, r)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}
```

## 📊 성능 고려사항

### 1. 동시성 패턴

```go
// Worker Pool 패턴
type WorkerPool struct {
    workerCount int
    jobQueue    chan Job
    quit        chan bool
}

func (wp *WorkerPool) Start() {
    for i := 0; i < wp.workerCount; i++ {
        go wp.worker()
    }
}

func (wp *WorkerPool) worker() {
    for {
        select {
        case job := <-wp.jobQueue:
            job.Process()
        case <-wp.quit:
            return
        }
    }
}

// Fan-out/Fan-in 패턴
func FanOut(input <-chan Work, workers int) []<-chan Result {
    outputs := make([]<-chan Result, workers)
    
    for i := 0; i < workers; i++ {
        output := make(chan Result)
        outputs[i] = output
        
        go func() {
            defer close(output)
            for work := range input {
                result := process(work)
                output <- result
            }
        }()
    }
    
    return outputs
}

func FanIn(inputs ...<-chan Result) <-chan Result {
    output := make(chan Result)
    
    var wg sync.WaitGroup
    wg.Add(len(inputs))
    
    for _, input := range inputs {
        go func(ch <-chan Result) {
            defer wg.Done()
            for result := range ch {
                output <- result
            }
        }(input)
    }
    
    go func() {
        wg.Wait()
        close(output)
    }()
    
    return output
}
```

### 2. 캐싱 전략

```go
// 메모리 캐시
type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]cacheItem[V]
    ttl   time.Duration
}

type cacheItem[V any] struct {
    value  V
    expiry time.Time
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    item, exists := c.items[key]
    if !exists || time.Now().After(item.expiry) {
        var zero V
        return zero, false
    }
    
    return item.value, true
}

func (c *Cache[K, V]) Set(key K, value V) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    c.items[key] = cacheItem[V]{
        value:  value,
        expiry: time.Now().Add(c.ttl),
    }
}

// Redis 캐시 래퍼
type RedisCache struct {
    client *redis.Client
}

func (r *RedisCache) Get(ctx context.Context, key string, dest interface{}) error {
    val, err := r.client.Get(ctx, key).Result()
    if err != nil {
        return err
    }
    
    return json.Unmarshal([]byte(val), dest)
}
```

### 3. 데이터베이스 최적화

```go
// 연결 풀 설정
func setupDatabase(cfg DatabaseConfig) *sql.DB {
    db, err := sql.Open("postgres", cfg.DSN)
    if err != nil {
        panic(err)
    }
    
    // 연결 풀 설정
    db.SetMaxOpenConns(cfg.MaxOpenConns)    // 최대 열린 연결 수
    db.SetMaxIdleConns(cfg.MaxIdleConns)    // 최대 유휴 연결 수
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime) // 연결 최대 수명
    
    return db
}

// 배치 처리
func (r *userRepository) CreateBatch(ctx context.Context, users []*User) error {
    tx, err := r.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    stmt, err := tx.PrepareContext(ctx, `
        INSERT INTO users (id, email, name) VALUES ($1, $2, $3)
    `)
    if err != nil {
        return err
    }
    defer stmt.Close()
    
    for _, user := range users {
        if _, err := stmt.ExecContext(ctx, user.ID, user.Email, user.Name); err != nil {
            return err
        }
    }
    
    return tx.Commit()
}
```

## 🔐 보안 아키텍처

### 1. 인증/인가

```go
// JWT 토큰 관리
type TokenManager struct {
    secretKey   []byte
    accessTTL   time.Duration
    refreshTTL  time.Duration
}

func (tm *TokenManager) GenerateTokens(userID string) (*TokenPair, error) {
    accessToken, err := tm.generateAccessToken(userID)
    if err != nil {
        return nil, err
    }
    
    refreshToken, err := tm.generateRefreshToken(userID)
    if err != nil {
        return nil, err
    }
    
    return &TokenPair{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
    }, nil
}

// 미들웨어 체인
func AuthMiddleware(tm *TokenManager) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := extractToken(r)
            if token == "" {
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }
            
            claims, err := tm.ValidateToken(token)
            if err != nil {
                http.Error(w, "Invalid token", http.StatusUnauthorized)
                return
            }
            
            ctx := context.WithValue(r.Context(), "userID", claims.UserID)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

### 2. Rate Limiting

```go
// Token Bucket 구현
type RateLimiter struct {
    mu       sync.Mutex
    buckets  map[string]*bucket
    capacity int
    refill   time.Duration
}

type bucket struct {
    tokens   int
    lastSeen time.Time
}

func (rl *RateLimiter) Allow(key string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    b, exists := rl.buckets[key]
    if !exists {
        b = &bucket{
            tokens:   rl.capacity - 1,
            lastSeen: time.Now(),
        }
        rl.buckets[key] = b
        return true
    }
    
    now := time.Now()
    tokensToAdd := int(now.Sub(b.lastSeen) / rl.refill)
    b.tokens = min(rl.capacity, b.tokens+tokensToAdd)
    b.lastSeen = now
    
    if b.tokens > 0 {
        b.tokens--
        return true
    }
    
    return false
}
```

## 📝 아키텍처 문서화

### 1. C4 모델 다이어그램

```
Level 1: System Context
┌─────────────────────────────────────────────────┐
│                   User                          │
│                     │                           │
│                     ▼                           │
│  ┌─────────────────────────────────────────┐    │
│  │           Our System                    │    │
│  │                                         │    │
│  │  ┌─────────┐    ┌─────────┐           │    │
│  │  │   API   │    │Database │           │    │
│  │  └─────────┘    └─────────┘           │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘

Level 2: Container Diagram
┌─────────────────────────────────────────────────┐
│                Web Browser                      │
│                     │                           │
│                     ▼                           │
│  ┌─────────────────────────────────────────┐    │
│  │           Load Balancer                 │    │
│  │                  │                      │    │
│  │                  ▼                      │    │
│  │  ┌─────────┐    ┌─────────┐           │    │
│  │  │  API    │───▶│Database │           │    │
│  │  │ Server  │    │         │           │    │
│  │  └─────────┘    └─────────┘           │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

### 2. 의사결정 로그

```yaml
# docs/decisions/001-database-choice.md
# ADR-001: PostgreSQL을 주 데이터베이스로 선택

## Status
Accepted

## Context
사용자 데이터와 트랜잭션 데이터를 저장할 데이터베이스가 필요합니다.
- ACID 트랜잭션 지원 필요
- JSON 데이터 타입 지원 필요
- 높은 동시성 처리 필요

## Decision
PostgreSQL을 주 데이터베이스로 사용합니다.

## Consequences
### Positive
- 강력한 ACID 트랜잭션 지원
- 우수한 JSON 지원 (JSONB)
- 활발한 커뮤니티와 생태계

### Negative
- MySQL 대비 설정이 복잡
- 일부 클라우드 서비스에서 비용이 높을 수 있음
```

## 🎯 아키텍처 검증 체크리스트

### 확장성
- [ ] 수평 확장이 가능한가?
- [ ] 상태가 없는(stateless) 설계인가?
- [ ] 병목점이 명확히 식별되는가?

### 가용성
- [ ] 단일 장애점(SPOF)이 없는가?
- [ ] 장애 격리가 가능한가?
- [ ] 복구 절차가 명확한가?

### 보안
- [ ] 최소 권한 원칙을 따르는가?
- [ ] 인증/인가가 적절히 구현되었는가?
- [ ] 민감한 데이터가 암호화되는가?

### 유지보수성
- [ ] 코드가 모듈화되어 있는가?
- [ ] 의존성 방향이 올바른가?
- [ ] 테스트하기 쉬운 구조인가?

이 가이드를 참고하여 프로젝트에 적합한 아키텍처를 설계하고 문서화하세요!