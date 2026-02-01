---
name: go-guidelines
description: Use when writing, reviewing, refactoring, building, or deploying Go code. Covers gofmt, golangci-lint, formatting, naming conventions, control structures, functions, defer, data types, struct, slice, map, pointer, receiver, methods, interfaces, embedding, initialization, functional options pattern, constructor validation, package organization, generics, testing patterns, linting, build/deploy workflow, test file placement, error handling, context, module, go.mod, go.sum, build tags, ldflags, Makefile, and common anti-patterns.
---

# Go Guidelines

## Overview

Idiomatic Go guidelines based on [Effective Go](https://go.dev/doc/effective_go) and [Google Go Style Guide](https://google.github.io/styleguide/go/). Go has its own conventions — writing good Go means embracing them, not porting patterns from other languages.

### Core Proverbs

- **Clear is better than clever** — optimize for readability, not cleverness
- **A little copying is better than a little dependency** — duplicate small functions rather than import heavy packages
- **The zero value should be useful** — design types so `var x T` works without initialization
- **Don't communicate by sharing memory; share memory by communicating** — use channels
- **Errors are values** — program with them, don't just check them
- **Don't just check errors, handle them gracefully** — add context, decide action
- **Reflection is never clear** — avoid `reflect` unless building frameworks

## Formatting

| Rule | Detail |
|------|--------|
| Use `gofmt` | No exceptions — all Go code is gofmt'd |
| Indentation | Tabs, not spaces |
| Line length | No hard limit; break long lines naturally |
| Braces | Opening brace on same line (mandatory — semicolon insertion) |

### Import Grouping

```go
import (
    // 1. Standard library
    "context"
    "fmt"
    "net/http"

    // 2. Third-party packages
    "github.com/gorilla/mux"
    "go.uber.org/zap"

    // 3. Internal packages
    "myproject/internal/auth"
    "myproject/user"
)
```

## Naming Conventions

| What | Convention | Example |
|------|-----------|---------|
| Packages | Lowercase, single-word, no underscores | `bufio`, `strconv` |
| Exported names | `MixedCaps` (upper first letter) | `ReadWriter` |
| Unexported | `mixedCaps` (lower first letter) | `readBuf` |
| Getters | `Owner()` not `GetOwner()` | `func (o *Obj) Name() string` |
| Setters | `SetOwner()` | `func (o *Obj) SetName(n string)` |
| One-method interfaces | Method name + `-er` suffix | `Reader`, `Writer`, `Stringer` |
| Acronyms | All caps throughout | `URL`, `HTTP`, `ID` (not `Url`, `Http`, `Id`) |
| Constants | `MixedCaps` (not `ALL_CAPS`) | `MaxRetries`, not `MAX_RETRIES` |
| Receivers | 1-2 letters, consistent | `func (s *Server)`, not `func (server *Server)` |
| Local variables | Length proportional to scope size | 1-7 lines: `i`, `buf`; larger scopes: descriptive |

### Doc Comments

```go
// Package sort provides primitives for sorting slices.
package sort

// Fprint formats using the default formats and writes to w.
// It returns the number of bytes written and any write error.
func Fprint(w io.Writer, a ...any) (n int, err error) {
```

| Doc comment rule | Detail |
|------------------|--------|
| Start with the name of the thing | `// Fprint formats...` not `// This function formats...` |
| Full sentences | Capitalized, ending with period |
| Package comment in any file | One `// Package name ...` per package |

### Naming Pitfalls

```go
// BAD: Java/C# naming
func GetUserName() string { ... }
type IReader interface { ... }

// GOOD: Go naming
func UserName() string { ... }
type Reader interface { ... }

// BAD: stuttering in constructors
widget.NewWidget()
user.NewUser()

// GOOD: package name provides context
widget.New()
user.New()

// BAD: ALL_CAPS constants
const MAX_RETRIES = 3

// GOOD: MixedCaps — name by role, not value
const MaxRetries = 3
```

## Control Structures

| Rule | Detail |
|------|--------|
| Use `if` init statements | `if err := f(); err != nil { }` — keeps scope tight |
| Early return | Avoid `else` after `return` |
| `for` has three forms | C-style, while, infinite |
| `range` is idiomatic | Prefer `for k, v := range` over index-based loops |
| Range on string | Iterates runes (UTF-8), not bytes |
| Switch has no fall-through | Unlike C; use comma-separated cases for multiple matches |
| Switch on true | Replaces if-else chains cleanly |
| Type switch | `switch v := value.(type)` for interface type dispatch |

> Full code examples: [resources/types-functions.md](resources/types-functions.md#control-structures)

## Functions

| Rule | Detail |
|------|--------|
| Return `(value, error)` | The Go pattern for fallible operations |
| Named return values | Document meaning in godoc; naked returns OK in short functions only |
| `defer` runs on any return path | Including panics; arguments evaluated immediately |
| `defer` is LIFO | Last deferred runs first |
| Use `defer` for cleanup | Files, locks, connections |

> Full code examples: [resources/types-functions.md](resources/types-functions.md#functions)

## Data Types

### new vs make

| Function | Returns | Use for | Initializes |
|----------|---------|---------|-------------|
| `new(T)` | `*T` | Any type | Zeroed memory |
| `make(T, ...)` | `T` | Slices, maps, channels only | Internal data structures |

### Slices

| Slice rule | Detail |
|------------|--------|
| Always use return value of `append` | Underlying array may move |
| Pre-allocate with `make([]T, 0, cap)` | When size is known or estimable |
| `len(s)` vs `cap(s)` | Length is current size, capacity is maximum before reallocation |
| Nil slice is valid | `len(nil) == 0`, `append(nil, x)` works |
| Prefer nil for empty slices | `var s []T` not `s := []T{}` — nil encodes to `null` in JSON, empty literal to `[]` |

### Maps

| Map rule | Detail |
|----------|--------|
| Zero value for missing key | `m["absent"]` returns zero, not error |
| Always use comma-ok for presence check | `v, ok := m[key]` |
| Not safe for concurrent access | Use `sync.Map` or mutex |
| Iteration order is random | Don't rely on order |

> Full code examples: [resources/types-functions.md](resources/types-functions.md#data-types)

## Design Principles

### Zero Values Should Be Useful

Design types so `var x T` works without initialization. Example: `var buf bytes.Buffer` is ready to use with no `New()` call. Nil fields should fall back to sensible defaults.

### Don't Copy Types with Locks

Copying a `sync.Mutex` copies its state — undefined behavior. Always use pointer receivers on types containing locks.

### A Little Copying > A Little Dependency

Duplicate small utility functions rather than importing a heavy package for one helper.

### Security

| Rule | Detail |
|------|--------|
| `crypto/rand` not `math/rand` | For keys, tokens, secrets — `math/rand` is predictable |
| Build tags for `syscall` | `//go:build linux` when using platform-specific syscalls |
| Build tags for `cgo` | `//go:build cgo` — cgo sacrifices memory safety and cross-compilation |
| Avoid `unsafe` package | Voids all type safety and compatibility guarantees |
| Avoid `reflect` | Obscures intent, removes compile-time safety |

## Methods

### Pointer vs Value Receivers

| Receiver | Can modify? | Called on | Use when |
|----------|-------------|-----------|----------|
| `func (t T) Method()` | No (copy) | Values and pointers | Read-only, small types |
| `func (t *T) Method()` | Yes | Pointers only* | Mutation, large structs |

\* Go auto-takes address for addressable values.

- Don't pass pointers to small basic types — pass values
- **Consistency:** if any method needs pointer receiver, all should use pointer

> Full code examples: [resources/patterns.md](resources/patterns.md#methods)

## Interfaces

### Design Principles

| Interface rule | Detail |
|----------------|--------|
| Keep interfaces small | 1-2 methods ideal |
| Accept interfaces, return concrete types | Maximum flexibility for callers |
| Don't export implementation types when interface suffices | Return interface from constructors |
| Interfaces are satisfied implicitly | No `implements` keyword |
| Name one-method interfaces with `-er` | `Reader`, `Writer`, `Closer`, `Stringer` |
| Define in consuming package | Not in implementing package |
| Don't define before use | Wait until you have realistic usage |
| Don't export unused interfaces | If only used internally, keep unexported |
| Prefer synchronous functions | Let callers add concurrency — easier to test and reason about |

### Type Assertions

```go
// Safe form — comma-ok
str, ok := value.(string)
if !ok {
    // value is not a string
}

// Type switch — multiple type checks
switch v := value.(type) {
case string:
    return v
case fmt.Stringer:
    return v.String()
default:
    return fmt.Sprintf("%v", value)
}
```

### Embedding

```go
// Interface embedding — compose interfaces
type ReadWriter interface {
    Reader
    Writer
}

// Struct embedding — promote methods
type Job struct {
    Command string
    *log.Logger  // promoted: job.Println() works
}
```

## Initialization

| Init rule | Detail |
|-----------|--------|
| `init()` called after all variable declarations | Automatic |
| All imports initialized first | Dependency order |
| Multiple `init()` per file allowed | Run in order |
| Use for verification | Not complex logic |

> Full code examples: [resources/patterns.md](resources/patterns.md#initialization)

## Functional Options Pattern

Use for 3+ optional configuration fields, library APIs, or when defaults should work out of the box. Define `type Option func(*T)`, create `WithX` constructors, and apply in `NewT(opts ...Option)`.

| When to use | When NOT to use |
|-------------|-----------------|
| 3+ optional configuration fields | 1-2 required parameters — just use arguments |
| Library APIs (callers shouldn't know internals) | Internal structs with few fields — use literal |
| Defaults should work out of the box | Config loaded from file — use a config struct |

> Full code examples: [resources/patterns.md](resources/patterns.md#functional-options-pattern)

## Constructor & Validation

| Pattern | When |
|---------|------|
| `NewX() *X` | Simple construction, always succeeds |
| `NewX() (*X, error)` | Validation needed, may fail |
| `MustX() *X` | Panics on error — only for compile-time constants or tests |

> Full code examples: [resources/patterns.md](resources/patterns.md#constructor--validation)

## Generics (Go 1.18+)

| Use generics for | Don't use generics for |
|------------------|------------------------|
| Type-safe collections/containers | Simple functions that work with `interface{}` |
| Algorithm reuse across types | When only one type is ever used |
| Reducing code duplication | When it makes code harder to read |
| Options/builder patterns | Premature abstraction |

> Full code examples: [resources/patterns.md](resources/patterns.md#generics-go-118)

## Package Organization

```
// Domain-based (preferred for applications)
myapp/
├── user/           # User domain
│   ├── user.go
│   ├── store.go
│   └── handler.go
├── order/          # Order domain
│   ├── order.go
│   └── service.go
├── internal/       # Private packages
│   └── db/
└── cmd/
    └── server/main.go

// Flat (for libraries and small services)
mylib/
├── mylib.go        # Primary types and functions
├── option.go       # Options pattern
├── mylib_test.go
└── internal/       # Implementation details
```

| Rule | Detail |
|------|--------|
| `cmd/` | Entry points — `main` packages |
| `internal/` | Private — compiler-enforced, cannot be imported outside module |
| `pkg/` | Optional — public library code (some projects skip this) |
| Avoid `models/`, `utils/`, `helpers/` | Too generic — organize by domain |
| One package = one purpose | If you can't name it in one word, split it |

## Testing

| Testing rule | Detail |
|--------------|--------|
| `_test.go` suffix | Test files, excluded from production builds |
| `Test` prefix | Functions must start with `TestXxx(t *testing.T)` |
| `t.Run` for subtests | Enables selective running: `go test -run TestAdd/positive` |
| `t.Helper()` | Marks helper functions for better error locations |
| `t.Cleanup()` | Deferred cleanup that runs after test completes |
| `t.Parallel()` | Opt-in parallel execution within a test |
| `testdata/` directory | Test fixtures, ignored by Go tooling |
| Got before want | `t.Errorf("Func(%v) = %v, want %v", input, got, want)` |
| `t.Error` over `t.Fatal` | Keep going — report all failures in one run |
| `t.Fatal` only for setup | When test literally cannot proceed |
| No `t.Fatal` from goroutines | Only call from test function's goroutine |

> Full code examples: [resources/testing-verification.md](resources/testing-verification.md#testing)

## Static Verification

| Tool | Purpose |
|------|---------|
| `go vet ./...` | Built-in static analysis |
| `golangci-lint run ./...` | Comprehensive linter aggregator |
| `go test -race ./...` | Detect data races at runtime |
| `go test -count=1 ./...` | Disable test caching |
| `go test -cover ./...` | Coverage report |

> Full config and examples: [resources/testing-verification.md](resources/testing-verification.md#static-verification)

---

## Anti-Pattern Quick Reference

| Anti-Pattern | Better Alternative |
|--------------|--------------------|
| `GetName()` getter | `Name()` |
| Stuttering: `user.UserName` | `user.Name` |
| `interface{}` everywhere | Specific types or generics |
| Large interfaces (10+ methods) | Small, composed interfaces |
| Returning `error` and ignoring it | Always handle or explicitly `_ =` |
| `else` after `return` | Early return pattern |
| Naked goroutines (fire-and-forget) | Track with `sync.WaitGroup` or `errgroup` |
| `init()` with complex logic | Explicit initialization in `main()` |
| Mutable package-level vars | Pass config explicitly |
| Value receiver on large struct | Pointer receiver |
| Checking `== nil` on interface | Check concrete value (interfaces have type+value) |
| `panic` for expected errors | Return `error` |
| Underscore imports without comment | Document why: `import _ "pkg" // register driver` |
| `new(T)` when make is needed | `make` for slices/maps/channels |
| Ignoring `append` return value | Always `s = append(s, ...)` |
| Config struct with 10+ fields | Functional options pattern |
| No validation in constructor | `NewX() (*X, error)` with `validate()` |
| `utils/` or `helpers/` package | Organize by domain, not by kind |
| `models/` package for all types | Put types where they're used |
| Not using `t.Helper()` | Test helpers report wrong line numbers |
| Not using `t.Cleanup()` | Cleanup may not run on `t.Fatal()` |
| `go test` without `-race` | Data races go undetected |
| Copying struct with `sync.Mutex` | Use pointer receiver, never copy |
| `math/rand` for secrets | `crypto/rand` for tokens, keys, secrets |
| In-band errors (`-1`, empty string) | Return `(value, error)` or `(value, bool)` |
| Defining interface in implementing package | Define where consumed |
| `ALL_CAPS` constants | `MixedCaps` — Go convention |
| `import . "pkg"` (dot import) | Makes code origin unclear |
| Context in struct field | Always pass `context.Context` as first parameter |
| Clever one-liners | Clear, readable multi-line code |
| `reflect` for simple tasks | Generics or type assertions |

---

## Build & Deploy

### Always Use Makefile

Before running `go build` or any build command, check if a `Makefile` exists. If it does, **use it** — the Makefile encodes project-specific build steps (ldflags, embedding, code generation) that raw commands miss.

| Situation | Action |
|-----------|--------|
| Makefile exists with relevant target | `make deploy`, `make build`, `make test`, etc. |
| Makefile exists, no matching target | List targets, pick closest match |
| No Makefile | Fall back to `go build`, `go test`, etc. |

### Permissions & Ownership

Before building or deploying, check file permissions and ownership. Fix to match the project majority:

```bash
# Identify majority owner:group
stat -c '%U:%G' * | sort | uniq -c | sort -rn | head -5
# Fix if needed
chown -R <user>:<group> .
```

### Temporary Files

| File type | Location |
|-----------|----------|
| Build intermediates, scratch files | `/tmp` — never the project directory |
| Final build artifacts | Project output dir or `$GOBIN` |
| Downloaded dependencies | Standard location (`~/go/pkg/`) |

**Exception:** If project instructions (CLAUDE.md, Makefile) specify a different temp location, follow those.

### Test File Placement

| Test type | Location |
|-----------|----------|
| Temporary (debugging, one-off) | `/tmp` |
| Permanent, Go convention | `*_test.go` beside the source file |
| Permanent, integration/e2e | `tests/` directory (create if it doesn't exist) |
| Specific instructions exist | Follow those |

---

## Resources

Detailed code examples are organized into resource files for progressive disclosure:

- **[Types, Functions & Control Flow](resources/types-functions.md)** — Control structure patterns (`if`/`for`/`switch`), function signatures, defer, `new` vs `make`, composite literals, slices, maps, and printing
- **[Patterns](resources/patterns.md)** — Methods (pointer vs value receivers), initialization (`iota`, `init`), functional options, constructor validation, and generics
- **[Testing & Verification](resources/testing-verification.md)** — Table-driven tests, parallel tests, test helpers, testify, golangci-lint config, `go vet`, race detector, and coverage
