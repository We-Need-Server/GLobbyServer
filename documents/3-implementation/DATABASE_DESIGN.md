# DATABASE DESIGN

본 문서는 GLobbyServer의 데이터 저장소 설계를 상세히 설명합니다. MySQL과 Redis의 역할을 명확히 구분하고, 각 저장소의 최적 활용 방안을 제시합니다.

---

## 1. 토큰 저장 전략

### 1.1. Access Token vs Refresh Token

| 항목 | Access Token | Refresh Token |
|------|--------------|---------------|
| **형식** | JWT (JSON Web Token) | 랜덤 문자열 (UUID 등) |
| **저장 위치** | ❌ **저장 안 함** (Stateless) | ✅ **Redis** (SHA256 해시) |
| **전달 방식** | HttpOnly 쿠키 / gRPC 메타데이터 | HttpOnly 쿠키 / gRPC 요청 |
| **만료 시간** | 15분 (짧음) | 7일 (김) |
| **검증 방식** | JWT 서명 검증 (로컬) | Redis 조회 필수 |
| **폐기 가능** | ❌ 불가능 (만료까지 유효) | ✅ 가능 (DEL 명령) |
| **자체 검증** | ✅ 가능 (서명 + exp 클레임) | ❌ 불가능 (저장소 조회 필수) |

### 1.2. Access Token을 저장하지 않는 이유

**핵심 원칙**: JWT는 **자체 검증 가능한 토큰**이므로 서버에 저장할 필요가 없습니다.

**장점**:
1. **고성능**: DB 조회 없이 로컬에서 O(1) 검증 가능
2. **Stateless**: 서버가 세션 상태를 저장하지 않아 수평 확장 용이
3. **간단한 검증 로직**:
   ```go
   claims, err := jwt.Parse(accessToken, func(token *jwt.Token) (interface{}, error) {
       return []byte(jwtSecret), nil
   })
   // userID := claims["user_id"]  // JWT 내부에 모든 정보 포함
   ```

**단점 및 해결책**:

| 단점 | 해결책 |
|------|--------|
| 로그아웃 시 즉시 폐기 불가능 | • 만료 시간 짧게 설정 (15분)<br>• Refresh Token 삭제로 갱신 차단<br>• 선택: Redis 블랙리스트 구현 |
| 비밀번호 변경 시 무효화 불가능 | • 모든 Refresh Token 삭제로 전체 로그아웃 강제<br>• 선택: JWT 버전 필드 추가 |
| 탈취 시 만료까지 악용 가능 | • 짧은 만료 시간 (15분)<br>• HTTPS 필수<br>• HttpOnly 쿠키 사용 |

**선택사항: Access Token 블랙리스트**

즉시 폐기가 필요한 경우 Redis 블랙리스트 구현:
```redis
# 로그아웃 시
SET blacklist:{access_token_jti} "1" EX 900  # 15분 TTL

# 검증 시
if redis.Exists("blacklist:" + jti) {
    return ErrTokenRevoked
}
```

⚠️ **주의**: 블랙리스트는 DB 조회를 추가하므로 Stateless 장점이 사라집니다.

---

## 2. 데이터 저장소 역할 분담

### 2.1. MySQL - 영구 데이터 저장소
**역할**: 사용자 계정 정보의 영구 저장
**특징**: ACID 트랜잭션, 관계형 데이터, 디스크 기반

**저장 데이터**:
- 사용자 계정 정보 (username, password, 메타데이터)
- 사용자 프로필 정보
- 감사(Audit) 로그 (선택사항)

**장점**:
- 데이터 영구 보존
- 복잡한 쿼리 지원 (JOIN, GROUP BY 등)
- 트랜잭션 보장

**단점**:
- 디스크 I/O로 인한 상대적으로 느린 속도
- 스케일 아웃 어려움

---

### 2.2. Redis - 임시 데이터 및 세션 저장소
**역할**: Refresh Token의 임시 저장 및 빠른 조회
**특징**: 메모리 기반, Key-Value 저장소, TTL 자동 만료

**저장 데이터**:
- Refresh Token (SHA256 해시)
- User ID → Token 매핑 (다중 기기 관리)

**장점**:
- 초고속 조회 (O(1), 메모리 기반)
- TTL 자동 만료 기능
- 간단한 Key-Value 구조
- 수평 확장 용이 (Redis Cluster)

**단점**:
- 메모리 기반으로 휘발성 (재시작 시 데이터 손실 가능)
- 복잡한 쿼리 불가능

---

## 3. MySQL 스키마 설계

### 3.1. users 테이블

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY COMMENT '사용자 고유 ID',
    username VARCHAR(50) NOT NULL COMMENT '사용자 계정명 (영문, 숫자, _ 허용)',
    hashed_password CHAR(60) NOT NULL COMMENT 'bcrypt 해시 (고정 60자)',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '계정 생성 시각',
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '마지막 수정 시각',
    last_login_at TIMESTAMP NULL DEFAULT NULL COMMENT '마지막 로그인 시각',
    is_active BOOLEAN NOT NULL DEFAULT TRUE COMMENT '계정 활성화 여부',

    -- 인덱스
    UNIQUE KEY uk_username (username),
    KEY idx_is_active (is_active),
    KEY idx_created_at (created_at)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci
  COMMENT='사용자 계정 정보';
```

### 3.2. 필드 설계 근거

| 필드 | 타입 | 근거 |
|------|------|------|
| `id` | `BIGINT UNSIGNED` | 음수 불필요, 범위 2배 확장 (0 ~ 18경) |
| `username` | `VARCHAR(50)` | 일반적인 계정명 길이 충분 (255는 과도) |
| `hashed_password` | `CHAR(60)` | bcrypt는 항상 60자 고정 → CHAR 사용 |
| `created_at` | `TIMESTAMP` | 타임존 자동 변환 (글로벌 서비스 대비) |
| `last_login_at` | `TIMESTAMP NULL` | 로그인 추적 (보안 감사, 휴면 계정 관리) |
| `is_active` | `BOOLEAN` | 소프트 삭제 (계정 비활성화) |

### 3.3. 인덱스 전략

```sql
-- 로그인 시 필수 조회 (가장 빈번)
UNIQUE KEY uk_username (username);

-- 활성 사용자 필터링
KEY idx_is_active (is_active);

-- 가입 통계 조회
KEY idx_created_at (created_at);
```

**조회 성능 예시**:
```sql
-- O(log n) 조회 (uk_username 인덱스 활용)
SELECT id, hashed_password, is_active
FROM users
WHERE username = 'john_doe' AND is_active = TRUE;
```

---

## 4. Redis 데이터 구조 설계

### 4.1. Refresh Token 저장 구조

#### 패턴 1: Token → User ID 매핑
```redis
KEY: refresh_token:{SHA256(token)}
VALUE: {user_id}
TYPE: String
TTL: 604800 seconds (7일)

예시:
SET refresh_token:a1b2c3d4e5... "12345" EX 604800
```

**설계 근거**:
- SHA256 해싱으로 원본 토큰 보호
- 해시 충돌 확률: 거의 0 (2^256)
- TTL로 만료된 토큰 자동 삭제

#### 패턴 2: User ID → Token Set 매핑 (다중 기기 관리)
```redis
KEY: user_tokens:{user_id}
VALUE: SET {token_hash1, token_hash2, ...}
TYPE: Set
TTL: 604800 seconds (7일)

예시:
SADD user_tokens:12345 "a1b2c3d4e5..."
EXPIRE user_tokens:12345 604800
```

**설계 근거**:
- 한 사용자가 여러 기기에서 로그인 가능
- 전체 로그아웃 시 모든 토큰 일괄 삭제 가능
- Set 자료구조로 중복 방지

### 4.2. Redis 주요 연산

| 연산 | Redis 명령어 | 시간 복잡도 |
|------|--------------|-------------|
| **토큰 저장** | `SET refresh_token:{hash} {user_id} EX 604800` | O(1) |
| **토큰 검증** | `GET refresh_token:{hash}` | O(1) |
| **토큰 폐기** | `DEL refresh_token:{hash}` | O(1) |
| **사용자 토큰 추가** | `SADD user_tokens:{user_id} {hash}` | O(1) |
| **전체 토큰 조회** | `SMEMBERS user_tokens:{user_id}` | O(N) |
| **전체 로그아웃** | `DEL user_tokens:{user_id}` + 각 토큰 DEL | O(N) |
| **TTL 확인** | `TTL refresh_token:{hash}` | O(1) |

### 4.3. 실제 사용 예시

#### 로그인 시 (Access Token + Refresh Token 발급)
```go
// 1. Access Token 생성 (JWT) - Redis/MySQL 저장 안 함!
accessToken := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
    "user_id": user.ID,
    "username": user.Username,
    "exp": time.Now().Add(15 * time.Minute).Unix(),
})
accessTokenString, _ := accessToken.SignedString([]byte(jwtSecret))

// 2. Refresh Token 생성 (랜덤 문자열)
refreshToken := generateRandomToken() // "abc123xyz..."
tokenHash := sha256.Sum256([]byte(refreshToken))
hashStr := hex.EncodeToString(tokenHash[:])

// 3. Redis에 Refresh Token만 저장 (7일 TTL)
client.Set(ctx, "refresh_token:"+hashStr, userID, 7*24*time.Hour)
client.SAdd(ctx, "user_tokens:"+userID, hashStr)
client.Expire(ctx, "user_tokens:"+userID, 7*24*time.Hour)

// 4. 쿠키 설정
setCookie("access_token", accessTokenString, 15*60)    // Access Token: 15분
setCookie("refresh_token", refreshToken, 7*24*60*60)   // Refresh Token: 7일
```

#### Access Token 검증 시 (일반 요청)
```go
// Access Token은 Redis/MySQL 조회 없이 로컬 검증!
accessToken := getCookie("access_token")

// JWT 파싱 및 검증
claims, err := jwt.Parse(accessToken, func(token *jwt.Token) (interface{}, error) {
    return []byte(jwtSecret), nil
})

if err != nil {
    return Unauthorized  // 서명 검증 실패
}

// 만료 확인
if claims["exp"].(float64) < float64(time.Now().Unix()) {
    return Unauthorized  // 만료됨
}

// 인증 성공 - userID 추출
userID := claims["user_id"].(float64)
```

#### Refresh Token 검증 시 (gRPC RefreshToken)
```go
// 1. Refresh Token 해싱
tokenHash := sha256.Sum256([]byte(refreshToken))
hashStr := hex.EncodeToString(tokenHash[:])

// 2. Redis에서 User ID 조회
userID, err := client.Get(ctx, "refresh_token:"+hashStr).Result()
if err == redis.Nil {
    return errors.New("invalid or expired refresh token")
}

// 3. 새로운 Access Token 발급
newAccessToken := generateAccessToken(userID)
```

#### 로그아웃 시
```go
// 1. Refresh Token 해싱
tokenHash := sha256.Sum256([]byte(refreshToken))
hashStr := hex.EncodeToString(tokenHash[:])

// 2. Redis에서 삭제
client.Del(ctx, "refresh_token:"+hashStr)
client.SRem(ctx, "user_tokens:"+userID, hashStr)
```

#### 전체 로그아웃 (모든 기기)
```go
// 1. 사용자의 모든 토큰 조회
tokenHashes, _ := client.SMembers(ctx, "user_tokens:"+userID).Result()

// 2. 각 토큰 삭제
for _, hash := range tokenHashes {
    client.Del(ctx, "refresh_token:"+hash)
}

// 3. Set 삭제
client.Del(ctx, "user_tokens:"+userID)
```

---

## 5. 데이터 일관성 및 동기화

### 5.1. MySQL ↔ Redis 동기화 전략

**원칙**: Redis는 독립적인 세션 저장소로 동작하며, MySQL과의 실시간 동기화는 불필요합니다.

**시나리오별 처리**:

| 시나리오 | MySQL | Redis | 동기화 필요 |
|---------|-------|-------|------------|
| 사용자 생성 | INSERT | - | ❌ |
| 로그인 | SELECT + UPDATE last_login_at | SET (Refresh Token) | ❌ |
| 로그아웃 | - | DEL (Refresh Token) | ❌ |
| 토큰 갱신 | - | GET (검증만) | ❌ |
| 사용자 삭제 | DELETE | DEL user_tokens:{user_id} | ⚠️ 필요 |
| 비밀번호 변경 | UPDATE | DEL user_tokens:{user_id} (선택) | ⚠️ 선택 |

**사용자 삭제 시 처리**:
```go
// 1. MySQL에서 사용자 삭제
db.Exec("DELETE FROM users WHERE id = ?", userID)

// 2. Redis에서 해당 사용자의 모든 토큰 삭제
tokenHashes, _ := redis.SMembers(ctx, "user_tokens:"+userID).Result()
for _, hash := range tokenHashes {
    redis.Del(ctx, "refresh_token:"+hash)
}
redis.Del(ctx, "user_tokens:"+userID)
```

### 5.2. Redis 데이터 유실 대응

**문제**: Redis 재시작 시 모든 Refresh Token 손실

**해결 방안**:
1. **AOF(Append Only File) 활성화**: 재시작 시 복구 가능
   ```bash
   redis-server --appendonly yes
   ```
2. **RDB 스냅샷**: 주기적 백업
3. **재로그인 유도**: 토큰 없을 경우 401 Unauthorized 반환

**권장 설정 (docker-compose.yml)**:
```yaml
redis:
  image: redis:7-alpine
  command: redis-server --appendonly yes
  volumes:
    - redis-data:/data
```

---

## 6. 보안 고려사항

### 6.1. Access Token 보안

| 위협 | 대응 방안 |
|------|----------|
| **토큰 탈취** | • HTTPS 필수<br>• HttpOnly 쿠키 사용 (JS 접근 차단)<br>• SameSite=Strict 설정 |
| **XSS 공격** | • HttpOnly 속성으로 스크립트 접근 차단<br>• 입력값 검증 및 이스케이프 |
| **중간자 공격** | • HTTPS/TLS 암호화<br>• Secure 쿠키 속성 |
| **탈취된 토큰 악용** | • 짧은 만료 시간 (15분)<br>• 선택: 블랙리스트 구현 |

### 6.2. Refresh Token 보안

| 위협 | 대응 방안 |
|------|----------|
| **토큰 탈취** | SHA256 해싱 후 저장 (원본 노출 방지) |
| **재사용 공격** | 로그아웃 시 즉시 Redis에서 삭제 |
| **만료되지 않는 토큰** | TTL 7일로 자동 만료 |
| **무제한 생성** | 동일 사용자 최대 N개 제한 (선택) |

### 6.3. Redis 접근 제어

**권장 설정**:
```redis
# redis.conf
requirepass your-strong-password
bind 127.0.0.1 ::1  # 로컬 접근만 허용
maxmemory 256mb
maxmemory-policy allkeys-lru  # 메모리 부족 시 LRU 삭제
```

---

## 7. 성능 최적화

### 7.1. Access Token 검증 성능

**JWT 검증 시간**: ~0.1ms (로컬 검증)
```go
// Redis/MySQL 조회 불필요!
BenchmarkJWTVerify-8    10000000    0.1 ms/op
```

**vs Redis 조회 시간**: ~1-5ms (네트워크 + 조회)
```go
// 매 요청마다 Redis 조회하면 성능 저하
BenchmarkRedisGet-8     1000000     3.2 ms/op
```

**결론**: Access Token을 저장하지 않음으로써 **30배 이상** 빠른 검증 가능

### 7.2. Redis 메모리 관리

**예상 사용량 계산**:
```
- 1개 토큰: 약 200 bytes (키 + 값 + 오버헤드)
- 동시 활성 사용자 10만 명: 20 MB
- 사용자당 3개 기기: 60 MB
```

**권장 설정**:
- **maxmemory**: 256MB ~ 1GB (트래픽에 따라 조정)
- **maxmemory-policy**: allkeys-lru (오래된 토큰 자동 삭제)

### 7.3. MySQL 쿼리 최적화

**인덱스 활용 확인**:
```sql
EXPLAIN SELECT id, hashed_password
FROM users
WHERE username = 'john_doe' AND is_active = TRUE;
```

**예상 결과**:
```
+----+------+---------------+---------+-------+------+
| id | type | possible_keys | key     | rows  | Extra |
+----+------+---------------+---------+-------+------+
|  1 | ref  | uk_username   | uk_username | 1 | Using where |
+----+------+---------------+---------+-------+------+
```

---

## 8. 모니터링 및 운영

### 8.1. Redis 모니터링 지표

```bash
# 메모리 사용량
redis-cli INFO memory

# 키 개수
redis-cli DBSIZE

# 만료 키 통계
redis-cli INFO stats | grep expired_keys

# 히트율
redis-cli INFO stats | grep keyspace_hits
```

### 8.2. MySQL 모니터링 지표

```sql
-- 활성 사용자 수
SELECT COUNT(*) FROM users WHERE is_active = TRUE;

-- 최근 1시간 가입자
SELECT COUNT(*) FROM users
WHERE created_at > NOW() - INTERVAL 1 HOUR;

-- 로그인 활동 (최근 24시간)
SELECT COUNT(*) FROM users
WHERE last_login_at > NOW() - INTERVAL 24 HOUR;
```

---

## 9. 마이그레이션 및 백업

### 9.1. MySQL 백업
```bash
# 전체 백업
mysqldump -u root -p globby > backup.sql

# users 테이블만 백업
mysqldump -u root -p globby users > users_backup.sql
```

### 9.2. Redis 백업
```bash
# RDB 스냅샷 생성
redis-cli SAVE

# AOF 파일 복사
cp /data/appendonly.aof /backup/
```

---

## 10. FAQ

### Q1. Access Token을 왜 저장하지 않나요?
**A**: JWT는 자체 검증이 가능하므로 DB에 저장할 필요가 없습니다. 서명 검증만으로 유효성을 확인할 수 있어 성능이 30배 이상 빠르며, Stateless 아키텍처로 서버 확장이 용이합니다.

### Q2. 로그아웃해도 Access Token이 유효한데 문제 없나요?
**A**:
- Access Token은 15분만 유효하므로 위험이 제한적입니다
- Refresh Token을 Redis에서 삭제하여 갱신을 차단합니다
- 즉시 폐기가 필요하면 Redis 블랙리스트를 구현할 수 있습니다 (성능 tradeoff)

### Q3. Redis 장애 시 서비스 영향은?
**A**: Refresh Token 갱신 실패 → 사용자는 재로그인 필요. Access Token은 여전히 유효하므로 15분간은 서비스 이용 가능.

**A**:
- **Access Token 검증**: 영향 없음 (JWT 로컬 검증)
- **Refresh Token 갱신**: 실패 → 사용자 재로그인 필요
- **현재 로그인 사용자**: Access Token 만료까지(최대 15분) 계속 이용 가능

### Q4. MySQL과 Redis 중 어느 것이 더 중요한가?
**A**: MySQL이 더 중요. Redis 장애 시 재로그인으로 복구 가능하지만, MySQL 장애 시 모든 인증 불가.

### Q5. Refresh Token을 MySQL에 저장하면 안 되는 이유는?
**A**:
- 성능: 매 요청마다 DB 조회 부담
- TTL: 만료 토큰 자동 삭제 기능 없음
- 확장성: Redis Cluster로 수평 확장 용이

### Q6. 사용자가 많아지면 Redis 메모리가 부족하지 않나?
**A**:
- TTL로 자동 삭제되므로 활성 사용자 수만 유지
- maxmemory-policy 설정으로 LRU 삭제
- 필요 시 Redis Cluster로 확장

---

이 설계를 따르면 MySQL과 Redis의 장점을 최대한 활용하여 안전하고 빠른 인증 시스템을 구축할 수 있습니다!
