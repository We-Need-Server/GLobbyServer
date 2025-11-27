# Go 프로젝트 초기 설정 가이드

새로운 Go 프로젝트를 시작할 때 따라야 할 단계별 가이드입니다.

> 📚 **코딩 스타일**: 이 프로젝트는 [Uber Go Style Guide](../code-guide/DEV_STYLE.md)를 준수합니다.

## 📋 빠른 시작 체크리스트

```bash
# 1. 프로젝트 디렉토리 생성
mkdir myproject && cd myproject

# 2. Go 모듈 초기화
go mod init github.com/username/myproject

# 3. 기본 디렉토리 구조 생성
mkdir -p cmd/app internal pkg api test scripts docs

# 4. .gitignore 생성
touch .gitignore

# 5. Makefile 생성
touch Makefile

# 6. README.md 생성
touch README.md
```

## 🚀 단계별 상세 가이드

### 1. Go 모듈 초기화

```bash
# 일반적인 형식
go mod init github.com/username/projectname

# 로컬 개발용
go mod init myproject

# 회사 프로젝트
go mod init company.com/team/projectname
```

### 2. 프로젝트 구조 생성

#### 최소 구조 (작은 프로젝트)
```
myproject/
├── go.mod
├── go.sum
├── main.go
├── main_test.go
└── README.md
```

#### 표준 구조 (중간 규모 프로젝트)
```
myproject/
├── cmd/
│   └── app/
│       └── main.go          # 애플리케이션 진입점
├── internal/                # 비공개 패키지
│   ├── config/             # 설정 관리
│   ├── handler/            # HTTP/gRPC 핸들러
│   ├── service/            # 비즈니스 로직
│   └── repository/         # 데이터 액세스
├── pkg/                    # 공개 패키지 (다른 프로젝트에서 사용 가능)
├── api/                    # API 정의 파일
│   ├── openapi/           # OpenAPI/Swagger 정의
│   └── proto/             # Protocol Buffer 정의
├── configs/               # 설정 파일
├── scripts/               # 빌드, 배포 스크립트
├── test/                  # 추가 테스트 (통합, E2E)
├── docs/                  # 프로젝트 문서
├── .gitignore
├── .golangci.yml          # 린터 설정
├── Dockerfile
├── docker-compose.yml
├── Makefile
├── README.md
├── go.mod
└── go.sum
```

### 3. .gitignore 설정

```gitignore
# Binaries for programs and plugins
*.exe
*.exe~
*.dll
*.so
*.dylib

# Test binary, built with `go test -c`
*.test

# Output of the go coverage tool
*.out

# Dependency directories
vendor/

# Go workspace file
go.work

# Environment variables
.env
.env.local

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Project specific
bin/
dist/
*.log
```

### 4. Makefile 템플릿

```makefile
# 변수 정의
BINARY_NAME=myapp
BINARY_DIR=bin
GO_FILES=$(shell find . -name '*.go' -type f)
PACKAGES=$(shell go list ./...)

# Go 관련 변수
GOCMD=go
GOBUILD=$(GOCMD) build
GOCLEAN=$(GOCMD) clean
GOTEST=$(GOCMD) test
GOGET=$(GOCMD) get
GOMOD=$(GOCMD) mod
GOFMT=gofmt
GOLINT=golangci-lint

# 빌드 플래그
LDFLAGS=-ldflags "-s -w"

# 기본 타겟
.DEFAULT_GOAL := help

## help: 사용 가능한 명령어 표시
.PHONY: help
help:
	@echo "사용 가능한 명령어:"
	@sed -n 's/^##//p' ${MAKEFILE_LIST} | column -t -s ':' | sed -e 's/^/ /'

## build: 애플리케이션 빌드
.PHONY: build
build:
	@echo "Building..."
	@mkdir -p $(BINARY_DIR)
	$(GOBUILD) $(LDFLAGS) -o $(BINARY_DIR)/$(BINARY_NAME) cmd/app/main.go

## run: 애플리케이션 실행
.PHONY: run
run: build
	@echo "Running..."
	./$(BINARY_DIR)/$(BINARY_NAME)

## test: 테스트 실행
.PHONY: test
test:
	@echo "Testing..."
	$(GOTEST) -v -race -cover ./...

## test-coverage: 테스트 커버리지 확인
.PHONY: test-coverage
test-coverage:
	@echo "Checking test coverage..."
	$(GOTEST) -v -race -coverprofile=coverage.out ./...
	$(GOCMD) tool cover -html=coverage.out -o coverage.html
	@echo "Coverage report generated: coverage.html"

## bench: 벤치마크 실행
.PHONY: bench
bench:
	@echo "Running benchmarks..."
	$(GOTEST) -bench=. -benchmem ./...

## lint: 린터 실행
.PHONY: lint
lint:
	@echo "Linting..."
	$(GOLINT) run ./...

## fmt: 코드 포맷팅
.PHONY: fmt
fmt:
	@echo "Formatting..."
	$(GOFMT) -s -w .

## vet: go vet 실행
.PHONY: vet
vet:
	@echo "Vetting..."
	$(GOCMD) vet ./...

## deps: 의존성 다운로드
.PHONY: deps
deps:
	@echo "Downloading dependencies..."
	$(GOMOD) download
	$(GOMOD) tidy

## clean: 빌드 파일 정리
.PHONY: clean
clean:
	@echo "Cleaning..."
	$(GOCLEAN)
	rm -rf $(BINARY_DIR)
	rm -f coverage.out coverage.html

## install-tools: 개발 도구 설치
.PHONY: install-tools
install-tools:
	@echo "Installing tools..."
	go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
	go install golang.org/x/tools/cmd/goimports@latest
	go install github.com/golang/mock/mockgen@latest

## docker-build: Docker 이미지 빌드
.PHONY: docker-build
docker-build:
	@echo "Building Docker image..."
	docker build -t $(BINARY_NAME):latest .

## docker-run: Docker 컨테이너 실행
.PHONY: docker-run
docker-run: docker-build
	@echo "Running Docker container..."
	docker run --rm -p 8080:8080 $(BINARY_NAME):latest

## all: 전체 빌드 프로세스
.PHONY: all
all: deps fmt vet lint test build
```

### 5. 기본 main.go 작성

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"
)

func main() {
	if err := run(); err != nil {
		log.Fatalf("application failed: %v", err)
	}
}

func run() error {
	// 컨텍스트 설정
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 시그널 처리
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)

	// 애플리케이션 시작
	fmt.Println("Application started")

	// 종료 대기
	select {
	case <-sigChan:
		fmt.Println("\nShutdown signal received")
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

### 6. 기본 테스트 파일

```go
package main

import (
	"testing"
)

func TestRun(t *testing.T) {
	// 여기에 테스트 로직 작성
	t.Log("Test placeholder")
}
```

### 7. README.md 템플릿

```markdown
# Project Name

프로젝트에 대한 간단한 설명을 여기에 작성합니다.

## 시작하기

### 필요 조건

- Go 1.21 이상
- Make (선택사항)

### 설치

\```bash
# 저장소 클론
git clone https://github.com/username/projectname.git
cd projectname

# 의존성 설치
go mod download
\```

### 빌드 및 실행

\```bash
# 빌드
make build

# 실행
make run

# 또는 직접 실행
go run cmd/app/main.go
\```

### 테스트

\```bash
# 테스트 실행
make test

# 커버리지 확인
make test-coverage
\```

## 프로젝트 구조

\```
.
├── cmd/           # 애플리케이션 진입점
├── internal/      # 비공개 애플리케이션 코드
├── pkg/          # 공개 라이브러리 코드
└── ...
\```

## 기여하기

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 라이선스

이 프로젝트는 MIT 라이선스를 따릅니다. 자세한 내용은 [LICENSE](LICENSE) 파일을 참조하세요.
```

### 8. 개발 도구 설치

```bash
# golangci-lint (종합 린터)
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# goimports (import 자동 정리)
go install golang.org/x/tools/cmd/goimports@latest

# mockgen (mock 생성)
go install github.com/golang/mock/mockgen@latest

# godoc (문서 생성)
go install golang.org/x/tools/cmd/godoc@latest

# air (핫 리로드)
go install github.com/cosmtrek/air@latest
```

### 9. .golangci.yml 설정

```yaml
# .golangci.yml
linters:
  enable:
    - gofmt
    - goimports
    - golint
    - govet
    - errcheck
    - staticcheck
    - ineffassign
    - typecheck
    - gocritic
    - gosimple

linters-settings:
  gofmt:
    simplify: true
  goimports:
    local-prefixes: github.com/username/projectname

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - errcheck

run:
  timeout: 5m
  tests: true
```

### 10. VS Code 설정 (선택사항)

`.vscode/settings.json`:
```json
{
  "go.lintTool": "golangci-lint",
  "go.lintFlags": [
    "--fast"
  ],
  "go.formatTool": "goimports",
  "go.useLanguageServer": true,
  "[go]": {
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": true
    }
  },
  "gopls": {
    "experimentalWorkspaceModule": true,
    "ui.semanticTokens": true
  }
}
```

## 🎯 다음 단계

프로젝트 초기 설정이 완료되면:

1. `1-requirements/REQUIREMENTS.md`에서 요구사항 정의
2. `3-implementation/TDD_GUIDE.md`를 참고하여 테스트 작성
3. `3-implementation/CODE_STRUCTURE.md`를 참고하여 코드 구조화
4. `2-design/ARCHITECTURE.md`를 참고하여 아키텍처 설계

## 💡 팁

- 처음에는 단순하게 시작하고 필요에 따라 구조를 확장하세요
- 표준 라이브러리를 우선적으로 사용하세요
- 외부 의존성은 신중하게 선택하세요
- 테스트를 먼저 작성하는 습관을 들이세요
- `go mod tidy`를 자주 실행하여 의존성을 정리하세요

이 가이드를 따라 Go 프로젝트를 체계적으로 시작할 수 있습니다!