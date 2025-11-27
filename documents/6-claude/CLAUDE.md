# CLAUDE.md - 최종 작업 지시서

## 1. 프로젝트 목표
게임 사용자를 위한 회원가입 및 로그인 로비 서버를 구현합니다. 이 서버는 다음 두 가지 인터페이스를 제공해야 합니다:
-   **웹 인터페이스**: 사용자가 브라우저를 통해 직접 가입하고 로그인할 수 있는 SSR(서버 사이드 렌더링) 웹사이트.
-   **gRPC 인터페이스**: 내부의 다른 마이크로서비스가 사용자의 인증 토큰을 검증하고 갱신할 수 있는 고성능 API.

## 2. 핵심 설계 문서
모든 구현은 아래의 청사진 문서를 100% 준수해야 합니다.
-   **요구사항 명세서**: `documents/1-requirements/REQUIREMENTS.md`
-   **아키텍처 설계도**: `documents/2-design/ARCHITECTURE.md`
-   **API 설계도**: `documents/2-design/API_DESIGN.md`
-   **테스트 케이스 명세서**: `documents/3-implementation/TEST_CASES.md`

## 3. 구현 단계별 지침

### 1단계: 프로젝트 설정 및 디렉토리 구조 생성
1.  `go mod init <your_module_path>` 명령으로 Go 모듈을 초기화합니다.
2.  `ARCHITECTURE.md`에 명시된 모든 디렉토리(`cmd`, `internal`, `proto`, `templates`, `static`)를 생성합니다.
3.  필요한 Go 패키지를 설치합니다.
    ```bash
    go get github.com/gin-gonic/gin
    go get google.golang.org/grpc
    go get google.golang.org/protobuf/cmd/protoc-gen-go
    go get google.golang.org/grpc/cmd/protoc-gen-go-grpc
    go get github.com/spf13/viper
    go get go.uber.org/zap
    go get github.com/go-sql-driver/mysql
    go getgolang.org/x/crypto/bcrypt
    go get github.com/golang-jwt/jwt/v4
    ```

### 2단계: gRPC 코드 생성
1.  `API_DESIGN.md`에 명시된 내용으로 `proto/auth.proto` 파일을 작성합니다.
2.  아래 명령어를 실행하여 gRPC 서버 및 클라이언트 코드를 생성합니다.
    ```bash
    protoc --go_out=. --go_opt=paths=source_relative \
           --go-grpc_out=. --go-grpc_opt=paths=source_relative \
           proto/auth.proto
    ```

### 3단계: 설정, 로깅, DB 초기화 구현
1.  **설정 (`internal/config`)**: `Viper`를 사용하여 `config.yaml` 파일 또는 환경 변수로부터 DB 연결 정보(DSN), 서버 포트, JWT 비밀 키 등을 읽어오는 로직을 구현합니다.
2.  **로깅 (`internal/logger`)**: `Zap`을 사용하여 JSON 형식의 구조화된 로거를 설정하는 로직을 구현합니다.
3.  **데이터베이스 (`internal/repository`)**: `MySQL` 드라이버를 사용하여 데이터베이스 연결을 관리하고, `users` 테이블에 대한 CRUD 작업을 수행하는 함수들을 구현합니다.

### 4단계: 핵심 비즈니스 로직 구현
1.  **인증 로직 (`internal/auth`)**: 
    -   `bcrypt`를 사용한 비밀번호 해싱 및 검증 함수를 구현합니다.
    -   `jwt`를 사용하여 Access Token 및 Refresh Token을 생성하고 검증하는 함수를 구현합니다.

### 5단계: 서버 및 핸들러 구현
1.  **gRPC 구현 (`internal/handler`, `internal/server`)**: 
    -   `AuthService` gRPC 서비스를 구현합니다. `ValidateToken`과 `RefreshToken` RPC가 `auth`와 `repository` 로직을 호출하여 동작하도록 합니다.
2.  **웹 서버 구현 (`internal/handler`, `internal/server`)**: 
    -   `API_DESIGN.md`에 명시된 각 웹 라우트(`GET /`, `POST /login` 등)에 대한 Gin 핸들러 함수를 구현합니다.
    -   핸들러는 `repository`와 `auth` 로직을 사용하여 비즈니스 로직을 처리하고, 결과를 `templates` 디렉토리의 HTML 파일에 담아 렌더링하거나 리다이렉트합니다.
    -   인증이 필요한 라우트를 보호하기 위한 쿠키 기반 인증 미들웨어를 구현합니다.

### 6단계: 실행 진입점(main) 구현
1.  **`cmd/was/main.go`**: 
    -   설정 및 로거를 초기화합니다.
    -   Gin 웹 서버를 설정하고 실행합니다.
    -   OS 시그널을 감지하여 서버가 정상 종료(Graceful Shutdown)되도록 구현합니다.
2.  **`cmd/rpc/main.go`**: 
    -   설정 및 로거를 초기화합니다.
    -   gRPC 서버를 설정하고 실행합니다.
    -   OS 시그널을 감지하여 서버가 정상 종료(Graceful Shutdown)되도록 구현합니다.

### 7단계: 테스트
1.  `TEST_CASES.md`에 명시된 모든 시나리오를 커버하는 단위 테스트 및 통합 테스트 코드를 작성합니다.
2.  `go test ./...` 명령어를 실행하여 모든 테스트가 통과하는지 확인합니다.

### 8단계: 컨테이너화
1.  Go 애플리케이션을 빌드하고 실행하는 `Dockerfile`을 작성합니다.
2.  Go 앱과 MySQL 데이터베이스를 함께 실행하기 위한 `docker-compose.yml` 파일을 작성합니다.

모든 단계를 순서대로 진행하여 프로젝트를 완성해주십시오.
