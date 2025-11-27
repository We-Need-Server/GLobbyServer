# Go 패키지 및 기능 학습 노트

이 문서는 프로젝트 진행 중 사용한 Go 패키지와 기능들에 대한 학습 내용을 기록합니다.

## 📚 표준 라이브러리

### `net/http`
HTTP 클라이언트와 서버 구현을 위한 표준 패키지

#### 사용한 기능
- **`http.HandleFunc`**: HTTP 핸들러 등록
- **`http.ListenAndServe`**: HTTP 서버 시작
- **`http.ResponseWriter`**: HTTP 응답 작성
- **`http.Request`**: HTTP 요청 처리

#### 예제
```go
func main() {
    http.HandleFunc("/", homeHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}

func homeHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"message": "Hello, World!"})
}
```

#### 학습 포인트
- `ResponseWriter`는 인터페이스로, 테스트 시 `httptest.ResponseRecorder` 사용 가능
- 미들웨어 패턴: `func(http.Handler) http.Handler` 형태로 체이닝
- 컨텍스트는 `r.Context()`로 접근

#### 참고 자료
- [공식 문서](https://pkg.go.dev/net/http)
- [HTTP 미들웨어 패턴](https://medium.com/@matryer/writing-middleware-in-golang-and-how-go-makes-it-so-much-fun-4375c1246e81)

---

### `context`
고루틴 간 취소, 타임아웃, 값 전달을 위한 패키지

#### 사용한 기능
- **`context.Background()`**: 루트 컨텍스트 생성
- **`context.WithTimeout()`**: 타임아웃 설정
- **`context.WithCancel()`**: 취소 가능한 컨텍스트
- **`context.WithValue()`**: 값을 담은 컨텍스트

#### 예제
```go
// 타임아웃 설정
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// 데이터베이스 쿼리
rows, err := db.QueryContext(ctx, "SELECT * FROM users")
if err != nil {
    if ctx.Err() == context.DeadlineExceeded {
        log.Println("Query timeout")
    }
    return err
}

// 값 전달
ctx = context.WithValue(ctx, "userID", "12345")
userID := ctx.Value("userID").(string)
```

#### 학습 포인트
- 모든 블로킹 작업에 컨텍스트 전달 권장
- `defer cancel()` 호출로 리소스 정리
- 컨텍스트 값은 요청 범위 데이터만 사용 (의존성 주입 X)

#### 참고 자료
- [Go Concurrency Patterns: Context](https://go.dev/blog/context)

---

### `database/sql`
SQL 데이터베이스 연동을 위한 표준 패키지

#### 사용한 기능
- **`sql.Open()`**: 데이터베이스 연결
- **`db.QueryContext()`**: SELECT 쿼리 실행
- **`db.ExecContext()`**: INSERT/UPDATE/DELETE 실행
- **`db.BeginTx()`**: 트랜잭션 시작

#### 예제
```go
// 연결 설정
db, err := sql.Open("postgres", dsn)
if err != nil {
    return nil, err
}

// 연결 풀 설정
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(5)
db.SetConnMaxLifetime(5 * time.Minute)

// 쿼리 실행
func getUserByID(ctx context.Context, db *sql.DB, id string) (*User, error) {
    query := "SELECT id, email, name FROM users WHERE id = $1"
    
    var user User
    err := db.QueryRowContext(ctx, query, id).Scan(&user.ID, &user.Email, &user.Name)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, ErrUserNotFound
        }
        return nil, err
    }
    
    return &user, nil
}

// 트랜잭션
func createUserWithProfile(ctx context.Context, db *sql.DB, user *User) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // 사용자 생성
    _, err = tx.ExecContext(ctx, "INSERT INTO users (id, email) VALUES ($1, $2)", 
        user.ID, user.Email)
    if err != nil {
        return err
    }

    // 프로필 생성
    _, err = tx.ExecContext(ctx, "INSERT INTO profiles (user_id, name) VALUES ($1, $2)", 
        user.ID, user.Name)
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

#### 학습 포인트
- 연결 풀 설정이 성능에 큰 영향
- `QueryRowContext`는 하나의 결과, `QueryContext`는 여러 결과
- SQL injection 방지를 위해 파라미터 바인딩 필수
- 트랜잭션 사용 시 `defer tx.Rollback()` 패턴

---

### `encoding/json`
JSON 인코딩/디코딩을 위한 표준 패키지

#### 사용한 기능
- **`json.Marshal()`**: Go 값을 JSON으로 변환
- **`json.Unmarshal()`**: JSON을 Go 값으로 변환
- **`json.NewEncoder()`**: 스트림 기반 인코딩
- **`json.NewDecoder()`**: 스트림 기반 디코딩

#### 예제
```go
type User struct {
    ID    string `json:"id"`
    Email string `json:"email"`
    Name  string `json:"name,omitempty"`
    Age   int    `json:"age"`
}

// 인코딩
user := User{ID: "1", Email: "user@example.com", Age: 30}
data, err := json.Marshal(user)
if err != nil {
    return err
}

// 디코딩
var user User
err = json.Unmarshal(data, &user)
if err != nil {
    return err
}

// HTTP 요청/응답에서 사용
func createUserHandler(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    // 처리 로직...

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

#### 학습 포인트
- 구조체 태그로 JSON 필드명 제어: `json:"field_name"`
- `omitempty`로 빈 값 제외: `json:"name,omitempty"`
- `json:"-"`로 필드 제외
- 스트림 처리 시 `NewEncoder`/`NewDecoder` 사용

---

### `time`
시간 처리를 위한 표준 패키지

#### 사용한 기능
- **`time.Now()`**: 현재 시간
- **`time.Duration`**: 시간 간격
- **`time.Parse()`**: 문자열을 시간으로 파싱
- **`time.Timer`**: 타이머 설정

#### 예제
```go
// 현재 시간
now := time.Now()
fmt.Println(now.Format("2006-01-02 15:04:05"))

// 시간 간격
duration := 5 * time.Second
deadline := now.Add(duration)

// 시간 비교
if time.Now().After(deadline) {
    fmt.Println("Deadline exceeded")
}

// 타이머
timer := time.NewTimer(5 * time.Second)
select {
case <-timer.C:
    fmt.Println("Timer expired")
case <-ctx.Done():
    timer.Stop()
    fmt.Println("Cancelled")
}

// 주기적 실행
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()

for {
    select {
    case <-ticker.C:
        fmt.Println("Tick")
    case <-ctx.Done():
        return
    }
}
```

#### 학습 포인트
- Go의 시간 포맷은 `2006-01-02 15:04:05` 기준
- `Timer`는 일회성, `Ticker`는 주기적
- 사용 후 반드시 `Stop()` 호출하여 리소스 정리

---

## 🔧 외부 패키지

### `github.com/lib/pq`
PostgreSQL 드라이버

#### 설치
```bash
go get github.com/lib/pq
```

#### 사용법
```go
import (
    "database/sql"
    _ "github.com/lib/pq"  // 드라이버 등록만
)

func main() {
    dsn := "postgres://user:password@localhost/dbname?sslmode=disable"
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
}
```

#### 학습 포인트
- 드라이버는 `_` import로 등록만 함
- DSN 형식: `postgres://user:password@host:port/dbname?options`
- SSL 모드 설정 필수 (`sslmode=disable` 또는 `sslmode=require`)

---

### `github.com/gorilla/mux`
HTTP 라우터 패키지

#### 설치
```bash
go get -u github.com/gorilla/mux
```

#### 사용한 기능
- 경로 변수 추출
- HTTP 메서드 제한
- 미들웨어 체이닝
- 서브라우터

#### 예제
```go
func main() {
    r := mux.NewRouter()
    
    // 경로 변수
    r.HandleFunc("/users/{id}", getUserHandler).Methods("GET")
    r.HandleFunc("/users/{id}", updateUserHandler).Methods("PUT")
    
    // 미들웨어
    r.Use(loggingMiddleware)
    r.Use(authMiddleware)
    
    // 서브라우터
    api := r.PathPrefix("/api/v1").Subrouter()
    api.HandleFunc("/users", listUsersHandler).Methods("GET")
    
    log.Fatal(http.ListenAndServe(":8080", r))
}

func getUserHandler(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID := vars["id"]
    // 처리 로직...
}

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}
```

#### 학습 포인트
- 경로 변수는 `mux.Vars(r)`로 추출
- 미들웨어는 `func(http.Handler) http.Handler` 형태
- 서브라우터로 API 버저닝 가능

#### 대안
- `github.com/go-chi/chi`: 더 가벼운 라우터
- `github.com/gin-gonic/gin`: 완전한 웹 프레임워크

---

### `github.com/stretchr/testify`
테스트 유틸리티 패키지

#### 설치
```bash
go get github.com/stretchr/testify
```

#### 사용한 기능
- **assert**: 테스트 어설션
- **require**: 실패 시 테스트 중단
- **mock**: Mock 객체 생성
- **suite**: 테스트 스위트

#### 예제
```go
func TestUserService_CreateUser(t *testing.T) {
    // assert 사용
    user := &User{Email: "test@example.com"}
    assert.NotNil(t, user)
    assert.Equal(t, "test@example.com", user.Email)
    
    // require 사용 (실패 시 중단)
    err := user.Validate()
    require.NoError(t, err)
    
    // 이후 코드는 err가 nil임이 보장됨
    result := user.Process()
    assert.True(t, result)
}

// Mock 사용
type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) Create(ctx context.Context, user *User) error {
    args := m.Called(ctx, user)
    return args.Error(0)
}

func TestWithMock(t *testing.T) {
    mockRepo := new(MockUserRepository)
    mockRepo.On("Create", mock.Anything, mock.AnythingOfType("*User")).Return(nil)
    
    service := NewUserService(mockRepo)
    err := service.CreateUser(context.Background(), CreateUserRequest{})
    
    assert.NoError(t, err)
    mockRepo.AssertExpectations(t)
}
```

#### 학습 포인트
- `assert` vs `require`: require는 실패 시 테스트 중단
- Mock 설정: `On(method, args...).Return(values...)`
- `mock.Anything`으로 임의 값 매칭
- `AssertExpectations()`로 모든 기대값 호출 확인

---

### `github.com/golang-jwt/jwt/v5`
JWT 토큰 처리 패키지

#### 설치
```bash
go get github.com/golang-jwt/jwt/v5
```

#### 사용한 기능
- JWT 토큰 생성
- JWT 토큰 검증
- Claims 추출

#### 예제
```go
type Claims struct {
    UserID string `json:"user_id"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func generateToken(userID, role string, secretKey []byte) (string, error) {
    claims := Claims{
        UserID: userID,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "myapp",
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secretKey)
}

func validateToken(tokenString string, secretKey []byte) (*Claims, error) {
    claims := &Claims{}
    
    token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
        return secretKey, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if !token.Valid {
        return nil, errors.New("invalid token")
    }
    
    return claims, nil
}
```

#### 학습 포인트
- Claims 구조체는 `jwt.RegisteredClaims` 임베딩
- 서명 알고리즘 확인 필수 (보안)
- 만료 시간 검증 자동 수행

---

## 🎯 Go 관용구 및 패턴

### 에러 처리 패턴

#### 에러 래핑
```go
func processUser(id string) error {
    user, err := getUserFromDB(id)
    if err != nil {
        return fmt.Errorf("failed to get user %s: %w", id, err)
    }
    
    if err := validateUser(user); err != nil {
        return fmt.Errorf("user validation failed: %w", err)
    }
    
    return nil
}

// 에러 체크
err := processUser("123")
if err != nil {
    if errors.Is(err, ErrUserNotFound) {
        // 특정 에러 처리
    }
    log.Printf("Process failed: %v", err)
}
```

#### 사용한 패키지
- `fmt.Errorf`의 `%w` 동사로 에러 래핑
- `errors.Is`로 에러 타입 확인
- `errors.As`로 에러 타입 변환

### 인터페이스 패턴

#### 작은 인터페이스
```go
// ✅ 좋은 예 - 작은 인터페이스
type Reader interface {
    Read(ctx context.Context, id string) (*Entity, error)
}

type Writer interface {
    Write(ctx context.Context, entity *Entity) error
}

// 구성으로 큰 인터페이스 만들기
type ReadWriter interface {
    Reader
    Writer
}
```

#### 사용하는 곳에서 인터페이스 정의
```go
// ❌ 나쁜 예 - repository 패키지에서 정의
package repository
type UserRepository interface { ... }

// ✅ 좋은 예 - service 패키지에서 정의
package service
type UserRepository interface { ... }  // service가 필요한 것만 정의
```

### 고루틴 패턴

#### Worker Pool
```go
func workerPool(jobs <-chan Job, results chan<- Result) {
    const numWorkers = 5
    
    var wg sync.WaitGroup
    
    // 워커 시작
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                result := processJob(job)
                results <- result
            }
        }()
    }
    
    // 모든 워커 완료 대기 후 결과 채널 닫기
    go func() {
        wg.Wait()
        close(results)
    }()
}
```

#### Context를 이용한 취소
```go
func doWork(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // 작업 수행
            if err := performTask(); err != nil {
                return err
            }
        }
    }
}
```

---

## 📝 코딩 팁

### 변수 명명 규칙
```go
// ✅ 좋은 예
var userCount int
var maxRetries = 3
const defaultTimeout = 30 * time.Second

// ❌ 나쁜 예  
var uc int           // 축약어 사용
var MAX_RETRIES = 3  // 대문자 + 언더스코어
```

### 제로값 활용
```go
// ✅ 제로값 활용
var mu sync.Mutex    // 바로 사용 가능
var wg sync.WaitGroup // 바로 사용 가능

// ✅ 제로값 초기화
var users []User     // nil slice, append 가능
if users == nil {
    users = []User{}  // 명시적 초기화는 필요시에만
}
```

### 인터페이스 만족 확인
```go
// 컴파일 타임에 인터페이스 구현 확인
var _ io.Reader = (*MyReader)(nil)
var _ http.Handler = (*MyHandler)(nil)
```

---

## 🔗 유용한 링크

### 공식 문서
- [Go Documentation](https://go.dev/doc/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)

### 추천 도서
- "The Go Programming Language" by Alan Donovan and Brian Kernighan
- "Go in Action" by William Kennedy
- "Concurrency in Go" by Katherine Cox-Buday

### 유용한 도구
- [golangci-lint](https://golangci-lint.run/): 종합 린터
- [go mod](https://go.dev/ref/mod): 모듈 관리
- [air](https://github.com/cosmtrek/air): 핫 리로드
- [delve](https://github.com/go-delve/delve): 디버거

---

**마지막 업데이트**: 2025-07-11  
**다음 업데이트 예정**: 새로운 패키지 사용 시

> 💡 **참고**: 이 문서는 프로젝트 진행 중 지속적으로 업데이트됩니다. 새로운 패키지나 기능을 사용할 때마다 해당 내용을 추가해주세요.