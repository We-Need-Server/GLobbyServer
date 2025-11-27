# API DESIGN

본 문서는 GLobbyServer의 API 명세를 정의합니다. 시스템은 사용자를 위한 SSR(Server-Side Rendering) 기반의 웹 인터페이스와, 내부 마이크로서비스를 위한 gRPC 인터페이스, 두 가지로 구성됩니다.

---

## 1. 웹 인터페이스 (SSR)

브라우저를 통해 접속하는 일반 사용자를 위한 인터페이스입니다. JSON이 아닌 완전한 HTML 페이지를 렌더링하여 응답하며, 인증은 `HttpOnly` 쿠키를 통해 처리됩니다.

### **`GET /`**
-   **설명**: 로그인 페이지를 보여줍니다. 사용자가 이미 로그인 상태인 경우, `/profile`로 리다이렉트될 수 있습니다.
-   **성공 응답**: `200 OK`
    -   Content-Type: `text/html`
    -   Body: `login.html` 렌더링 결과

### **`POST /login`**
-   **설명**: 사용자 로그인을 처리합니다.
-   **요청 형식**: `application/x-www-form-urlencoded`
    -   `username` (string)
    -   `password` (string)
-   **처리 로직**:
    1.  MySQL에서 사용자 정보 조회 (`username` 기준)
    2.  비밀번호 검증 (bcrypt)
    3.  Access Token (JWT) 생성 (만료: 15분)
    4.  Refresh Token (랜덤 문자열) 생성
    5.  Refresh Token을 SHA256 해싱 후 Redis에 저장
        -   `SET refresh_token:{hash} {user_id} EX 604800` (7일)
        -   `SADD user_tokens:{user_id} {hash}`
-   **성공 응답**: `302 Found`
    -   `Set-Cookie`: `access_token=...; HttpOnly; Secure; SameSite=Strict; Max-Age=900` (15분)
    -   `Set-Cookie`: `refresh_token=...; HttpOnly; Secure; SameSite=Strict; Max-Age=604800` (7일)
    -   `Location`: `/profile`
-   **실패 응답**: `401 Unauthorized`
    -   Body: 에러 메시지를 포함한 `login.html` 렌더링 결과

### **`GET /register`**
-   **설명**: 회원가입 페이지를 보여줍니다.
-   **성공 응답**: `200 OK`
    -   Content-Type: `text/html`
    -   Body: `register.html` 렌더링 결과

### **`POST /register`**
-   **설명**: 신규 사용자 등록을 처리합니다.
-   **요청 형식**: `application/x-www-form-urlencoded`
    -   `username` (string)
    -   `password` (string)
-   **성공 응답**: `302 Found`
    -   `Location`: `/` (로그인 페이지)
-   **실패 응답**: `400 Bad Request` 또는 `409 Conflict`
    -   Body: 에러 메시지를 포함한 `register.html` 렌더링 결과

### **`GET /profile`**
-   **설명**: 로그인된 사용자의 프로필 페이지를 보여줍니다. (인증 필요)
-   **성공 응답**: `200 OK`
    -   Content-Type: `text/html`
    -   Body: 사용자 정보를 포함한 `profile.html` 렌더링 결과
-   **실패 응답 (인증 실패 시)**: `302 Found`
    -   `Location`: `/` (로그인 페이지)

### **`POST /logout`**
-   **설명**: 사용자 로그아웃을 처리합니다.
-   **처리 로직**:
    1.  쿠키에서 `refresh_token` 값 추출
    2.  Refresh Token을 SHA256 해싱
    3.  Redis에서 토큰 삭제
        -   `DEL refresh_token:{hash}`
        -   `SREM user_tokens:{user_id} {hash}`
    4.  쿠키 만료 처리
-   **성공 응답**: `302 Found`
    -   `Set-Cookie`: `access_token=; Max-Age=0` (즉시 만료)
    -   `Set-Cookie`: `refresh_token=; Max-Age=0` (즉시 만료)
    -   `Location`: `/`

---

## 2. gRPC API (내부 마이크로서비스용)

내부 서비스 간의 고성능 통신을 위한 인터페이스입니다.

### Protocol Buffer 정의 (`proto/auth.proto`)

```protobuf
syntax = "proto3";

package auth;

// Go 코드 생성 시 사용할 패키지 경로
option go_package = "./proto";

// 다른 마이크로서비스들이 인증 관련 기능을 호출할 수 있는 서비스
service AuthService {
  // Access Token의 유효성을 검사한다.
  rpc ValidateToken(ValidateTokenRequest) returns (ValidateTokenResponse);

  // Refresh Token을 사용하여 새로운 Access Token을 발급받는다.
  rpc RefreshToken(RefreshTokenRequest) returns (RefreshTokenResponse);
}

// --- ValidateToken RPC 관련 메시지 ---

message ValidateTokenRequest {
  // 검증할 Access Token
  string token = 1;
}

message ValidateTokenResponse {
  // 토큰 유효 여부
  bool is_valid = 1;
  // 유효한 경우, 해당 토큰의 사용자 ID
  string user_id = 2;
}

// --- RefreshToken RPC 관련 메시지 ---

message RefreshTokenRequest {
  // 새로운 Access Token을 발급받기 위한 Refresh Token
  string refresh_token = 1;
}

message RefreshTokenResponse {
  // 새로 발급된 Access Token
  string access_token = 1;
}

// --- RPC 처리 로직 ---
// ValidateToken RPC:
//   1. JWT 서명 검증
//   2. 만료 시각 확인
//   3. user_id 추출 후 반환
//
// RefreshToken RPC:
//   1. Refresh Token을 SHA256 해싱
//   2. Redis에서 조회: GET refresh_token:{hash}
//   3. User ID 조회 성공 시:
//      - 새로운 Access Token (JWT) 생성
//      - 반환
//   4. 조회 실패 시 (토큰 없음 또는 만료):
//      - Unauthenticated 에러 반환
```

---

## 3. API 보안 (API Security)

-   **웹 인터페이스 보안**:
    -   CSRF(Cross-Site Request Forgery) 공격을 방어하기 위해 `SameSite=Strict` 쿠키 정책을 사용합니다.
    -   XSS(Cross-Site Scripting) 공격을 방어하기 위해 `HttpOnly` 쿠키 속성을 사용하여 스크립트가 토큰에 접근하는 것을 막습니다.
    -   중간자 공격을 방어하기 위해 `Secure` 쿠키 속성을 사용하여 HTTPS 연결에서만 쿠키가 전송되도록 합니다.
-   **gRPC 인터페이스 보안**:
    -   gRPC 서버는 내부망에서만 접근 가능하도록 배포하여 외부 노출을 최소화합니다.
    -   필요 시, 상호 TLS(mTLS) 인증을 도입하여 허가된 마이크로서비스 간의 통신만 허용하도록 확장할 수 있습니다.
