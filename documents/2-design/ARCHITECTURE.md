# ARCHITECTURE.md

## 1. 시스템 아키텍처 (System Architecture)
본 시스템은 외부 사용자용 웹 인터페이스와 내부 마이크로서비스용 gRPC 인터페이스를 동시에 제공하는 **하이브리드 아키텍처**를 채택한다.

-   **Gin 웹 서버 (WAS)**: 사용자의 HTTP 요청을 받아 회원가입/로그인 등의 페이지를 HTML로 렌더링하고, 폼 데이터를 처리한다. 인증은 `HttpOnly` 쿠키를 통해 관리된다.
-   **gRPC 서버**: 내부 마이크로서비스들의 인증 관련 요청(토큰 검증/갱신)을 gRPC를 통해 처리한다. 내부망 통신을 전제로 하며, 높은 성능과 타입 안정성을 보장한다.

두 서버는 하나의 Go 애플리케이션 내에서 동시에 실행되며, 데이터베이스와 인증 로직 등 핵심 비즈니스 로직을 공유한다.

```
+------------------+      HTTP/S      +--------------------------------+      +------------------+
|                  |                  |       GLobbyServer (Go App)      |      |                  |
|   End User's     +----------------->+                                |      |    Database      |
|     Browser      |   (HTML/Cookie)  |      +-------------------+       |      |    (MySQL)       |
|                  |                  |      | Gin Web Server    |       |      |                  |
+------------------+                  |      | (Port: 8080)      +<----------->+------------------+
                                      |      +-------------------+       |      |                  |
                                      |              | shared logic      |      |   Refresh Token  |
+------------------+      gRPC        |      +-------------------+       |      |      Store       |
|                  |                  |      | gRPC Server       |       |      |                  |
| Other Internal   +----------------->+      | (Port: 50051)     |       +<----------->+------------------+
|  Microservices   |  (Token Auth)    |      +-------------------+       |
|                  |                  |                                |
+------------------+                  +--------------------------------+
```

## 2. 기술 스택 (Technology Stack)
-   **언어**: Go
-   **웹 프레임워크**: Gin
-   **RPC 프레임워크**: gRPC
-   **데이터베이스**: MySQL
-   **설정 관리**: Viper
-   **로깅**: Zap
-   **UI 스타일링**: Bootstrap (CDN 방식)
-   **컨테이너**: Docker, Docker-Compose

## 3. 디렉토리 구조 (Directory Structure)
```
/GLobbyServer
|
├── cmd/
│   ├── rpc/
│   │   └── main.go      # gRPC 서버의 시작점
│   └── was/
│       └── main.go      # WAS (Gin 웹서버)의 시작점
|
├── internal/
│   ├── auth/            # 인증 관련 핵심 로직 (JWT 생성, 파싱 등)
│   ├── handler/         # HTTP 핸들러 및 gRPC 서비스 구현체
│   ├── repository/      # 데이터베이스 처리 로직
│   └── server/          # Gin, gRPC 서버 설정 및 실행 로직
|
├── proto/
│   ├── auth.proto       # gRPC 서비스 정의
│   └── (생성된 .pb.go 파일들)
|
├── templates/           # HTML 템플릿
├── static/              # CSS, JS 파일
|
├── go.mod
└── ...
```

## 4. 데이터베이스 설계 (Database Design)
-   **`users` 테이블**: 사용자 정보를 저장한다.
    -   `id` (BIGINT AUTO_INCREMENT, PK): 사용자 고유 ID
    -   `username` (VARCHAR(255) UNIQUE NOT NULL): 사용자 계정명
    -   `hashed_password` (VARCHAR(255) NOT NULL): `bcrypt`로 해싱된 비밀번호
    -   `created_at` (DATETIME DEFAULT CURRENT_TIMESTAMP): 생성 일시
    -   `updated_at` (DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP): 수정 일시

## 5. 인증 및 인가 (Authentication & Authorization)
-   **웹 플로우 (Cookie-based)**:
    1.  로그인 성공 시, 서버는 Access Token과 Refresh Token을 JWT 형식으로 발급한다.
    2.  두 토큰은 `HttpOnly`, `Secure`, `SameSite=Strict` 속성을 가진 쿠키에 담아 클라이언트에 전달한다.
    3.  이후 모든 웹 요청 시, Gin 미들웨어는 쿠키에 담긴 Access Token을 검증하여 인증을 처리한다.
-   **gRPC 플로우 (Token-based)**:
    1.  **토큰 검증**: 내부 마이크로서비스는 Access Token을 gRPC 요청의 메타데이터에 담아 인증 서버로 전송하고, 서버는 유효성을 검사하여 결과를 반환한다.
    2.  **토큰 갱신**: Access Token이 만료되었을 경우, 마이크로서비스는 기존에 발급받은 Refresh Token을 gRPC 요청에 담아 전송한다. 서버는 Refresh Token의 유효성을 검증한 후 새로운 Access Token을 발급하여 반환한다.

## 6. 핵심 컴포넌트 설계 (Core Component Design)
-   **`repository`**: 데이터베이스에 대한 CRUD(Create, Read, Update, Delete) 연산을 담당한다. 순수한 DB 로직만 포함한다.
-   **`auth`**: JWT 생성, 파싱, 검증 등 인증과 관련된 순수 비즈니스 로직을 담당한다.
-   **`handler`**: HTTP 요청 및 gRPC 요청을 직접 받아 처리하는 계층. `repository`와 `auth` 서비스를 호출하여 요청을 수행하고, 적절한 응답(HTML 또는 gRPC 응답)을 반환한다.
-   **`server`**: Gin 및 gRPC 서버의 인스턴스를 생성하고, 미들웨어와 핸들러를 등록하며, 정상 종료 로직을 포함하여 서버를 실행/종료하는 책임을 가진다.

## 7. 배포 전략 (Deployment Strategy)
-   `docker-compose.yml`을 사용하여 Go 애플리케이션(WAS+gRPC)과 MySQL 데이터베이스를 컨테이너로 함께 실행한다.
-   Go 애플리케이션 컨테이너는 두 개의 포트를 외부에 노출시킨다 (예: `8080` for WAS, `50051` for gRPC).

## 8. 보완 사항 (Additional Considerations)
-   **설정 관리**: `Viper`를 사용하여 `config.yaml` 파일 또는 환경 변수로부터 설정을 로드한다.
-   **구조화된 로깅**: `Zap`을 사용하여 모든 로그를 JSON 형식으로 출력한다.
-   **정상 종료**: OS 시그널(`SIGINT`, `SIGTERM`)을 감지하여 실행 중인 두 서버(Gin, gRPC)가 안전하게 종료되도록 처리한다.
