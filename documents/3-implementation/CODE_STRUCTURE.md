# Go 프로젝트 코드 구조 가이드

Go 프로젝트의 효과적인 코드 구조화를 위한 가이드입니다.

## 📋 코딩 스타일 가이드

이 프로젝트는 **Uber Go Style Guide**를 따릅니다.
- 📚 **참고 문서**: `code-guide/DEV_STYLE.md` (Uber Go Style Guide)
- 🔗 **원본 링크**: [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)

주요 원칙:
- 명확하고 읽기 쉬운 코드 작성
- 일관된 네이밍 컨벤션
- 적절한 에러 처리
- 효율적인 메모리 사용

## 📁 표준 Go 프로젝트 레이아웃

### 기본 디렉토리 구조

```
myproject/
├── cmd/                    # 메인 애플리케이션
│   ├── api-server/        # REST API 서버
│   │   └── main.go
│   ├── cli/               # CLI 도구
│   │   └── main.go
│   └── worker/            # 백그라운드 작업자
│       └── main.go
├── internal/              # 비공개 애플리케이션 코드
│   ├── app/              # 애플리케이션 로직
│   ├── config/           # 설정 관리
│   ├── domain/           # 도메인 모델
│   ├── handler/          # HTTP/gRPC 핸들러
│   ├── repository/       # 데이터 액세스 레이어
│   ├── service/          # 비즈니스 로직
│   └── utils/            # 내부 유틸리티
├── pkg/                  # 공개 라이브러리 코드
│   ├── client/           # 클라이언트 라이브러리
│   ├── errors/           # 에러 타입 정의
│   └── validator/        # 검증 유틸리티
├── api/                  # API 정의
│   ├── openapi/          # OpenAPI/Swagger 스펙
│   │   └── api.yaml
│   └── proto/            # Protocol Buffer 정의
│       └── service.proto
├── web/                  # 웹 애플리케이션 (선택)
│   ├── static/           # 정적 파일
│   └── templates/        # HTML 템플릿
├── configs/              # 설정 파일
│   ├── config.yaml
│   └── config.prod.yaml
├── scripts/              # 스크립트
│   ├── build.sh
│   └── deploy.sh
├── test/                 # 추가 테스트
│   ├── integration/      # 통합 테스트
│   └── e2e/             # End-to-End 테스트
├── testdata/            # 테스트 데이터
├── docs/                # 프로젝트 문서
├── .gitignore
├── .golangci.yml
├── docker-compose.yml
├── Dockerfile
├── Makefile
├── README.md
├── go.mod
└── go.sum
```

## 🏗 레이어별 상세 구조

### 1. `cmd/` - 애플리케이션 진입점

각 애플리케이션의 main 함수를 포함합니다.

```go
// cmd/api-server/main.go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    "syscall"
    
    "myproject/internal/app"
    "myproject/internal/config"
)

func main() {
    cfg, err := config.Load()
    if err != nil {
        log.Fatalf("failed to load config: %v", err)
    }
    
    app := app.New(cfg)
    
    ctx, cancel := signal.NotifyContext(context.Background(), 
        os.Interrupt, syscall.SIGTERM)
    defer cancel()
    
    if err := app.Run(ctx); err != nil {
        log.Fatalf("app failed: %v", err)
    }
}
```

### 2. `internal/` - 비공개 애플리케이션 코드

#### `internal/domain/` - 도메인 모델

```go
// internal/domain/user.go
package domain

import (
    "time"
    "errors"
)

type User struct {
    ID        string
    Email     string
    Name      string
    CreatedAt time.Time
    UpdatedAt time.Time
}

func (u *User) Validate() error {
    if u.Email == "" {
        return errors.New("email is required")
    }
    if u.Name == "" {
        return errors.New("name is required")
    }
    return nil
}

type UserRepository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id string) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
}

type UserService interface {
    CreateUser(ctx context.Context, req CreateUserRequest) (*User, error)
    GetUser(ctx context.Context, id string) (*User, error)
    UpdateUser(ctx context.Context, id string, req UpdateUserRequest) (*User, error)
}
```

#### `internal/service/` - 비즈니스 로직

```go
// internal/service/user.go
package service

import (
    "context"
    "fmt"
    
    "myproject/internal/domain"
)

type userService struct {
    userRepo domain.UserRepository
}

func NewUserService(userRepo domain.UserRepository) domain.UserService {
    return &userService{
        userRepo: userRepo,
    }
}

func (s *userService) CreateUser(ctx context.Context, req CreateUserRequest) (*domain.User, error) {
    // 비즈니스 로직 검증
    if err := req.Validate(); err != nil {
        return nil, fmt.Errorf("invalid request: %w", err)
    }
    
    // 중복 확인
    existing, err := s.userRepo.GetByEmail(ctx, req.Email)
    if err == nil && existing != nil {
        return nil, errors.New("user with this email already exists")
    }
    
    user := &domain.User{
        ID:    generateID(),
        Email: req.Email,
        Name:  req.Name,
    }
    
    if err := s.userRepo.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("failed to create user: %w", err)
    }
    
    return user, nil
}
```

#### `internal/repository/` - 데이터 액세스

```go
// internal/repository/user.go
package repository

import (
    "context"
    "database/sql"
    "fmt"
    
    "myproject/internal/domain"
)

type userRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) domain.UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Create(ctx context.Context, user *domain.User) error {
    query := `
        INSERT INTO users (id, email, name, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5)
    `
    
    _, err := r.db.ExecContext(ctx, query,
        user.ID, user.Email, user.Name, user.CreatedAt, user.UpdatedAt)
    if err != nil {
        return fmt.Errorf("failed to insert user: %w", err)
    }
    
    return nil
}

func (r *userRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    query := `
        SELECT id, email, name, created_at, updated_at
        FROM users
        WHERE id = $1
    `
    
    user := &domain.User{}
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &user.ID, &user.Email, &user.Name, &user.CreatedAt, &user.UpdatedAt)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, domain.ErrUserNotFound
        }
        return nil, fmt.Errorf("failed to get user: %w", err)
    }
    
    return user, nil
}
```

#### `internal/handler/` - HTTP 핸들러

```go
// internal/handler/user.go
package handler

import (
    "encoding/json"
    "net/http"
    
    "myproject/internal/domain"
    "myproject/pkg/errors"
)

type UserHandler struct {
    userService domain.UserService
}

func NewUserHandler(userService domain.UserService) *UserHandler {
    return &UserHandler{
        userService: userService,
    }
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        errors.WriteError(w, http.StatusBadRequest, "invalid request body")
        return
    }
    
    user, err := h.userService.CreateUser(r.Context(), req)
    if err != nil {
        errors.WriteError(w, http.StatusInternalServerError, err.Error())
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    id := r.URL.Query().Get("id")
    if id == "" {
        errors.WriteError(w, http.StatusBadRequest, "id is required")
        return
    }
    
    user, err := h.userService.GetUser(r.Context(), id)
    if err != nil {
        if errors.Is(err, domain.ErrUserNotFound) {
            errors.WriteError(w, http.StatusNotFound, "user not found")
            return
        }
        errors.WriteError(w, http.StatusInternalServerError, err.Error())
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

#### `internal/config/` - 설정 관리

```go
// internal/config/config.go
package config

import (
    "fmt"
    "os"
    "strconv"
)

type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    Redis    RedisConfig
}

type ServerConfig struct {
    Port         int    `yaml:"port"`
    Host         string `yaml:"host"`
    ReadTimeout  int    `yaml:"read_timeout"`
    WriteTimeout int    `yaml:"write_timeout"`
}

type DatabaseConfig struct {
    Host     string `yaml:"host"`
    Port     int    `yaml:"port"`
    User     string `yaml:"user"`
    Password string `yaml:"password"`
    Name     string `yaml:"name"`
    SSLMode  string `yaml:"ssl_mode"`
}

func Load() (*Config, error) {
    cfg := &Config{
        Server: ServerConfig{
            Port:         getEnvInt("SERVER_PORT", 8080),
            Host:         getEnv("SERVER_HOST", "localhost"),
            ReadTimeout:  getEnvInt("SERVER_READ_TIMEOUT", 30),
            WriteTimeout: getEnvInt("SERVER_WRITE_TIMEOUT", 30),
        },
        Database: DatabaseConfig{
            Host:     getEnv("DB_HOST", "localhost"),
            Port:     getEnvInt("DB_PORT", 5432),
            User:     getEnv("DB_USER", "postgres"),
            Password: getEnv("DB_PASSWORD", ""),
            Name:     getEnv("DB_NAME", "myapp"),
            SSLMode:  getEnv("DB_SSL_MODE", "disable"),
        },
    }
    
    return cfg, cfg.Validate()
}

func (c *Config) Validate() error {
    if c.Database.Password == "" {
        return fmt.Errorf("database password is required")
    }
    return nil
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if intValue, err := strconv.Atoi(value); err == nil {
            return intValue
        }
    }
    return defaultValue
}
```

### 3. `pkg/` - 공개 라이브러리

다른 프로젝트에서도 사용할 수 있는 라이브러리 코드입니다.

```go
// pkg/errors/errors.go
package errors

import (
    "encoding/json"
    "net/http"
)

type ErrorResponse struct {
    Error   string `json:"error"`
    Code    int    `json:"code"`
    Message string `json:"message"`
}

func WriteError(w http.ResponseWriter, code int, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    
    resp := ErrorResponse{
        Error:   http.StatusText(code),
        Code:    code,
        Message: message,
    }
    
    json.NewEncoder(w).Encode(resp)
}

// pkg/validator/validator.go
package validator

import (
    "fmt"
    "net/mail"
    "regexp"
)

func ValidateEmail(email string) error {
    if _, err := mail.ParseAddress(email); err != nil {
        return fmt.Errorf("invalid email format: %w", err)
    }
    return nil
}

func ValidatePassword(password string) error {
    if len(password) < 8 {
        return fmt.Errorf("password must be at least 8 characters")
    }
    
    hasUpper := regexp.MustCompile(`[A-Z]`).MatchString(password)
    hasLower := regexp.MustCompile(`[a-z]`).MatchString(password)
    hasNumber := regexp.MustCompile(`\d`).MatchString(password)
    
    if !hasUpper || !hasLower || !hasNumber {
        return fmt.Errorf("password must contain uppercase, lowercase, and number")
    }
    
    return nil
}
```

## 🔄 의존성 방향

### 클린 아키텍처 원칙

```
외부 → 내부 방향으로만 의존성이 흘러야 함

cmd/ → internal/app → internal/handler → internal/service → internal/domain
                   ↘ internal/repository → internal/domain
```

### 의존성 주입 패턴

```go
// internal/app/app.go
package app

import (
    "context"
    "database/sql"
    "net/http"
    
    "myproject/internal/config"
    "myproject/internal/handler"
    "myproject/internal/repository"
    "myproject/internal/service"
)

type App struct {
    server *http.Server
    db     *sql.DB
}

func New(cfg *config.Config) *App {
    // 데이터베이스 연결
    db := setupDatabase(cfg.Database)
    
    // 의존성 주입
    userRepo := repository.NewUserRepository(db)
    userService := service.NewUserService(userRepo)
    userHandler := handler.NewUserHandler(userService)
    
    // 라우터 설정
    router := setupRouter(userHandler)
    
    server := &http.Server{
        Addr:    fmt.Sprintf("%s:%d", cfg.Server.Host, cfg.Server.Port),
        Handler: router,
    }
    
    return &App{
        server: server,
        db:     db,
    }
}

func (a *App) Run(ctx context.Context) error {
    errChan := make(chan error, 1)
    
    go func() {
        if err := a.server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            errChan <- err
        }
    }()
    
    select {
    case err := <-errChan:
        return err
    case <-ctx.Done():
        return a.server.Shutdown(context.Background())
    }
}
```

## 📝 패키지 네이밍 규칙

### 좋은 패키지명
- 짧고 명확함: `user`, `auth`, `config`
- 명사 사용: `handler`, `service`, `repository`
- 복수형 피함: `users` ❌ → `user` ✅

### 파일 네이밍
- 패키지 기능 중심: `user.go`, `user_test.go`
- 타입별 분리: `handlers.go`, `models.go`, `repository.go`

## 🧪 테스트 구조

```
internal/
├── domain/
│   ├── user.go
│   └── user_test.go
├── service/
│   ├── user.go
│   ├── user_test.go
│   └── testdata/
│       └── users.json
└── repository/
    ├── user.go
    ├── user_test.go
    └── user_integration_test.go
```

### 테스트 파일 예시

```go
// internal/service/user_test.go
package service

import (
    "context"
    "testing"
    
    "myproject/internal/domain"
    "myproject/internal/repository/mocks"
)

func TestUserService_CreateUser(t *testing.T) {
    tests := []struct {
        name    string
        req     CreateUserRequest
        setup   func(*mocks.MockUserRepository)
        wantErr bool
    }{
        {
            name: "success",
            req: CreateUserRequest{
                Email: "test@example.com",
                Name:  "Test User",
            },
            setup: func(repo *mocks.MockUserRepository) {
                repo.EXPECT().
                    GetByEmail(gomock.Any(), "test@example.com").
                    Return(nil, domain.ErrUserNotFound)
                repo.EXPECT().
                    Create(gomock.Any(), gomock.Any()).
                    Return(nil)
            },
            wantErr: false,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            ctrl := gomock.NewController(t)
            defer ctrl.Finish()
            
            repo := mocks.NewMockUserRepository(ctrl)
            tt.setup(repo)
            
            service := NewUserService(repo)
            
            _, err := service.CreateUser(context.Background(), tt.req)
            if (err != nil) != tt.wantErr {
                t.Errorf("CreateUser() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

## 🔧 생성 스크립트

### 패키지 생성 자동화

```bash
#!/bin/bash
# scripts/new-service.sh

SERVICE_NAME=$1

if [ -z "$SERVICE_NAME" ]; then
    echo "Usage: $0 <service-name>"
    exit 1
fi

# 디렉토리 생성
mkdir -p internal/domain
mkdir -p internal/service
mkdir -p internal/repository
mkdir -p internal/handler

# 기본 파일 생성
cat > internal/domain/${SERVICE_NAME}.go << EOF
package domain

import "context"

type ${SERVICE_NAME^} struct {
    ID   string
    Name string
}

type ${SERVICE_NAME^}Repository interface {
    Create(ctx context.Context, entity *${SERVICE_NAME^}) error
    GetByID(ctx context.Context, id string) (*${SERVICE_NAME^}, error)
}

type ${SERVICE_NAME^}Service interface {
    Create${SERVICE_NAME^}(ctx context.Context, req Create${SERVICE_NAME^}Request) (*${SERVICE_NAME^}, error)
    Get${SERVICE_NAME^}(ctx context.Context, id string) (*${SERVICE_NAME^}, error)
}
EOF

echo "Generated service structure for: $SERVICE_NAME"
```

## 💡 Best Practices

### 1. 한 파일에는 관련된 타입만
```go
// user.go - User 관련 타입만
type User struct { ... }
type CreateUserRequest struct { ... }
type UpdateUserRequest struct { ... }

// auth.go - 인증 관련 타입만
type LoginRequest struct { ... }
type Token struct { ... }
```

### 2. 인터페이스는 사용하는 곳에 정의
```go
// ❌ repository 패키지에 인터페이스 정의
package repository
type UserRepository interface { ... }

// ✅ domain 패키지에 인터페이스 정의
package domain
type UserRepository interface { ... }
```

### 3. 순환 의존성 방지
```go
// ❌ 순환 의존성
package service
import "myproject/internal/handler" // X

package handler  
import "myproject/internal/service" // X

// ✅ 공통 패키지 사용
package domain
type UserService interface { ... }

package service
import "myproject/internal/domain"

package handler
import "myproject/internal/domain"
```

이 구조를 따라 체계적이고 유지보수하기 쉬운 Go 프로젝트를 만들 수 있습니다!