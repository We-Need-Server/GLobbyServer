# Go TDD (테스트 주도 개발) 실천 가이드

Go 프로젝트에서 TDD를 효과적으로 실천하기 위한 가이드입니다.

> 📚 **코딩 스타일**: 이 가이드는 [Uber Go Style Guide](../code-guide/DEV_STYLE.md)를 따릅니다.

## 📋 TDD란?

Test-Driven Development는 다음 사이클을 반복하는 개발 방법론입니다:

1. **Red** 🔴: 실패하는 테스트 작성
2. **Green** 🟢: 테스트를 통과하는 최소한의 코드 작성
3. **Refactor** 🔵: 코드 개선

## 🚀 빠른 시작

### 1. 테스트 파일 생성
```bash
# 구현 파일: calculator.go
# 테스트 파일: calculator_test.go (반드시 _test.go로 끝나야 함)
```

### 2. 첫 번째 테스트 작성 (Red)
```go
package calculator

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5
    
    if result != expected {
        t.Errorf("Add(2, 3) = %d; want %d", result, expected)
    }
}
```

### 3. 최소 구현 (Green)
```go
package calculator

func Add(a, b int) int {
    return a + b
}
```

### 4. 리팩토링 (Refactor)
필요한 경우 코드를 개선합니다.

## 📝 Go 테스트 작성 패턴

### 1. Table-Driven Tests (권장)

```go
func TestCalculate(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        op       string
        expected int
        wantErr  bool
    }{
        {
            name:     "addition",
            a:        2,
            b:        3,
            op:       "+",
            expected: 5,
            wantErr:  false,
        },
        {
            name:     "subtraction",
            a:        5,
            b:        3,
            op:       "-",
            expected: 2,
            wantErr:  false,
        },
        {
            name:     "division by zero",
            a:        5,
            b:        0,
            op:       "/",
            expected: 0,
            wantErr:  true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := Calculate(tt.a, tt.b, tt.op)
            
            if (err != nil) != tt.wantErr {
                t.Errorf("Calculate() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            
            if result != tt.expected {
                t.Errorf("Calculate() = %v, want %v", result, tt.expected)
            }
        })
    }
}
```

### 2. 에러 테스트

```go
func TestDivide(t *testing.T) {
    t.Run("valid division", func(t *testing.T) {
        result, err := Divide(10, 2)
        if err != nil {
            t.Fatalf("unexpected error: %v", err)
        }
        if result != 5 {
            t.Errorf("Divide(10, 2) = %d; want 5", result)
        }
    })
    
    t.Run("division by zero", func(t *testing.T) {
        _, err := Divide(10, 0)
        if err == nil {
            t.Error("expected error for division by zero")
        }
    })
}
```

### 3. 인터페이스 테스트

```go
// 구현해야 할 인터페이스
type Storage interface {
    Save(ctx context.Context, key string, value []byte) error
    Load(ctx context.Context, key string) ([]byte, error)
}

// Mock 구현
type mockStorage struct {
    data map[string][]byte
}

func (m *mockStorage) Save(ctx context.Context, key string, value []byte) error {
    if m.data == nil {
        m.data = make(map[string][]byte)
    }
    m.data[key] = value
    return nil
}

func (m *mockStorage) Load(ctx context.Context, key string) ([]byte, error) {
    value, ok := m.data[key]
    if !ok {
        return nil, errors.New("not found")
    }
    return value, nil
}

// 테스트
func TestService(t *testing.T) {
    storage := &mockStorage{}
    service := NewService(storage)
    
    ctx := context.Background()
    err := service.Process(ctx, "test-key", []byte("test-data"))
    if err != nil {
        t.Fatalf("Process failed: %v", err)
    }
    
    // 검증
    saved, _ := storage.Load(ctx, "test-key")
    if string(saved) != "test-data" {
        t.Errorf("expected %q, got %q", "test-data", string(saved))
    }
}
```

## 🎯 TDD 실천 단계별 가이드

### Step 1: 요구사항을 테스트로 변환

**요구사항**: "사용자는 두 숫자를 더할 수 있어야 한다"

```go
func TestAddTwoNumbers(t *testing.T) {
    // Given
    a := 5
    b := 3
    
    // When
    result := Add(a, b)
    
    // Then
    if result != 8 {
        t.Errorf("Add(%d, %d) = %d; want %d", a, b, result, 8)
    }
}
```

### Step 2: 컴파일 에러 해결

```go
// 최소한의 구현으로 컴파일 에러만 해결
func Add(a, b int) int {
    return 0  // 일단 컴파일만 되도록
}
```

### Step 3: 테스트 통과시키기

```go
func Add(a, b int) int {
    return a + b  // 실제 구현
}
```

### Step 4: 추가 테스트 케이스

```go
func TestAddWithNegativeNumbers(t *testing.T) {
    result := Add(-5, 3)
    if result != -2 {
        t.Errorf("Add(-5, 3) = %d; want -2", result)
    }
}

func TestAddWithZero(t *testing.T) {
    result := Add(0, 5)
    if result != 5 {
        t.Errorf("Add(0, 5) = %d; want 5", result)
    }
}
```

## 🧪 고급 테스트 패턴

### 1. 서브테스트 활용

```go
func TestUserService(t *testing.T) {
    service := NewUserService()
    
    t.Run("Create", func(t *testing.T) {
        t.Run("valid user", func(t *testing.T) {
            user := &User{Name: "John", Email: "john@example.com"}
            err := service.Create(user)
            if err != nil {
                t.Errorf("unexpected error: %v", err)
            }
        })
        
        t.Run("invalid email", func(t *testing.T) {
            user := &User{Name: "John", Email: "invalid"}
            err := service.Create(user)
            if err == nil {
                t.Error("expected error for invalid email")
            }
        })
    })
    
    t.Run("Update", func(t *testing.T) {
        // Update 관련 테스트
    })
}
```

### 2. 테스트 헬퍼 함수

```go
// 테스트 헬퍼 함수는 t.Helper()를 호출해야 함
func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper()
    if got != want {
        t.Errorf("got %v, want %v", got, want)
    }
}

func assertNoError(t *testing.T, err error) {
    t.Helper()
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}

// 사용 예
func TestSomething(t *testing.T) {
    result, err := DoSomething()
    assertNoError(t, err)
    assertEqual(t, result, "expected")
}
```

### 3. 벤치마크 테스트

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(100, 200)
    }
}

// 메모리 할당 확인
func BenchmarkStringConcat(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 10; j++ {
            s += "a"
        }
    }
}
```

### 4. 병렬 테스트

```go
func TestParallel(t *testing.T) {
    t.Parallel() // 이 테스트를 병렬로 실행
    
    tests := []struct {
        name  string
        input int
        want  int
    }{
        {"test1", 1, 2},
        {"test2", 2, 4},
        {"test3", 3, 6},
    }
    
    for _, tt := range tests {
        tt := tt // capture range variable
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // 서브테스트도 병렬로
            result := Double(tt.input)
            if result != tt.want {
                t.Errorf("Double(%d) = %d; want %d", tt.input, result, tt.want)
            }
        })
    }
}
```

## 🔧 테스트 도구 및 명령어

### 기본 테스트 명령어

```bash
# 현재 패키지 테스트
go test

# 모든 패키지 테스트
go test ./...

# Verbose 모드
go test -v

# 특정 테스트만 실행
go test -run TestAdd

# 테스트 커버리지
go test -cover
go test -coverprofile=coverage.out
go tool cover -html=coverage.out

# 레이스 컨디션 검사
go test -race

# 짧은 테스트만 실행
go test -short

# 벤치마크 실행
go test -bench=.
go test -bench=. -benchmem

# 테스트 캐시 무시
go test -count=1
```

### 테스트 구조화

```
project/
├── calculator.go
├── calculator_test.go      # 단위 테스트
├── calculator_bench_test.go # 벤치마크 테스트
└── testdata/              # 테스트 데이터
    └── test_cases.json
```

## 📊 테스트 커버리지 목표

### 커버리지 기준
- **최소**: 70%
- **권장**: 80%
- **이상적**: 90%+

### 커버리지 제외 대상
- 생성된 코드 (mockgen, protoc 등)
- main 함수
- 단순 getter/setter

### 커버리지 향상 팁
```go
// 경계값 테스트
func TestBoundaryValues(t *testing.T) {
    tests := []struct {
        name  string
        input int
        want  bool
    }{
        {"minimum", 0, true},
        {"maximum", 100, true},
        {"below minimum", -1, false},
        {"above maximum", 101, false},
    }
    // ...
}
```

## ✅ TDD 체크리스트

### 테스트 작성 전
- [ ] 요구사항을 명확히 이해했는가?
- [ ] 테스트 케이스를 도출했는가?
- [ ] 엣지 케이스를 고려했는가?

### 테스트 작성 중
- [ ] 테스트가 실패하는 것을 확인했는가? (Red)
- [ ] 하나의 테스트는 하나의 기능만 테스트하는가?
- [ ] 테스트 이름이 명확한가?

### 구현 중
- [ ] 테스트를 통과하는 최소한의 코드를 작성했는가? (Green)
- [ ] 모든 테스트가 통과하는가?
- [ ] 리팩토링이 필요한가? (Refactor)

### 테스트 작성 후
- [ ] 테스트 커버리지가 충분한가?
- [ ] 테스트가 독립적인가?
- [ ] 테스트 실행 시간이 적절한가?

## 💡 Best Practices

### 1. 테스트는 문서다
```go
// 좋은 예: 테스트 이름이 기능을 설명
func TestUserService_CreateUser_WithValidData_ShouldSucceed(t *testing.T) {
    // ...
}

// 나쁜 예: 모호한 테스트 이름
func TestUser1(t *testing.T) {
    // ...
}
```

### 2. AAA 패턴 사용
```go
func TestSomething(t *testing.T) {
    // Arrange (준비)
    service := NewService()
    input := "test data"
    
    // Act (실행)
    result, err := service.Process(input)
    
    // Assert (검증)
    if err != nil {
        t.Errorf("unexpected error: %v", err)
    }
    if result != "expected" {
        t.Errorf("got %q, want %q", result, "expected")
    }
}
```

### 3. 테스트 격리
```go
// 각 테스트는 독립적이어야 함
func TestIndependent(t *testing.T) {
    t.Run("test1", func(t *testing.T) {
        // 이 테스트는 다른 테스트에 영향을 주지 않음
        db := setupTestDB(t)
        defer cleanupTestDB(t, db)
        // ...
    })
}
```

### 4. 명확한 실패 메시지
```go
// 좋은 예
if got != want {
    t.Errorf("Add(%d, %d) returned %d, want %d", a, b, got, want)
}

// 나쁜 예
if got != want {
    t.Error("test failed")
}
```

## 🎓 다음 단계

1. 요구사항을 테스트로 변환하는 연습
2. Mock 패키지 사용법 학습 (`github.com/golang/mock`)
3. 통합 테스트 작성
4. 테스트 자동화 (CI/CD)

TDD는 연습이 필요한 기술입니다. 작은 기능부터 시작하여 점진적으로 복잡한 기능에 적용해보세요!