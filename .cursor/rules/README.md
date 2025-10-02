# Cursor Rules for Go Development

This directory contains Cursor AI rules for Go microservices development with Clean Architecture principles.

## Rule Files

### Always Applied Rules

These rules are applied to every AI request:

1. **`go-core-practices.mdc`** - Core Go development practices
   - Function design and error handling
   - Dependency management and concurrency
   - Tooling, logging, and performance

2. **`go-architecture.mdc`** - Architecture and project structure
   - Clean Architecture patterns
   - Domain-Driven Design principles
   - Project layout and documentation standards

3. **`go-observability.mdc`** - OpenTelemetry and observability
   - Distributed tracing setup
   - Metrics collection
   - Structured logging and alerting

### Context-Specific Rules

These rules are applied based on file patterns (globs):

4. **`go-http-api.mdc`** - HTTP server and API development
   - Applied to: handlers, controllers, API files
   - Standard library HTTP patterns
   - Middleware, validation, and error handling
   - Security and resilience patterns

5. **`go-testing.mdc`** - Testing practices
   - Applied to: `*_test.go` files and test directories
   - Unit testing with testify
   - Integration testing with TestContainers
   - HTTP handler testing patterns

6. **`go-database.mdc`** - Database operations
   - Applied to: repository, database, migration files
   - sqlx and SQL best practices
   - Migration management with goose
   - Transaction handling and optimization

## How Rules Work

- **Always Applied**: Rules with `alwaysApply: true` are included in every AI interaction
- **Glob-Based**: Rules with `globs` are applied when working with matching files
- **Manual**: Rules can also be manually referenced in conversations

## Usage

These rules are automatically loaded by Cursor AI. No additional setup required.

To reference a file in your rule, use:
```markdown
[filename.ext](mdc:filename.ext)
```

## Philosophy

All rules are based on:
- Idiomatic Go development
- Clean Architecture principles
- Interface-driven development
- Test-driven development
- Production-ready observability
- Security by default

