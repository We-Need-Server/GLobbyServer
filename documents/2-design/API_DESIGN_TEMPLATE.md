# API 설계 가이드

REST API, gRPC, GraphQL 등 다양한 API 설계를 위한 가이드입니다.

## 📋 API 설계 원칙

### 1. RESTful 설계 원칙
- **Stateless**: 서버는 클라이언트 상태를 저장하지 않음
- **Cacheable**: 응답이 캐시 가능해야 함
- **Uniform Interface**: 일관된 인터페이스
- **Layered System**: 계층화된 시스템

### 2. 리소스 중심 설계
```
# 좋은 예 - 리소스 중심
GET    /api/v1/users           # 사용자 목록 조회
POST   /api/v1/users           # 사용자 생성
GET    /api/v1/users/{id}      # 특정 사용자 조회
PUT    /api/v1/users/{id}      # 사용자 전체 수정
PATCH  /api/v1/users/{id}      # 사용자 부분 수정
DELETE /api/v1/users/{id}      # 사용자 삭제

# 나쁜 예 - 동작 중심
POST   /api/v1/createUser
POST   /api/v1/updateUser
POST   /api/v1/deleteUser
```

## 🚀 REST API 설계

### HTTP 메서드 사용법

| 메서드 | 용도 | 멱등성 | 안전성 | 예시 |
|--------|------|---------|---------|------|
| GET | 조회 | ✅ | ✅ | `GET /users` |
| POST | 생성 | ❌ | ❌ | `POST /users` |
| PUT | 전체 수정 | ✅ | ❌ | `PUT /users/123` |
| PATCH | 부분 수정 | ❌ | ❌ | `PATCH /users/123` |
| DELETE | 삭제 | ✅ | ❌ | `DELETE /users/123` |

### 상태 코드 가이드

```go
// 성공 응답
const (
    StatusOK        = 200  // GET, PUT, PATCH 성공
    StatusCreated   = 201  // POST 성공 (생성됨)
    StatusNoContent = 204  // DELETE 성공 (내용 없음)
)

// 클라이언트 오류
const (
    StatusBadRequest   = 400  // 잘못된 요청
    StatusUnauthorized = 401  // 인증 필요
    StatusForbidden    = 403  // 권한 없음
    StatusNotFound     = 404  // 리소스 없음
    StatusConflict     = 409  // 리소스 충돌
    StatusUnprocessable = 422  // 검증 실패
)

// 서버 오류
const (
    StatusInternalServerError = 500  // 서버 내부 오류
    StatusBadGateway         = 502  // 게이트웨이 오류
    StatusServiceUnavailable = 503  // 서비스 사용 불가
)
```

### Go REST API 구현 예시

```go
// internal/handler/user.go
package handler

import (
    "encoding/json"
    "net/http"
    "strconv"
    
    "github.com/gorilla/mux"
    "myproject/internal/domain"
    "myproject/pkg/response"
)

type UserHandler struct {
    userService domain.UserService
}

func NewUserHandler(userService domain.UserService) *UserHandler {
    return &UserHandler{userService: userService}
}

// GET /api/v1/users
func (h *UserHandler) ListUsers(w http.ResponseWriter, r *http.Request) {
    // 쿼리 파라미터 파싱
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    if page <= 0 {
        page = 1
    }
    
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    if limit <= 0 || limit > 100 {
        limit = 20
    }
    
    filter := domain.UserFilter{
        Page:  page,
        Limit: limit,
        Email: r.URL.Query().Get("email"),
    }
    
    users, total, err := h.userService.ListUsers(r.Context(), filter)
    if err != nil {
        response.Error(w, http.StatusInternalServerError, "Failed to list users", err)
        return
    }
    
    resp := response.ListResponse{
        Data: users,
        Meta: response.Meta{
            Page:  page,
            Limit: limit,
            Total: total,
        },
    }
    
    response.JSON(w, http.StatusOK, resp)
}

// POST /api/v1/users
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        response.Error(w, http.StatusBadRequest, "Invalid request body", err)
        return
    }
    
    // 요청 검증
    if err := req.Validate(); err != nil {
        response.ValidationError(w, err)
        return
    }
    
    user, err := h.userService.CreateUser(r.Context(), req)
    if err != nil {
        if domain.IsValidationError(err) {
            response.ValidationError(w, err)
            return
        }
        response.Error(w, http.StatusInternalServerError, "Failed to create user", err)
        return
    }
    
    response.JSON(w, http.StatusCreated, user)
}

// GET /api/v1/users/{id}
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID := vars["id"]
    
    user, err := h.userService.GetUser(r.Context(), userID)
    if err != nil {
        if domain.IsNotFoundError(err) {
            response.Error(w, http.StatusNotFound, "User not found", err)
            return
        }
        response.Error(w, http.StatusInternalServerError, "Failed to get user", err)
        return
    }
    
    response.JSON(w, http.StatusOK, user)
}

// PUT /api/v1/users/{id}
func (h *UserHandler) UpdateUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID := vars["id"]
    
    var req UpdateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        response.Error(w, http.StatusBadRequest, "Invalid request body", err)
        return
    }
    
    if err := req.Validate(); err != nil {
        response.ValidationError(w, err)
        return
    }
    
    user, err := h.userService.UpdateUser(r.Context(), userID, req)
    if err != nil {
        if domain.IsNotFoundError(err) {
            response.Error(w, http.StatusNotFound, "User not found", err)
            return
        }
        if domain.IsValidationError(err) {
            response.ValidationError(w, err)
            return
        }
        response.Error(w, http.StatusInternalServerError, "Failed to update user", err)
        return
    }
    
    response.JSON(w, http.StatusOK, user)
}

// DELETE /api/v1/users/{id}
func (h *UserHandler) DeleteUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID := vars["id"]
    
    err := h.userService.DeleteUser(r.Context(), userID)
    if err != nil {
        if domain.IsNotFoundError(err) {
            response.Error(w, http.StatusNotFound, "User not found", err)
            return
        }
        response.Error(w, http.StatusInternalServerError, "Failed to delete user", err)
        return
    }
    
    w.WriteHeader(http.StatusNoContent)
}
```

### 요청/응답 DTO 정의

```go
// internal/handler/dto.go
package handler

import (
    "errors"
    "net/mail"
)

type CreateUserRequest struct {
    Email string `json:"email"`
    Name  string `json:"name"`
    Age   int    `json:"age"`
}

func (r CreateUserRequest) Validate() error {
    if r.Email == "" {
        return errors.New("email is required")
    }
    
    if _, err := mail.ParseAddress(r.Email); err != nil {
        return errors.New("invalid email format")
    }
    
    if r.Name == "" {
        return errors.New("name is required")
    }
    
    if r.Age < 0 || r.Age > 150 {
        return errors.New("age must be between 0 and 150")
    }
    
    return nil
}

type UpdateUserRequest struct {
    Name *string `json:"name,omitempty"`
    Age  *int    `json:"age,omitempty"`
}

func (r UpdateUserRequest) Validate() error {
    if r.Name != nil && *r.Name == "" {
        return errors.New("name cannot be empty")
    }
    
    if r.Age != nil && (*r.Age < 0 || *r.Age > 150) {
        return errors.New("age must be between 0 and 150")
    }
    
    return nil
}

type UserResponse struct {
    ID        string    `json:"id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    Age       int       `json:"age"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}
```

### 공통 응답 패키지

```go
// pkg/response/response.go
package response

import (
    "encoding/json"
    "net/http"
)

type Response struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   *ErrorInfo  `json:"error,omitempty"`
    Meta    *Meta       `json:"meta,omitempty"`
}

type ErrorInfo struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Details string `json:"details,omitempty"`
}

type Meta struct {
    Page  int `json:"page"`
    Limit int `json:"limit"`
    Total int `json:"total"`
}

type ListResponse struct {
    Data interface{} `json:"data"`
    Meta Meta        `json:"meta"`
}

func JSON(w http.ResponseWriter, statusCode int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    
    resp := Response{
        Success: statusCode >= 200 && statusCode < 300,
        Data:    data,
    }
    
    json.NewEncoder(w).Encode(resp)
}

func Error(w http.ResponseWriter, statusCode int, message string, err error) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    
    errorInfo := &ErrorInfo{
        Code:    http.StatusText(statusCode),
        Message: message,
    }
    
    if err != nil {
        errorInfo.Details = err.Error()
    }
    
    resp := Response{
        Success: false,
        Error:   errorInfo,
    }
    
    json.NewEncoder(w).Encode(resp)
}

func ValidationError(w http.ResponseWriter, err error) {
    Error(w, http.StatusUnprocessableEntity, "Validation failed", err)
}
```

## 🔌 gRPC API 설계

### Protocol Buffer 정의

```protobuf
// api/proto/user.proto
syntax = "proto3";

package user.v1;

option go_package = "myproject/api/proto/user/v1;userv1";

import "google/protobuf/timestamp.proto";

service UserService {
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
}

message User {
  string id = 1;
  string email = 2;
  string name = 3;
  int32 age = 4;
  google.protobuf.Timestamp created_at = 5;
  google.protobuf.Timestamp updated_at = 6;
}

message CreateUserRequest {
  string email = 1;
  string name = 2;
  int32 age = 3;
}

message CreateUserResponse {
  User user = 1;
}

message GetUserRequest {
  string id = 1;
}

message GetUserResponse {
  User user = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 limit = 2;
  string email_filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
}

message UpdateUserRequest {
  string id = 1;
  optional string name = 2;
  optional int32 age = 3;
}

message UpdateUserResponse {
  User user = 1;
}

message DeleteUserRequest {
  string id = 1;
}

message DeleteUserResponse {
  // empty response
}
```

### gRPC 서버 구현

```go
// internal/grpc/user_server.go
package grpc

import (
    "context"
    
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    userv1 "myproject/api/proto/user/v1"
    "myproject/internal/domain"
)

type UserServer struct {
    userv1.UnimplementedUserServiceServer
    userService domain.UserService
}

func NewUserServer(userService domain.UserService) *UserServer {
    return &UserServer{
        userService: userService,
    }
}

func (s *UserServer) CreateUser(ctx context.Context, req *userv1.CreateUserRequest) (*userv1.CreateUserResponse, error) {
    // 요청 검증
    if req.Email == "" {
        return nil, status.Error(codes.InvalidArgument, "email is required")
    }
    
    if req.Name == "" {
        return nil, status.Error(codes.InvalidArgument, "name is required")
    }
    
    // 서비스 호출
    createReq := domain.CreateUserRequest{
        Email: req.Email,
        Name:  req.Name,
        Age:   int(req.Age),
    }
    
    user, err := s.userService.CreateUser(ctx, createReq)
    if err != nil {
        if domain.IsValidationError(err) {
            return nil, status.Error(codes.InvalidArgument, err.Error())
        }
        return nil, status.Error(codes.Internal, "failed to create user")
    }
    
    // 응답 변환
    return &userv1.CreateUserResponse{
        User: toProtoUser(user),
    }, nil
}

func (s *UserServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    if req.Id == "" {
        return nil, status.Error(codes.InvalidArgument, "id is required")
    }
    
    user, err := s.userService.GetUser(ctx, req.Id)
    if err != nil {
        if domain.IsNotFoundError(err) {
            return nil, status.Error(codes.NotFound, "user not found")
        }
        return nil, status.Error(codes.Internal, "failed to get user")
    }
    
    return &userv1.GetUserResponse{
        User: toProtoUser(user),
    }, nil
}

func toProtoUser(user *domain.User) *userv1.User {
    return &userv1.User{
        Id:        user.ID,
        Email:     user.Email,
        Name:      user.Name,
        Age:       int32(user.Age),
        CreatedAt: timestamppb.New(user.CreatedAt),
        UpdatedAt: timestamppb.New(user.UpdatedAt),
    }
}
```

## 📡 GraphQL API 설계

### 스키마 정의

```graphql
# api/graphql/schema.graphql

type User {
  id: ID!
  email: String!
  name: String!
  age: Int!
  createdAt: String!
  updatedAt: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  createdAt: String!
}

input CreateUserInput {
  email: String!
  name: String!
  age: Int!
}

input UpdateUserInput {
  name: String
  age: Int
}

type Query {
  user(id: ID!): User
  users(page: Int = 1, limit: Int = 20, email: String): UserConnection!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

## 🔐 API 보안

### 인증 미들웨어

```go
// internal/middleware/auth.go
package middleware

import (
    "context"
    "net/http"
    "strings"
    
    "myproject/pkg/jwt"
)

func AuthMiddleware(jwtManager *jwt.Manager) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Authorization 헤더 확인
            authHeader := r.Header.Get("Authorization")
            if authHeader == "" {
                http.Error(w, "Authorization header required", http.StatusUnauthorized)
                return
            }
            
            // Bearer 토큰 추출
            tokenParts := strings.SplitN(authHeader, " ", 2)
            if len(tokenParts) != 2 || tokenParts[0] != "Bearer" {
                http.Error(w, "Invalid authorization header format", http.StatusUnauthorized)
                return
            }
            
            token := tokenParts[1]
            
            // 토큰 검증
            claims, err := jwtManager.ValidateToken(token)
            if err != nil {
                http.Error(w, "Invalid token", http.StatusUnauthorized)
                return
            }
            
            // 컨텍스트에 사용자 정보 추가
            ctx := context.WithValue(r.Context(), "userID", claims.UserID)
            ctx = context.WithValue(ctx, "userRole", claims.Role)
            
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// 옵셔널 인증 (인증이 없어도 진행)
func OptionalAuthMiddleware(jwtManager *jwt.Manager) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            authHeader := r.Header.Get("Authorization")
            if authHeader != "" {
                tokenParts := strings.SplitN(authHeader, " ", 2)
                if len(tokenParts) == 2 && tokenParts[0] == "Bearer" {
                    if claims, err := jwtManager.ValidateToken(tokenParts[1]); err == nil {
                        ctx := context.WithValue(r.Context(), "userID", claims.UserID)
                        r = r.WithContext(ctx)
                    }
                }
            }
            
            next.ServeHTTP(w, r)
        })
    }
}
```

### Rate Limiting

```go
// internal/middleware/ratelimit.go
package middleware

import (
    "net"
    "net/http"
    "sync"
    "time"
    
    "golang.org/x/time/rate"
)

type RateLimiter struct {
    visitors map[string]*rate.Limiter
    mu       sync.RWMutex
    rate     rate.Limit
    burst    int
}

func NewRateLimiter(r rate.Limit, b int) *RateLimiter {
    return &RateLimiter{
        visitors: make(map[string]*rate.Limiter),
        rate:     r,
        burst:    b,
    }
}

func (rl *RateLimiter) getLimiter(ip string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    limiter, exists := rl.visitors[ip]
    if !exists {
        limiter = rate.NewLimiter(rl.rate, rl.burst)
        rl.visitors[ip] = limiter
    }
    
    return limiter
}

func (rl *RateLimiter) Middleware() func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip, _, err := net.SplitHostPort(r.RemoteAddr)
            if err != nil {
                http.Error(w, "Unable to parse IP", http.StatusInternalServerError)
                return
            }
            
            limiter := rl.getLimiter(ip)
            if !limiter.Allow() {
                http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

// 정리 작업 (주기적으로 실행)
func (rl *RateLimiter) cleanupVisitors() {
    for {
        time.Sleep(time.Minute)
        rl.mu.Lock()
        for ip, limiter := range rl.visitors {
            if limiter.Allow() {  // 여유가 있는 경우 제거
                delete(rl.visitors, ip)
            }
        }
        rl.mu.Unlock()
    }
}
```

## 📚 API 문서화

### OpenAPI 스펙 예시

```yaml
# api/openapi/api.yaml
openapi: 3.0.3
info:
  title: User Management API
  description: API for managing users
  version: 1.0.0
  contact:
    name: API Support
    email: support@example.com

servers:
  - url: https://api.example.com/v1
    description: Production server
  - url: https://staging.api.example.com/v1
    description: Staging server

security:
  - bearerAuth: []

paths:
  /users:
    get:
      summary: List users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            minimum: 1
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
        - name: email
          in: query
          schema:
            type: string
            format: email
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserListResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'

    post:
      summary: Create a new user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/BadRequest'
        '422':
          $ref: '#/components/responses/ValidationError'

  /users/{id}:
    parameters:
      - name: id
        in: path
        required: true
        schema:
          type: string
          format: uuid

    get:
      summary: Get user by ID
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          format: uuid
          example: "123e4567-e89b-12d3-a456-426614174000"
        email:
          type: string
          format: email
          example: "user@example.com"
        name:
          type: string
          example: "John Doe"
        age:
          type: integer
          minimum: 0
          maximum: 150
          example: 30
        created_at:
          type: string
          format: date-time
          example: "2023-01-01T00:00:00Z"
        updated_at:
          type: string
          format: date-time
          example: "2023-01-01T00:00:00Z"

    CreateUserRequest:
      type: object
      required:
        - email
        - name
        - age
      properties:
        email:
          type: string
          format: email
          example: "user@example.com"
        name:
          type: string
          minLength: 1
          example: "John Doe"
        age:
          type: integer
          minimum: 0
          maximum: 150
          example: 30

    Error:
      type: object
      properties:
        success:
          type: boolean
          example: false
        error:
          type: object
          properties:
            code:
              type: string
              example: "Bad Request"
            message:
              type: string
              example: "Invalid request parameters"
            details:
              type: string
              example: "email field is required"

  responses:
    BadRequest:
      description: Bad request
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    Unauthorized:
      description: Unauthorized
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    ValidationError:
      description: Validation error
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

## 🧪 API 테스트

### 통합 테스트 예시

```go
// test/integration/user_api_test.go
package integration

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
    
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "myproject/internal/app"
    "myproject/internal/config"
)

func TestUserAPI(t *testing.T) {
    // 테스트용 앱 설정
    cfg := &config.Config{
        Database: config.DatabaseConfig{
            DSN: "postgres://test:test@localhost/test_db",
        },
    }
    
    app := app.New(cfg)
    server := httptest.NewServer(app.Handler())
    defer server.Close()
    
    client := &http.Client{}
    
    t.Run("Create User", func(t *testing.T) {
        reqBody := map[string]interface{}{
            "email": "test@example.com",
            "name":  "Test User",
            "age":   25,
        }
        
        body, _ := json.Marshal(reqBody)
        req, _ := http.NewRequest("POST", server.URL+"/api/v1/users", bytes.NewBuffer(body))
        req.Header.Set("Content-Type", "application/json")
        
        resp, err := client.Do(req)
        require.NoError(t, err)
        defer resp.Body.Close()
        
        assert.Equal(t, http.StatusCreated, resp.StatusCode)
        
        var user map[string]interface{}
        err = json.NewDecoder(resp.Body).Decode(&user)
        require.NoError(t, err)
        
        assert.Equal(t, "test@example.com", user["email"])
        assert.Equal(t, "Test User", user["name"])
        assert.Equal(t, float64(25), user["age"])
        assert.NotEmpty(t, user["id"])
    })
    
    t.Run("Get User", func(t *testing.T) {
        // 먼저 사용자 생성
        userID := createTestUser(t, server.URL, client)
        
        req, _ := http.NewRequest("GET", server.URL+"/api/v1/users/"+userID, nil)
        resp, err := client.Do(req)
        require.NoError(t, err)
        defer resp.Body.Close()
        
        assert.Equal(t, http.StatusOK, resp.StatusCode)
        
        var user map[string]interface{}
        err = json.NewDecoder(resp.Body).Decode(&user)
        require.NoError(t, err)
        
        assert.Equal(t, userID, user["id"])
    })
}

func createTestUser(t *testing.T, baseURL string, client *http.Client) string {
    reqBody := map[string]interface{}{
        "email": "test@example.com",
        "name":  "Test User",
        "age":   25,
    }
    
    body, _ := json.Marshal(reqBody)
    req, _ := http.NewRequest("POST", baseURL+"/api/v1/users", bytes.NewBuffer(body))
    req.Header.Set("Content-Type", "application/json")
    
    resp, err := client.Do(req)
    require.NoError(t, err)
    defer resp.Body.Close()
    
    var user map[string]interface{}
    err = json.NewDecoder(resp.Body).Decode(&user)
    require.NoError(t, err)
    
    return user["id"].(string)
}
```

## 📊 API 모니터링

### 메트릭 수집

```go
// internal/middleware/metrics.go
package middleware

import (
    "net/http"
    "strconv"
    "time"
    
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status_code"},
    )
    
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint", "status_code"},
    )
)

func MetricsMiddleware() func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            
            // Response writer 래핑하여 상태 코드 캡처
            ww := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
            
            next.ServeHTTP(ww, r)
            
            duration := time.Since(start)
            statusCode := strconv.Itoa(ww.statusCode)
            
            httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, statusCode).Inc()
            httpRequestDuration.WithLabelValues(r.Method, r.URL.Path, statusCode).Observe(duration.Seconds())
        })
    }
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

이 가이드를 참고하여 일관되고 확장 가능한 API를 설계하세요!