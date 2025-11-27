# GLobbyServer - 게임 로비 인증 서버

게임 사용자를 위한 회원가입 및 로그인 기능을 제공하는 로비 서버입니다.

## 📋 프로젝트 개요

### 목적
- **웹 인터페이스**: 브라우저를 통한 SSR 기반 회원가입/로그인
- **gRPC 인터페이스**: 내부 마이크로서비스를 위한 토큰 검증/갱신 API

### 핵심 기술 스택
- **언어**: Go 1.22+
- **웹**: Gin (SSR)
- **RPC**: gRPC
- **데이터베이스**: MySQL 8.0 (사용자 정보)
- **캐시**: Redis 7.x (Refresh Token)
- **인증**: JWT (Access Token) + Redis (Refresh Token)
- **로깅**: Zap
- **설정**: Viper

---

## 🏗️ 아키텍처

### 시스템 구성
```
사용자 브라우저 → Gin 웹서버 (8080) → MySQL + Redis
내부 서비스   → gRPC 서버 (50051) → Redis (토큰 검증)
```

### 토큰 전략
- **Access Token (JWT)**: 15분 만료, 저장 안 함 (Stateless), 로컬 검증
- **Refresh Token**: 7일 만료, Redis 저장 (SHA256 해시), 갱신용

### 디렉토리 구조
```
/GLobbyServer
├── cmd/
│   ├── rpc/main.go          # gRPC 서버 진입점
│   └── was/main.go          # 웹서버 진입점
├── internal/
│   ├── auth/                # JWT, bcrypt 인증 로직
│   ├── config/              # Viper 설정
│   ├── handler/
│   │   ├── grpc/           # gRPC 핸들러
│   │   └── web/            # Gin 핸들러
│   ├── logger/              # Zap 로거
│   ├── repository/
│   │   ├── mysql/          # 사용자 DB
│   │   └── redis/          # Refresh Token
│   └── server/              # 서버 설정
├── proto/                   # gRPC 프로토콜 정의
├── templates/               # HTML 템플릿
├── documents/               # 설계 문서
│   ├── 1-requirements/
│   ├── 2-design/
│   └── 3-implementation/
└── docker-compose.yml
```

---

## 📖 문서 구조

### 설계 문서 (documents/)
모든 구현은 아래 문서를 기반으로 진행됩니다:

| 문서 | 경로 | 설명 |
|------|------|------|
| **요구사항** | `1-requirements/REQUIREMENTS.md` | 기능/비기능 요구사항 (FR-001~006, NFR-001~004) |
| **아키텍처** | `2-design/ARCHITECTURE.md` | 시스템 구조, 기술 스택, 데이터 저장소 설계 |
| **API 설계** | `2-design/API_DESIGN.md` | 웹 라우트, gRPC RPC 정의 |
| **테스트 케이스** | `3-implementation/TEST_CASES.md` | TC-001~025 (웹, gRPC, Redis 테스트) |
| **DB 설계** | `3-implementation/DATABASE_DESIGN.md` | MySQL vs Redis 역할, 키 패턴, 성능 최적화 |

### 핵심 설계 결정
- **Access Token을 저장하지 않는 이유**: JWT 자체 검증으로 30배 빠른 성능 (0.1ms vs 3.2ms)
- **Refresh Token을 Redis에 저장**: TTL 자동 만료, O(1) 조회, 즉시 폐기 가능
- **MySQL vs Redis 분리**: 영구 데이터 vs 임시 세션 저장소

---

## 🚀 빠른 시작

### 1. 사전 준비
```bash
# Go 1.22 이상 설치 필요
go version

# Protocol Buffers 컴파일러 설치
brew install protobuf  # macOS
# apt install protobuf-compiler  # Ubuntu
```

### 2. 의존성 설치
```bash
go mod download
go get github.com/gin-gonic/gin
go get google.golang.org/grpc
go get google.golang.org/protobuf/cmd/protoc-gen-go
go get google.golang.org/grpc/cmd/protoc-gen-go-grpc
go get github.com/spf13/viper
go get go.uber.org/zap
go get github.com/go-sql-driver/mysql
go get golang.org/x/crypto/bcrypt
go get github.com/golang-jwt/jwt/v4
go get github.com/redis/go-redis/v9
```

### 3. gRPC 코드 생성
```bash
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       proto/auth.proto
```

### 4. Docker Compose로 실행
```bash
docker-compose up -d
```

서비스 접속:
- **웹**: http://localhost:8080
- **gRPC**: localhost:50051
- **MySQL**: localhost:3306
- **Redis**: localhost:6379

---

## 🔄 개발 워크플로우 (TDD)

### TDD 사이클
1. **RED**: 실패하는 테스트 작성 → 커밋
2. **GREEN**: 최소 구현으로 테스트 통과 → 커밋
3. **REFACTOR**: 코드 개선 → 테스트 재확인 → 커밋
4. **문서화**: 진행 상황 기록
5. **컨텍스트 정리**: `/compact` 실행

### Git 커밋 규칙
```bash
# 테스트 추가
git commit -m "test: add failing test for user login"

# 기능 구현
git commit -m "feat: implement JWT token generation"

# 리팩토링
git commit -m "refactor: improve password validation logic"

# 문서 업데이트
git commit -m "docs: update DATABASE_DESIGN.md with Redis patterns"
```

---

## 📝 진행 상황

### ✅ 완료된 작업 (Phase 1: 설계)

#### 문서 작성 완료
- [x] `REQUIREMENTS.md` - 요구사항 정의 (FR-001~006, NFR-001~004)
- [x] `ARCHITECTURE.md` - 하이브리드 아키텍처 설계 (Gin + gRPC + MySQL + Redis)
- [x] `API_DESIGN.md` - 웹 라우트 및 gRPC RPC 정의
- [x] `TEST_CASES.md` - 25개 테스트 케이스 (TC-001~025)
- [x] `DATABASE_DESIGN.md` - MySQL vs Redis 역할 분담, 토큰 저장 전략

#### 핵심 설계 결정 완료
- [x] **토큰 전략 수립**:
  - Access Token (JWT): 저장 안 함, 15분 만료, Stateless 검증
  - Refresh Token: Redis 저장 (SHA256 해시), 7일 TTL, 즉시 폐기 가능
- [x] **데이터 저장소 분리**:
  - MySQL: 사용자 계정 정보 영구 저장
  - Redis: Refresh Token 임시 저장 (자동 만료)
- [x] **Redis 키 패턴 설계**:
  - `refresh_token:{hash}` → user_id (O(1) 조회)
  - `user_tokens:{user_id}` → SET (다중 기기 관리)

#### 문서 최적화
- [x] Redis 기반 Refresh Token 관리 아키텍처 반영
- [x] Access Token Stateless 검증 방식 명시
- [x] 성능 벤치마크 결과 포함 (JWT 0.1ms vs Redis 3.2ms)
- [x] 보안 고려사항 및 해결책 문서화

### ⏳ 진행 예정 작업 (Phase 2: 구현)

#### 1단계: 프로젝트 기초 설정
- [ ] 디렉토리 구조 생성 (`cmd`, `internal`, `proto`, `templates`)
- [ ] `go.mod` 초기화 및 패키지 설치
- [ ] `config.yaml` 템플릿 작성
- [ ] `docker-compose.yml` 설정 (MySQL + Redis)

#### 2단계: gRPC 프로토콜
- [ ] `proto/auth.proto` 작성
- [ ] protoc 컴파일러로 Go 코드 생성

#### 3단계: 인프라 레이어 (TDD)
- [ ] Config 모듈 (Viper)
- [ ] Logger 모듈 (Zap)
- [ ] MySQL Repository (사용자 CRUD)
- [ ] Redis Repository (Refresh Token 관리)

#### 4단계: 인증 로직 (TDD)
- [ ] Password 모듈 (bcrypt)
- [ ] JWT 모듈 (Access/Refresh Token)

#### 5단계: 서버 및 핸들러 (TDD)
- [ ] gRPC 서버 (ValidateToken, RefreshToken)
- [ ] 웹 서버 (Gin 라우터, 핸들러)
- [ ] HTML 템플릿 (login, register, profile)

#### 6단계: 통합 테스트
- [ ] TC-001~025 검증
- [ ] 테스트 커버리지 확인

#### 7단계: 컨테이너화
- [ ] Dockerfile 작성
- [ ] docker-compose.yml 완성
- [ ] 배포 테스트

---

## 🧪 테스트

### 테스트 실행
```bash
# 전체 테스트
go test ./...

# 커버리지 확인
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# 특정 패키지만
go test ./internal/auth/...
```

### 테스트 범위
- **단위 테스트**: 각 모듈 독립 검증
- **통합 테스트**: MySQL + Redis 연동 검증
- **E2E 테스트**: 전체 플로우 검증 (로그인 → 인증 → 갱신 → 로그아웃)

---

## 🔧 설정

### 환경 변수
```bash
# 서버
SERVER_WAS_PORT=8080
SERVER_RPC_PORT=50051

# MySQL
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=root
DB_NAME=globby

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0

# JWT
JWT_SECRET=your-secret-key-change-in-production
JWT_ACCESS_TTL=15m
JWT_REFRESH_TTL=168h  # 7 days
```

---

## 📚 참고 자료

### 설계 원칙
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Project Layout](https://github.com/golang-standards/project-layout)

### 학습 자료
- [TDD in Go](https://quii.gitbook.io/learn-go-with-tests/)
- [gRPC Go Tutorial](https://grpc.io/docs/languages/go/quickstart/)
- [Redis Go Client](https://redis.io/docs/clients/go/)

---

## 🤝 기여 가이드

### 개발 절차
1. 이슈 생성 또는 기능 요청
2. Feature 브랜치 생성 (`feature/login-api`)
3. TDD 사이클로 구현
4. 테스트 통과 확인
5. PR 생성 및 코드 리뷰
6. Main 브랜치 병합

### 코드 품질 기준
- [ ] 모든 테스트 통과 (`go test ./...`)
- [ ] 테스트 커버리지 80% 이상
- [ ] `go fmt` 포맷팅 완료
- [ ] 린터 경고 없음

---

## 📄 라이선스

MIT License

---

## 📞 문의

프로젝트 관련 문의사항이 있으시면 이슈를 생성해주세요.

---

**버전**: 0.1.0 (설계 완료, 구현 대기)
**최종 업데이트**: 2025-01-28
**상태**: 🟡 Phase 1 완료 (설계) → Phase 2 시작 예정 (구현)
