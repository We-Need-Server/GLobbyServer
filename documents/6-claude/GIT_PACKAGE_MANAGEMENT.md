# Git and Package Management Guide

This guide establishes standardized practices for Git workflow and package management to ensure consistent, reliable, and efficient development processes.

## Overview

Proper version control and package management are fundamental to successful software development. This guide provides clear rules and automation strategies for managing code changes and dependencies.

## Package Management Rules

### Standard Package Manager Selection

**Rule: Use Go modules as the standard package manager for all projects.**

#### Why Go Modules?
- **Deterministic Builds**: go.sum ensures consistent builds across environments
- **Performance**: Built-in dependency resolution and caching
- **Security**: Built-in security auditing with govulncheck
- **Workspace Support**: Native workspace support for multi-module projects
- **Versioning**: Semantic versioning with minimum version selection

#### Required Commands
```bash
# Dependency installation
go mod tidy             # Install all dependencies and remove unused
go mod download         # Download dependencies to local cache

# Package management
go get [package]        # Add production dependency
go get -u [package]     # Update dependency
go get [package]@latest # Get latest version
go mod edit -dropre [package] # Remove dependency

# Information and maintenance
go list -m all          # List all dependencies
go list -m -versions [package] # Show package versions
govulncheck             # Security audit
go clean -modcache      # Clean module cache
```

#### Prohibited Commands
```bash
# DO NOT USE other package managers in Go projects
npm install            # ❌ Use go mod tidy
pip install           # ❌ Use go get
dep ensure            # ❌ Use go mod (dep is deprecated)
glide install         # ❌ Use go mod (glide is deprecated)
```

### Go.mod Standards

#### Required Fields
```go
module github.com/username/project-name

go 1.21

require (
    github.com/dependency/package v1.2.3
    github.com/another/package v0.4.5
)

require (
    // Indirect dependencies
    github.com/indirect/dependency v1.0.0 // indirect
)

replace github.com/old/package => github.com/new/package v1.0.0
```

#### Makefile Conventions
```makefile
# Common Go project commands
.PHONY: build test lint clean run

build:          # Production build
	go build -o bin/app ./cmd/app

test:           # Run tests
	go test ./...

test-watch:     # Watch mode testing
	go test -watch ./...

lint:           # Code linting
	golangci-lint run

lint-fix:       # Auto-fix linting issues
	golangci-lint run --fix

vet:            # Go vet
	go vet ./...

clean:          # Clean build artifacts
	rm -rf bin/ dist/

run:            # Run application
	go run ./cmd/app
```

## Git Workflow Rules

### Commit Management Strategy

**Rule: All meaningful changes must be managed with appropriate commits.**

#### Automatic Commit Triggers

1. **Dependency Changes**
   - When go.mod changes
   - When go.sum changes
   - When adding/removing dependencies

2. **Implementation Completion**
   - When new module implementation is complete
   - When feature implementation is finished
   - When bug fixes are complete

3. **Test Phase Completion**
   - When TDD Green Phase is complete
   - When test suite passes
   - When integration tests complete

4. **Documentation Updates**
   - When significant documentation changes occur
   - When API documentation is updated
   - When README or guides are modified

#### Commit Message Standards

Follow [Conventional Commits](https://www.conventionalcommits.org/) specification:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

##### Commit Types
- **feat**: New feature addition
- **fix**: Bug fix
- **docs**: Documentation changes
- **style**: Code style changes (formatting, semicolons, etc.)
- **refactor**: Code refactoring without functionality changes
- **test**: Add or modify tests
- **chore**: Build process, dependency updates, or auxiliary tool changes
- **perf**: Performance improvements
- **ci**: Continuous integration changes
- **build**: Build system changes

##### Examples
```bash
# Feature implementation
git commit -m "feat: implement logger module with CustomLogExporter"

# Bug fix
git commit -m "fix: resolve compilation errors in trace.go"

# Documentation
git commit -m "docs: update API documentation with new examples"

# Dependency updates
git commit -m "chore: update OpenTelemetry dependencies"

# Test additions
git commit -m "test: add comprehensive tests for logger module"
```

### Branch Strategy

#### Branch Naming Convention
- **main**: Stable release branch
- **develop**: Integration branch for features
- **feature/[feature-name]**: New feature development
- **fix/[issue-description]**: Bug fixes
- **docs/[doc-type]**: Documentation updates
- **refactor/[component-name]**: Code refactoring
- **test/[test-scope]**: Test-related changes

#### Branch Examples
```bash
# Feature branches
feature/console-patching
feature/otlp-exporter
feature/batch-processor

# Bug fix branches
fix/memory-leak-logger
fix/timestamp-precision
fix/type-errors

# Documentation branches
docs/api-reference
docs/getting-started
docs/troubleshooting
```

### Automated Git Processes

#### Process 1: Dependency Management
```bash
# After package installation
go mod tidy
git add go.mod go.sum
git commit -m "chore: update dependencies"
```

#### Process 2: Feature Implementation
```bash
# After feature completion
git add [modified-files]
git commit -m "feat: implement [feature-name]

- Add core functionality
- Include comprehensive tests
- Update documentation"
```

#### Process 3: Test Completion
```bash
# After TDD Green Phase
git add [test-files] [implementation-files]
git commit -m "test: add tests for [feature-name] and implement solution

- Red phase: Add failing tests
- Green phase: Implement minimal solution
- All tests passing"
```

### Git Hooks and Automation

#### Pre-commit Hook
```bash
#!/bin/sh
# .git/hooks/pre-commit

# Run Go formatting
go fmt ./...

# Run Go vetting
go vet ./...

# Run linting
golangci-lint run

# Run tests
go test ./...

# Check for secrets
git secrets --scan
```

#### Commit Message Hook
```bash
#!/bin/sh
# .git/hooks/commit-msg

# Validate commit message format
commitizen check --message-file $1
```

## Best Practices

### Repository Structure
```
project-root/
├── .git/
├── .gitignore
├── .gitattributes
├── go.mod
├── go.sum
├── README.md
├── CHANGELOG.md
├── cmd/
├── internal/
├── pkg/
├── api/
├── web/
├── docs/
├── scripts/
└── Makefile
```

### .gitignore Standards
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

# Dependency directories (remove the comment below to include it)
# vendor/

# Go workspace file
go.work

# Environment files
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# IDE files
.vscode/
.idea/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Logs
*.log

# Runtime data
pids
*.pid
*.seed
*.pid.lock

# Coverage directory
coverage/
*.lcov

# Temporary folders
tmp/
temp/

# Build artifacts
/bin/
/dist/
```

### Security Considerations

#### Dependency Security
```bash
# Regular security audits
govulncheck ./...
go list -json -m all | nancy sleuth

# Automatic security updates
go get -u all
go mod tidy
```

#### Secret Management
```bash
# Never commit secrets
git secrets --install
git secrets --register-aws
```

## Workflow Automation

### Claude AI Automation Rules

#### Automatic Actions
1. **After go mod tidy**: Automatically commit go.mod and go.sum changes
2. **After implementation**: Automatically commit with appropriate message
3. **After tests pass**: Automatically commit test and implementation files
4. **Before major tasks**: Check git status and commit pending changes

#### Automation Scripts
```bash
# scripts/auto-commit-deps.sh
#!/bin/bash
if [[ -n $(git status --porcelain go.mod go.sum) ]]; then
    git add go.mod go.sum
    git commit -m "chore: update dependencies"
fi

# scripts/auto-commit-implementation.sh
#!/bin/bash
git add .
git commit -m "feat: implement $1

🤖 Generated with [Claude Code](https://claude.ai/code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

### Team Workflow

#### Daily Workflow
1. **Morning**: Pull latest changes, install dependencies
2. **During Work**: Commit frequently with descriptive messages
3. **End of Day**: Push changes, update documentation

#### Weekly Workflow
1. **Monday**: Review dependency updates and security audits
2. **Friday**: Clean up branches, update documentation
3. **As Needed**: Security audits, dependency updates

## Troubleshooting

### Common Issues and Solutions

#### Issue: Dependency Conflicts
```bash
# Solution: Clean installation
go clean -modcache
go mod tidy
```

#### Issue: Git Conflicts
```bash
# Solution: Interactive rebase
git rebase -i HEAD~3
# Edit conflicts, then continue
git rebase --continue
```

#### Issue: Large Commit History
```bash
# Solution: Squash commits
git rebase -i HEAD~5
# Change 'pick' to 'squash' for commits to combine
```

### Performance Optimization

#### Go Performance
```bash
# Enable module proxy
go env -w GOPROXY=https://proxy.golang.org,direct

# Use local cache
go env -w GOMODCACHE=/path/to/cache

# Enable build cache
go env -w GOCACHE=/path/to/build/cache
```

#### Git Performance
```bash
# Enable file system monitor
git config core.fsmonitor true

# Enable commit graph
git config core.commitGraph true
git config gc.writeCommitGraph true
```

## Quality Assurance

### Pre-commit Checklist
- [ ] All tests passing
- [ ] Code linted and formatted
- [ ] Go compilation successful
- [ ] Dependencies security audited
- [ ] Commit message follows convention
- [ ] Documentation updated

### Release Preparation
- [ ] Version bumped appropriately
- [ ] CHANGELOG.md updated
- [ ] Dependencies updated and audited
- [ ] All tests passing
- [ ] Documentation complete
- [ ] Build successful

---

**Version**: 1.1  
**Last Updated**: 2025-07-11  
**Maintained by**: Claude AI Development Team