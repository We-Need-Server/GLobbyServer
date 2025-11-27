# Go 프로젝트 템플릿 사용 가이드

이 템플릿은 Go 프로젝트를 체계적으로 시작하고 Claude Code와 효과적으로 협업하기 위한 문서 모음입니다.

## 📋 템플릿 개요

### 목적
- **개발자**: Go 프로젝트 시작 시 참고할 체계적인 가이드 제공
- **Claude Code**: AI가 Go 프로젝트를 효과적으로 구현할 수 있도록 돕는 지침서

### 핵심 철학
- **체계적 문서화**: 모든 단계에서 철저한 문서 작성
- **TDD 중심 개발**: 테스트 우선 개발 방법론 준수
- **단계별 접근**: 요구사항 → 테스트 → 아키텍처 → 구현 순서
- **지식 기록**: 개발 과정에서 습득한 지식의 체계적 문서화

## 🚀 프로젝트 개발 워크플로우

### 전체 개발 프로세스 (7단계)

#### 1단계: 요구사항 정의
```markdown
📂 documents/1-requirements/REQUIREMENTS.md
```
- 프로젝트 목적 및 범위 명확히 정의
- 기능 요구사항 및 비기능 요구사항 작성
- 인수 조건 및 제약사항 명시
- 우선순위 및 일정 계획 수립

#### 2단계: 테스트 케이스 작성
```markdown
📂 프로젝트 루트/테스트 파일들
```
- 요구사항을 바탕으로 테스트 케이스 설계
- 입력값, 예상 출력, 예외 상황 정의
- 통합 테스트 시나리오 작성
- 성능 테스트 기준 설정

#### 3단계: 아키텍처 결정
```markdown
📂 documents/2-design/ARCHITECTURE.md
📂 documents/2-design/API_DESIGN.md
```
- 테스트 케이스 분석을 통한 아키텍처 패턴 선택
- 시스템 구성 요소 및 의존성 정의
- API 설계 (필요한 경우)
- 기술 스택 선택 및 근거 문서화

#### 4단계: 소스코드 문서화
```markdown
📂 documents/3-implementation/CODE_STRUCTURE.md
📂 documents/3-implementation/PROJECT_SETUP.md
```
- 전체 프로젝트 구조 설계
- 각 폴더별 역할 및 포함될 파일 정의
- 파일별 책임 및 인터페이스 명세
- 패키지 의존성 관계 문서화

#### 5단계: 구현 계획 및 Git 컨벤션
```markdown
📂 documents/4-progress/DEVELOPMENT_LOG.md
📂 documents/6-claude/GIT_PACKAGE_MANAGEMENT.md
```
- 개발 일정 및 마일스톤 설정
- Git 브랜치 전략 및 커밋 메시지 규칙 정의
- 코드 리뷰 프로세스 수립
- 배포 전략 계획

#### 6단계: CLAUDE.md 생성 및 수정
```bash
# Claude Code에서 /init 명령어 실행
/init
```
- 프로젝트별 맞춤형 CLAUDE.md 생성
- 코딩 스타일 및 개발 원칙 정의
- 사용할 Go 패키지 및 라이브러리 명시
- 품질 기준 및 체크리스트 작성

#### 7단계: 구현 및 버그 수정
```markdown
📂 TDD 사이클 반복 실행
```
- TDD 사이클을 통한 기능 구현
- 지속적인 테스트 및 버그 수정
- 리팩토링 및 코드 품질 개선
- 문서 업데이트 및 학습 내용 기록

## 🔄 TDD 구현 가이드라인

### TDD 사이클 (9단계 + 컨텍스트 관리)

#### 1단계: 실패하는 테스트 작성 (RED)
```go
func TestUserCreation(t *testing.T) {
    // 실패할 테스트 케이스 작성
    user := NewUser("test@example.com", "password123")
    assert.NotNil(t, user)
    assert.Equal(t, "test@example.com", user.Email)
}
```

#### 2단계: 테스트 실패 확인 및 커밋
```bash
go test ./...
# 실패 확인 후
git add .
git commit -m "test: add failing test for user creation feature"
```

#### 3단계: 테스트 통과를 위한 기능 구현 (GREEN)
```go
type User struct {
    Email    string
    Password string
}

func NewUser(email, password string) *User {
    return &User{
        Email:    email,
        Password: password,
    }
}
```

#### 4단계: 테스트 통과 확인 및 구현 커밋
```bash
go test ./...
# 성공 확인 후
git add .
git commit -m "feat: implement basic user creation functionality"
```

#### 5단계: 코드 개선 및 리팩토링 (REFACTOR)
```go
func NewUser(email, password string) (*User, error) {
    if email == "" {
        return nil, errors.New("email is required")
    }
    if len(password) < 8 {
        return nil, errors.New("password must be at least 8 characters")
    }
    return &User{
        Email:    email,
        Password: password,
    }, nil
}
```

#### 6단계: 개선 내용 확인 및 커밋
```bash
go test ./...
# 모든 테스트 통과 확인 후
git add .
git commit -m "refactor: add validation to user creation"
```

#### 7단계: TDD 과정에서 사용된 지식 기록
```markdown
📂 documents/5-learning/LEARNING_NOTES.md
- Go 에러 처리 패턴 학습
- testify 라이브러리 활용법
- 구조체 생성자 패턴
```

#### 8단계: 개발 진행 상황 기록
```markdown
📂 documents/4-progress/DEVELOPMENT_LOG.md
### 2025-07-11 (금)
#### ✅ 완료된 작업
- [x] User 도메인 모델 TDD 구현 완료
  - 테스트 케이스 3개 작성 및 통과
  - 기본 검증 로직 구현
  - 리팩토링을 통한 에러 처리 개선
```

#### 9단계: 다음 목표로 진행
- 다음 기능의 요구사항 확인
- 새로운 TDD 사이클 시작 준비

#### 10단계: 컨텍스트 관리 (/compact 실행)
```bash
# Claude Code에서 하나의 TDD 사이클 완료 후 실행
/compact
```
**중요**: 하나의 TDD 사이클이 완료될 때마다 반드시 `/compact`를 실행하여:
- 대화 컨텍스트 정리 및 최적화
- 메모리 사용량 감소
- 응답 속도 향상
- 다음 작업에 대한 집중도 개선

## 📁 문서 구조 및 활용 방법

### 1단계: 요구사항 정의 (documents/1-requirements/)
```
📂 1-requirements/
├── REQUIREMENTS_TEMPLATE.md     # 요구사항 작성 템플릿
└── REQUIREMENTS.md             # 실제 프로젝트 요구사항
```

**활용 방법:**
- 템플릿을 복사하여 프로젝트 요구사항 작성
- 기능별로 우선순위와 인수 조건 명시
- Claude Code가 이해하기 쉽도록 구체적으로 작성

### 2단계: 설계 (documents/2-design/)
```
📂 2-design/
├── ARCHITECTURE_TEMPLATE.md    # 아키텍처 설계 템플릿
├── API_DESIGN_TEMPLATE.md      # API 설계 템플릿
├── ARCHITECTURE.md             # 실제 아키텍처 문서
└── API_DESIGN.md              # 실제 API 설계 문서
```

**활용 방법:**
- 프로젝트 규모에 맞는 아키텍처 패턴 선택
- ADR(Architecture Decision Record) 형식으로 설계 결정 기록
- API가 필요한 경우 REST/gRPC 설계 지침 참고

### 3단계: 구현 (documents/3-implementation/)
```
📂 3-implementation/
├── PROJECT_SETUP.md            # 프로젝트 초기 설정
├── TDD_GUIDE.md               # TDD 실천 가이드
├── CODE_STRUCTURE.md          # 코드 구조 가이드
└── 실제 소스코드 폴더 구조 문서
```

**활용 방법:**
- `PROJECT_SETUP.md` 따라 프로젝트 기반 구조 생성
- TDD 사이클로 기능 구현
- 각 폴더별 역할과 파일 목록 문서화

### 4단계: 진행 관리 (documents/4-progress/)
```
📂 4-progress/
├── DEVELOPMENT_LOG_TEMPLATE.md # 개발 진행 로그 템플릿
└── DEVELOPMENT_LOG.md         # 실제 개발 진행 로그
```

**활용 방법:**
- 일별/주별 진행 상황 기록
- 이슈 및 해결 방법 문서화
- 성과 지표 및 학습 내용 추적

### 5단계: 학습 기록 (documents/5-learning/)
```
📂 5-learning/
├── LEARNING_NOTES_TEMPLATE.md  # 학습 노트 템플릿
└── LEARNING_NOTES.md          # 실제 학습 노트
```

**활용 방법:**
- 새로운 Go 패키지 사용 시 내용 추가
- TDD 과정에서 배운 지식 기록
- 팀 학습 자료로 활용

### 6단계: Claude 협업 (documents/6-claude/)
```
📂 6-claude/
├── CLAUDE_MD_TEMPLATE.md       # CLAUDE.md 생성 템플릿
├── CONTEXT_MANAGEMENT.md       # 컨텍스트 관리 가이드
├── DOCUMENTATION_WORKFLOW.md   # 문서화 워크플로우
└── GIT_PACKAGE_MANAGEMENT.md   # Git 및 패키지 관리
```

**활용 방법:**
- `/init` 명령어로 프로젝트별 CLAUDE.md 생성
- 컨텍스트 관리 규칙 숙지
- 체계적인 문서화 워크플로우 적용

## 🤖 Claude Code 협업 가이드

### CLAUDE.md 생성 및 활용

#### 1. /init 명령어 사용
```bash
# Claude Code에서 실행
/init
```
- 프로젝트 정보를 바탕으로 맞춤형 CLAUDE.md 생성
- Go 프로젝트 특성에 맞는 개발 지침 자동 생성
- 필요에 따라 수정 및 보완

#### 2. 프로젝트별 맞춤화
```markdown
# 생성된 CLAUDE.md에 추가할 내용
- 프로젝트 특성에 맞는 아키텍처 패턴
- 사용할 Go 패키지 및 라이브러리
- 코딩 스타일 및 네이밍 규칙
- 품질 기준 및 성능 목표
```

### 효과적인 협업 방법

#### 단계별 요청 방식
```markdown
# 1단계: 요구사항 기반 테스트 작성
"documents/1-requirements/REQUIREMENTS.md의 FR-001 요구사항을 
분석하여 테스트 케이스를 작성해주세요."

# 2단계: 아키텍처 결정
"작성된 테스트 케이스를 바탕으로 적절한 아키텍처 패턴을 
제안하고 documents/2-design/ARCHITECTURE.md에 기록해주세요."

# 3단계: TDD 구현
"TDD 사이클을 따라 [기능명]을 구현해주세요. 
각 단계마다 커밋하고 완료 후 /compact를 실행해주세요."

# 4단계: 문서화
"구현 완료된 기능을 DEVELOPMENT_LOG.md에 기록하고,
새로 사용한 패키지들을 LEARNING_NOTES.md에 추가해주세요."
```

### 컨텍스트 관리 규칙

#### /compact 실행 시점
- ✅ 하나의 TDD 사이클 완료 후 (RED → GREEN → REFACTOR)
- ✅ 주요 기능 구현 완료 후
- ✅ 문서화 작업 완료 후
- ✅ 복잡한 디버깅 세션 완료 후

#### 컨텍스트 관리 효과
- 응답 속도 향상 (2-3배)
- 메모리 사용량 감소 (70% 이상)
- 집중도 개선 및 품질 향상

## 💡 권장 워크플로우

### 새 프로젝트 시작 시

#### 1. 템플릿 설정
```bash
# 1. 템플릿 복사
cp -r golang-project-template my-new-project
cd my-new-project

# 2. Git 초기화
git init
git add .
git commit -m "initial: add golang project template"

# 3. 기존 README 백업
mv README.md TEMPLATE_README.md
```

#### 2. 요구사항 정의
```bash
# 1. 요구사항 문서 작성
cp documents/1-requirements/REQUIREMENTS_TEMPLATE.md \
   documents/1-requirements/REQUIREMENTS.md

# 2. 프로젝트 요구사항 작성
# 3. 커밋
git add documents/1-requirements/REQUIREMENTS.md
git commit -m "docs: add project requirements"
```

#### 3. 테스트 케이스 설계
```bash
# 1. 테스트 파일 생성
mkdir -p tests
# 2. 요구사항 기반 테스트 케이스 작성
# 3. 커밋
git add tests/
git commit -m "test: add test cases based on requirements"
```

### TDD 사이클 진행 시

#### 전체 과정 예시
```bash
# 1. RED: 실패하는 테스트 작성
echo "func TestUserCreation(t *testing.T) { ... }" > user_test.go
go test ./...  # 실패 확인
git add user_test.go
git commit -m "test: add failing test for user creation"

# 2. GREEN: 최소 구현
echo "func NewUser() *User { ... }" > user.go
go test ./...  # 성공 확인
git add user.go
git commit -m "feat: implement basic user creation"

# 3. REFACTOR: 개선
# 코드 개선 후
go test ./...  # 모든 테스트 통과 확인
git add .
git commit -m "refactor: improve user creation with validation"

# 4. 문서화
# DEVELOPMENT_LOG.md 업데이트
# LEARNING_NOTES.md 업데이트
git add documents/
git commit -m "docs: update development log and learning notes"

# 5. 컨텍스트 정리
# Claude Code에서 /compact 실행
```

## 🎯 성공 기준

이 템플릿을 잘 활용하고 있다면:

### 문서화 측면
- [ ] 모든 단계에서 적절한 문서가 작성됨
- [ ] 요구사항이 명확하고 구체적으로 정의됨
- [ ] 아키텍처 결정 사항이 문서화됨
- [ ] 개발 진행 상황이 체계적으로 기록됨

### 개발 프로세스 측면
- [ ] 모든 기능이 TDD로 구현됨
- [ ] 각 TDD 사이클마다 적절한 커밋이 이루어짐
- [ ] 테스트 커버리지가 높게 유지됨
- [ ] 코드 품질이 일관되게 관리됨

### 학습 및 성장 측면
- [ ] 새로운 기술 학습이 문서화됨
- [ ] 문제 해결 과정이 기록됨
- [ ] 팀 지식이 체계적으로 축적됨
- [ ] Claude Code와의 협업이 원활함

### 컨텍스트 관리 측면
- [ ] TDD 사이클 완료 후 /compact 실행
- [ ] 응답 속도 및 품질 개선 확인
- [ ] 메모리 효율성 향상 확인

## 🔧 프로젝트 유형별 가이드

### 웹 API 프로젝트
```markdown
1. REST API 요구사항 정의
2. API 엔드포인트 설계
3. 레이어드 아키텍처 적용
4. 데이터베이스 연동 구현
5. 인증/인가 시스템 구현
```

### CLI 도구 프로젝트
```markdown
1. 명령어 인터페이스 정의
2. 사용자 시나리오 기반 테스트
3. 단순한 아키텍처 적용
4. 명령어 파싱 및 실행 구현
5. 사용자 경험 개선
```

### 라이브러리 프로젝트
```markdown
1. 공개 API 요구사항 정의
2. 사용 사례 기반 테스트
3. 인터페이스 중심 설계
4. 핵심 기능 구현
5. 예제 및 문서 작성
```

## 📚 권장 도구 및 자료

### 개발 도구
- **IDE**: Goland
- **린터**: go fmt
- **테스팅**: go test
- **문서화**: godoc

### 참고 자료
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Project Layout](https://github.com/golang-standards/project-layout)
- [TDD in Go](https://quii.gitbook.io/learn-go-with-tests/)

## 🤝 템플릿 개선 및 기여

### 피드백 환영
- 사용 경험 및 개선 사항 제안
- 누락된 가이드 발견 시 알려주세요
- 실제 프로젝트 적용 사례 공유
- 새로운 Go 패턴 및 베스트 프랙티스 추가

### 지속적인 개선
이 템플릿은 실제 프로젝트 경험을 바탕으로 지속적으로 개선됩니다.

---

**버전**: 1.1  
**최종 업데이트**: 2025년 7월  
**문의**: 개선 사항이나 질문이 있으면 언제든 말씀해주세요!

> 💡 **중요**: 이 README.md는 프로젝트 시작 후 프로젝트별 README로 교체하세요. 이 파일은 `TEMPLATE_README.md`로 백업해두는 것을 권장합니다.

> 🔄 **컨텍스트 관리**: TDD 사이클 완료 후 반드시 `/compact`를 실행하여 최적의 개발 환경을 유지하세요.