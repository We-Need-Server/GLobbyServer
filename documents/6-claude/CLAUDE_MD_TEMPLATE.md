# CLAUDE.md Template

This template provides a standardized structure for CLAUDE.md files in software development projects, incorporating best practices and essential workflow rules derived from real project experiences.

## Project Overview

[PROJECT_NAME] - [Brief project description]

### Key Project Details
- **Language**: [Go version - e.g., Go 1.21+]
- **Build Tool**: [Build system - e.g., go build, make, goreleaser]
- **Target**: [Target environment - e.g., Linux, macOS, Windows, Docker]
- **Package Manager**: [Go modules - go.mod]
- **Architecture**: [Architecture approach - e.g., layered, microservices, hexagonal]

## Development Commands

```bash
# Essential commands for development
go build               # Build the project
go run main.go         # Run the application
go test ./...          # Run tests
go fmt ./...           # Format code
go vet ./...           # Vet code for issues
golangci-lint run      # Advanced linting
```

## Project Architecture

### Core Components Structure
```
cmd/
├── [app-name]/        # Application entry point
├── [cli-tool]/        # CLI tool entry point
internal/
├── [domain]/          # Domain logic
├── [service]/         # Business services
├── [repository]/      # Data access layer
└── [config]/          # Configuration
pkg/
├── [library]/         # Public packages
└── [utils]/           # Utility functions
```

### Key Architecture Decisions
- **[Decision 1]**: [Rationale]
- **[Decision 2]**: [Rationale]
- **[Decision 3]**: [Rationale]

## Core Dependencies

### Required Dependencies
- `[dependency1]` - [Purpose]
- `[dependency2]` - [Purpose]
- `[dependency3]` - [Purpose]

### Build Dependencies
- `[dev-dependency1]` - [Purpose]
- `[dev-dependency2]` - [Purpose]

## Key Features & Requirements

### API Design
```go
// Example API struct
type [StructName] struct {
    [Property1] [Type] `json:"property1"` // [Description]
    [Property2] [Type] `json:"property2"` // [Description]
}

// Usage example
config := [StructName]{
    [Property1]: "[value]",
    [Property2]: "[value]",
}
```

## Testing Strategy

The project uses comprehensive test coverage including:
- **Unit Tests**: Individual component testing
- **Integration Tests**: End-to-end functionality
- **Performance Tests**: Performance benchmarks
- **Compatibility Tests**: Cross-platform functionality

## Code Structure Guidelines

### Best Practices
- Use proper Go module structure and naming
- Implement idiomatic error handling
- Follow interface-based design patterns
- Maintain clear separation of concerns
- Use Go's built-in formatting and linting tools

### Module Organization
- **[module1]**: [Purpose and responsibilities]
- **[module2]**: [Purpose and responsibilities]
- **[module3]**: [Purpose and responsibilities]

## Documentation Workflow Requirements

### MANDATORY Documentation Process
**All implementation work must follow this documentation workflow:**

1. **Pre-implementation Planning**
   - Document goals and scope in DEVELOPMENT_LOG.md
   - Identify estimated time and potential blockers

2. **Learning During Implementation**
   - Document new technologies, APIs, patterns immediately using LEARNING_NOTES_TEMPLATE
   - File location: `documents/5-learning/LEARNING_NOTES_[date].md`

3. **Post-implementation Documentation**
   - Update DEVELOPMENT_LOG.md daily logs immediately after completion
   - Document blocker solutions, performance metrics, and next plans

### Documentation Rules
- **LEARNING_NOTES**: Mandatory when learning new technologies/concepts
- **DEVELOPMENT_LOG**: Mandatory update after all implementation completion
- **File Locations**: 
  - Development log: `documents/4-progress/DEVELOPMENT_LOG.md`
  - Learning notes: `documents/5-learning/LEARNING_NOTES_[YYYY-MM-DD].md`

### Non-compliance Consequences
Implementation without following this documentation process is considered **incomplete work**.
Claude must comply with these rules and generate appropriate documentation at all implementation stages.

## Git and Package Management Rules

### Package Manager Rules
**This project uses Go modules as the standard package manager.**

- **Dependency installation**: `go mod tidy`
- **Add packages**: `go get [package-name]`
- **Add dev dependencies**: `go get -t [package-name]`
- **Update dependencies**: `go get -u [package-name]`
- **Remove unused dependencies**: `go mod tidy`

### Git Commit Management Rules
**All meaningful changes must be managed with appropriate commits.**

#### Automatic Commit Targets
1. **Dependency changes**: When go.mod, go.sum change
2. **Implementation completion**: When new module implementation is complete
3. **Test passes**: When TDD Green Phase is complete
4. **Documentation updates**: When important documentation changes occur

#### Commit Message Rules
- **feat**: Add new feature
- **fix**: Bug fix
- **docs**: Documentation changes
- **test**: Add/modify test code
- **refactor**: Code refactoring
- **chore**: Build process or auxiliary tool changes

#### Branch Strategy
- **main**: Stable release branch
- **feature/[feature-name]**: New feature development
- **fix/[bug-name]**: Bug fixes
- **docs/[doc-name]**: Documentation updates

### Automation Process
Claude should automatically perform the following processes:

1. **Commit after dependency installation**
   ```bash
   go mod tidy
   git add go.mod go.sum
   git commit -m "chore: update dependencies"
   ```

2. **Commit after implementation completion**
   ```bash
   git add [modified-files]
   git commit -m "feat: implement [feature-name]"
   ```

3. **Commit after test passes**
   ```bash
   git add [test-files] [implementation-files]
   git commit -m "test: add tests for [feature-name] and implement solution"
   ```

## Context Management and /compact Rules

### Context Cleanup on TDD Cycle Completion
**Execute /compact mandatory when major work units (TDD Red-Green-Refactor cycle) are complete**

#### /compact Execution Timing
1. **After TDD cycle completion**
   - Red Phase (failing test creation) → Green Phase (implementation) → Refactor Phase (refactoring) complete
   - After one module (e.g., logger.go, trace.go) implementation complete
   - After related documentation updates and commits are complete

2. **After major milestone completion**
   - Phase unit work completion (e.g., Phase 1, Phase 2)
   - After integration testing completion
   - After build and deployment preparation completion

#### /compact Execution Process
```
1. Check current work status
   - Verify all tests pass
   - Verify commits are complete
   - Verify documentation updates are complete

2. Execute /compact
   - Compress conversation context
   - Optimize memory
   - Clean up token usage

3. Prepare for next work
   - Start new TDD cycle
   - Plan next module implementation
```

#### /compact Execution Criteria
- **Mandatory execution**: After TDD cycle completion (Red → Green → Refactor)
- **Recommended execution**: After 500+ lines of code implementation completion
- **Optional execution**: After complex debugging session completion

### Context Management Benefits
1. **Performance optimization**: Reduced token usage and improved response speed
2. **Improved focus**: Better focus on current work by removing unnecessary context
3. **Memory efficiency**: Optimized memory usage through conversation history cleanup
4. **Quality improvement**: Better code quality with clean context

## Error Handling

### Graceful Degradation
- Proper error logging without interfering with application
- Fallback mechanisms for network failures
- Handle missing dependencies gracefully

### Custom Error Types
```go
// Custom error type
type [ProjectName]Error struct {
    Code    string
    Message string
    Err     error
}

func (e *[ProjectName]Error) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("[%s] %s: %v", e.Code, e.Message, e.Err)
    }
    return fmt.Sprintf("[%s] %s", e.Code, e.Message)
}

func (e *[ProjectName]Error) Unwrap() error {
    return e.Err
}
```

## Performance Requirements

### Target Metrics
- **Binary size**: < [size]MB
- **Startup time**: < [time]ms
- **Memory usage**: < [size]MB
- **CPU usage**: < [percentage]%
- **Network latency**: Minimized through [optimization strategy]

## Security Considerations

- **No sensitive data exposure**: Never log secrets or keys
- **Input validation**: Validate all user inputs
- **Error handling**: Don't expose internal details in error messages
- **Dependencies**: Regular security audits of dependencies

## Development Notes

### Current Status
[Current project status - e.g., implementation phase, documentation phase, etc.]

### Next Steps
1. [Step 1]
2. [Step 2]
3. [Step 3]

### Important Considerations
- **[Consideration 1]**: [Details]
- **[Consideration 2]**: [Details]
- **[Consideration 3]**: [Details]

## Customization Guidelines

### How to Use This Template

1. **Copy and Customize**
   - Copy this template to your project root as `CLAUDE.md`
   - Replace all `[PLACEHOLDER]` values with project-specific information
   - Remove sections not applicable to your project

2. **Essential Sections**
   - Always keep: Project Overview, Development Commands, Documentation Workflow
   - Always keep: Git and Package Management Rules, Context Management Rules
   - Customize: Architecture, Dependencies, API Design based on project needs

3. **Project-Specific Adaptations**
   - Add technology-specific sections (e.g., web servers, CLI tools, APIs)
   - Include Go-specific best practices and idioms
   - Add domain-specific requirements (e.g., microservices, CLI, web APIs)

4. **Maintenance**
   - Update commands and dependencies as project evolves
   - Add new learnings and patterns discovered during development
   - Keep documentation workflow rules current

### Template Validation Checklist

- [ ] All placeholders replaced with actual values
- [ ] Development commands tested and working
- [ ] Documentation workflow rules clearly understood
- [ ] Git and package management rules configured
- [ ] Context management rules established
- [ ] Project-specific sections added
- [ ] Security considerations addressed
- [ ] Performance targets defined

---

**Template Version**: 1.1  
**Last Updated**: 2025-07-11  
**Created by**: Claude AI Development Team