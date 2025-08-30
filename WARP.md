# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Quick Start

### Essential Build Commands

```bash
# Build SQLite extensions first (required for most packages)
cd sqlite && make generate

# Test a specific module with proper timeout
gtimeout 30s go test -v ./jsonschema/...

# Test with coverage
gtimeout 30s go test -v -coverprofile=coverage.out ./sqlite/...
go tool cover -html=coverage.out -o coverage.html

# Build a module with SQLite tags
cd sqlite && CGO_CFLAGS="-DSQLITE_ENABLE_LOAD_EXTENSION=1" go build -tags "sqlite_allow_extension_load" ./...
```

### Running Tests Across All Modules

```bash
# Run tests for all modules (each has its own go.mod)
for dir in gemini jsonschema jsonpatch jsonpointer xmldom sqlite registry muid; do
    echo "Testing $dir..."
    cd $dir && gtimeout 30s go test -v ./... && cd ..
done
```

## Architecture Overview

This repository contains **8 independent Go modules** without a root go.mod file. Each module is self-contained:

### Module Dependencies and Relationships

```
gemini/          - AI client with Google Gemini API integration
├── Uses: Starlark scripting, OpenTelemetry metrics
├── Features: Rate limiting, model management, content generation

sqlite/          - Database layer with custom SQLite extensions  
├── Depends on: jsonschema, jsonpatch, jsonpointer (via local replace directives)
├── Extensions: Graph queries, Vector search (SQLite extensions in C/C++)
├── Build requirement: CGO, C compiler

jsonschema/      - JSON Schema validation and reflection
├── Uses: jsonpatch for schema operations
├── Features: Go struct to JSON schema reflection, validation

jsonpatch/       - RFC 6902 JSON Patch operations
└── Foundation library used by jsonschema

jsonpointer/     - RFC 6901 JSON Pointer implementation
└── Foundation library used by jsonschema

xmldom/          - XML DOM manipulation and XPath queries
├── Features: XML parsing, DOM tree operations, XPath evaluation
├── High test coverage (2000+ lines of tests)

registry/        - Service/component registry pattern
├── Features: Type-safe registration, dependency injection support

muid/           - Unique identifier generation
└── Features: Multiple ID formats, collision-resistant
```

### Key Architectural Patterns

- **No circular dependencies**: Modules have clean dependency boundaries
- **Local module replacement**: sqlite/ uses `replace` directives for local modules
- **Factory patterns**: Consistent `New`/`Make` function naming (New for pointers, Make for values)
- **Interface-driven design**: Heavy use of Go interfaces for modularity

## Build Requirements

### SQLite Module (Critical)

The `sqlite/` module has the most complex build requirements:

```bash
# Required build environment
export CGO_CFLAGS="-DSQLITE_ENABLE_LOAD_EXTENSION=1"
go build -tags "sqlite_allow_extension_load"

# Platform-specific notes:
# - Linux: Requires gcc, sqlite3-dev
# - macOS: May need to remove quarantine attributes from extensions
# - Windows: Build tags specify "!windows && cgo"
```

#### C/C++ Extensions Build Process

```bash
cd sqlite/
make generate  # Builds both graph and vec extensions

# This process:
# 1. Builds SQLite graph extension (C/C++)  
# 2. Builds SQLite vector extension (C/C++)
# 3. Copies platform-specific shared libraries (.so/.dylib)
# 4. Embeds extensions into Go code via go:embed
```

#### Extension Loading Mechanism

The SQLite extensions are embedded as byte arrays and dynamically loaded:

1. Extensions written to temporary files with 0755 permissions
2. macOS quarantine attributes removed via `xattr -d com.apple.quarantine`
3. Extensions loaded via `SELECT load_extension()` calls
4. Custom SQLite driver registered with connection hooks

### Standard Modules

Most other modules have standard Go build requirements:

```bash
# Standard build (works for most modules)
go build ./...

# With specific Go version requirement
go 1.24.5  # All modules standardized on this version
```

## Development Patterns

### Function Naming Conventions

```go
// Factory functions - consistent across codebase
func New(*Config) *Service    // Returns pointer
func Make(Config) Service     // Returns value  

// Example from gemini/model.go:
func NewModel(name ModelName, generateRateLimit *RateLimiter, tokenCountRateLimit *RateLimiter) *Model

// Example from jsonpointer/pointer.go:  
func New(path string) *Pointer
```

### Error Handling & Logging

```go
// Structured logging with slog (standard across modules)
import "log/slog"

slog.Error("gemini.client.generateContent: model not found", "model", model, "models", c.models)

// Error wrapping patterns
return fmt.Errorf("store.NewDB: failed to enable extension loading: %w", err)
```

### Testing Patterns

```go
// Timeout requirement - CRITICAL
func TestSomething(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    // ... test implementation
}

// Coverage target: >90% with 100% pass rate
// Benchmark tests included in most modules (*_bench_test.go)
```

### Concurrency Patterns

- **Prefer channels and atomics over mutexes** (per rules)
- **Rate limiting**: Custom implementation in `gemini/ratelimiter.go`
- **Context usage**: Extensive context.Context propagation for cancellation

## Testing Guidelines

### Test Execution with Timeouts

```bash
# Single test with timeout (REQUIRED)
gtimeout 30s go test -v -run TestSpecificTest ./package

# Coverage testing
gtimeout 30s go test -v -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html

# Benchmark testing
gtimeout 30s go test -v -bench=. ./...
```

### SQLite Testing Specifics

```bash
cd sqlite/

# Build extensions first
make generate

# Run all tests
gtimeout 30s go test -v ./...

# Run specific extension tests
gtimeout 30s go test -v -run TestGraph ./...
gtimeout 30s go test -v -run TestVec ./...

# Test with sanitizers (for extension debugging)
make test_hardened
```

### High Coverage Expectations

- **Target**: >90% test coverage with 100% pass rate
- **xmldom module**: Extensive test suite (comprehensive_test.go, conformance_test.go)
- **Integration tests**: Test cross-module interactions, especially sqlite dependencies

## Module-Specific Notes

### gemini/
- **API Keys**: Never commit keys; expects environment variables or secure injection
- **Rate Limiting**: Built-in per-model rate limiting with token counting
- **Starlark Integration**: Data model includes Starlark scripting capabilities
- **Models supported**: Flash, Pro, Embedding variants

### sqlite/
- **Extension Dependencies**: Must run `make generate` before testing/building
- **Platform Sensitivity**: Extensions may fail to load without proper CGO setup
- **Memory Management**: Extensions include sanitizer support for debugging
- **Graph Queries**: Custom SQLite extension for graph traversal operations
- **Vector Search**: Custom SQLite extension for similarity search

### xmldom/
- **Performance Critical**: Extensive benchmarking tests included
- **XPath Support**: Full XPath 1.0 implementation with 1000+ test cases
- **Memory Intensive**: Large XML documents may require GC tuning
- **Conformance**: Includes W3C conformance test suite

### jsonschema/
- **Go Reflection**: Can generate schemas from Go struct tags
- **Validation**: Full JSON Schema Draft 7 support
- **Performance**: Includes benchmark tests for large schema validation

### Dependencies (jsonpatch/, jsonpointer/)
- **RFC Compliant**: Strict adherence to respective RFC specifications
- **Foundation Libraries**: Changes here affect jsonschema and sqlite modules
- **Minimal Dependencies**: Pure Go implementations

### registry/
- **Type Safety**: Heavy use of Go generics for type-safe registration
- **Dependency Injection**: Supports constructor injection patterns

### muid/
- **Collision Resistance**: Multiple algorithms for different use cases
- **Performance**: Optimized for high-throughput ID generation

## Common Gotchas

### SQLite Extensions
- Always run `make generate` in sqlite/ before testing other modules
- CGO environment must be properly configured
- Extensions are platform-specific (.so on Linux, .dylib on macOS)

### Module Testing
- Use `gtimeout` for all test commands (rule requirement)
- Each module directory has its own go.mod - cd into module directory first
- SQLite module tests fail without extension build

### Build Tags
- SQLite requires `sqlite_allow_extension_load` build tag
- Some code has platform-specific build constraints (`!windows && cgo`)

### Import Paths
- Local modules use replace directives in go.mod
- External imports use full github.com/gogo-agent paths
- Avoid circular imports between modules

## Development Workflow

1. **Start with SQLite**: `cd sqlite && make generate`
2. **Test incrementally**: Use gtimeout for all tests
3. **Check coverage**: Aim for >90% coverage
4. **Build with proper tags**: Use required build tags for CGO modules
5. **Cross-module changes**: Test dependent modules after changes to foundation libraries
6. **Performance**: Run benchmarks for performance-critical changes
