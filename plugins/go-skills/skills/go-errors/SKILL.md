---
name: go-errors
description: Use when handling errors in Go code. Covers error interface, fmt.Errorf, %w verb, unwrap, error chain, error wrapping, errors.Is, errors.As, sentinel errors, custom error type, panic and recover, error handling strategy, error handling patterns, stack trace, pkg/errors, error group, errgroup, multi-error, validation errors, in-band errors, error type switch, and common error handling mistakes.
---

# Go Error Handling

## Core Principle

Errors are values in Go. Handle them explicitly — don't hide them, don't panic for expected failures.

## Error Interface

```go
type error interface {
    Error() string
}
```

## Creating Errors

```go
// Simple errors
import "errors"
var ErrNotFound = errors.New("not found")

// Formatted errors
import "fmt"
return fmt.Errorf("user %d not found", id)

// Wrapped errors (Go 1.13+) — preserves cause chain
return fmt.Errorf("fetching user: %w", err)

// Custom error type
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}
```

## Error Wrapping & Unwrapping

```go
// Wrap with context
if err := db.Query(sql); err != nil {
    return fmt.Errorf("querying users: %w", err)
}

// errors.Is — check for specific error in chain
if errors.Is(err, sql.ErrNoRows) {
    return nil, ErrNotFound
}

// errors.As — extract specific error type from chain
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println("Failed at path:", pathErr.Path)
}
```

| Verb | Effect |
|------|--------|
| `%w` | Wraps error — `errors.Is` / `errors.As` can unwrap |
| `%v` | Formats error — wrapping chain is lost |

### When to Wrap vs Not

| Scenario | Action | Why |
|----------|--------|-----|
| Adding context | `fmt.Errorf("...: %w", err)` | Preserves chain for callers |
| Hiding implementation detail | `fmt.Errorf("...: %v", err)` or new error | Don't leak internal types |
| Matching sentinel errors | Use `%w` | Callers can `errors.Is` |
| Library boundary | Consider carefully | Wrapping couples caller to your dependencies |

## Sentinel Errors

```go
// Package-level error values
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrConflict     = errors.New("conflict")
)

// Usage
func FindUser(id int) (*User, error) {
    user, err := db.Get(id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrNotFound
        }
        return nil, fmt.Errorf("finding user %d: %w", id, err)
    }
    return user, nil
}
```

| Sentinel rule | Detail |
|---------------|--------|
| Name with `Err` prefix | `ErrNotFound`, `ErrTimeout` |
| Package-level `var` | Not `const` — must be comparable with `errors.Is` |
| Use sparingly | Only for errors callers need to check programmatically |
| Document them | Part of your API contract |

## Custom Error Types

```go
type NotFoundError struct {
    Resource string
    ID       int
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s %d not found", e.Resource, e.ID)
}

// Optional: wrap underlying error
type QueryError struct {
    Query string
    Err   error
}

func (e *QueryError) Error() string {
    return fmt.Sprintf("query %q: %v", e.Query, e.Err)
}

func (e *QueryError) Unwrap() error {
    return e.Err  // enables errors.Is/errors.As
}
```

## Errors as Values

Errors are programmable values — use patterns to reduce repetitive `if err != nil`:

```go
// ErrorWriter: accumulates error, check once at end
type ErrorWriter struct {
    w   io.Writer
    err error
}

func (ew *ErrorWriter) Write(buf []byte) {
    if ew.err != nil { return }
    _, ew.err = ew.w.Write(buf)
}

// Usage — chain writes, check once
ew := &ErrorWriter{w: w}
ew.Write(header)
ew.Write(body)
ew.Write(footer)
if ew.err != nil {
    return ew.err
}
```

## No In-Band Errors

```go
// BAD: -1 or empty string as error signal
func Lookup(key string) string {
    if _, ok := m[key]; !ok { return "" }  // "" might be valid!
    return m[key]
}

// GOOD: return additional value
func Lookup(key string) (string, bool) {
    v, ok := m[key]
    return v, ok
}

// GOOD: return error
func Lookup(key string) (string, error) {
    v, ok := m[key]
    if !ok { return "", ErrNotFound }
    return v, nil
}
```

## Handle Every Error

```go
// BAD: silently ignored
f.Close()

// GOOD: handle or explicitly acknowledge
if err := f.Close(); err != nil {
    log.Printf("closing file: %v", err)
}

// OK: documented why it's safe to ignore
n, _ := buf.Write(data) // bytes.Buffer.Write never returns error
```

---

## Handling Patterns

### The Standard Pattern

```go
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doing something: %w", err)
}
// use result
```

### Multiple Cleanup Actions

```go
func process() (err error) {
    f, err := os.Open("file.txt")
    if err != nil {
        return err
    }
    defer f.Close()

    w, err := os.Create("output.txt")
    if err != nil {
        return err
    }
    defer func() {
        closeErr := w.Close()
        if err == nil {
            err = closeErr  // capture close error if no prior error
        }
    }()

    _, err = io.Copy(w, f)
    return err
}
```

### Error Type Switch

```go
switch err := err.(type) {
case nil:
    // success
case *ValidationError:
    http.Error(w, err.Message, http.StatusBadRequest)
case *NotFoundError:
    http.Error(w, err.Error(), http.StatusNotFound)
default:
    log.Printf("unexpected error: %v", err)
    http.Error(w, "internal error", http.StatusInternalServerError)
}
```

## Panic & Recover

### Panic

```go
// Panic for truly unrecoverable situations
func MustCompile(pattern string) *Regexp {
    re, err := Compile(pattern)
    if err != nil {
        panic("regexp: Compile(" + pattern + "): " + err.Error())
    }
    return re
}
```

| Panic rule | Detail |
|------------|--------|
| Only for programmer errors | Impossible states, violated invariants |
| Never for expected failures | Network errors, file not found → return `error` |
| `Must` prefix convention | `MustCompile`, `MustParse` — panics on error |
| Acceptable in `init()` | If package cannot initialize |

### Recover

```go
func safeHandler(w http.ResponseWriter, r *http.Request) {
    defer func() {
        if err := recover(); err != nil {
            log.Printf("panic recovered: %v\n%s", err, debug.Stack())
            http.Error(w, "internal error", http.StatusInternalServerError)
        }
    }()
    handleRequest(w, r)
}
```

| Recover rule | Detail |
|--------------|--------|
| Only in deferred functions | `recover()` returns nil outside defer |
| Stops unwinding | Returns panic value |
| Use at goroutine/request boundary | Prevent one failure from crashing server |
| Always log the panic | With stack trace (`debug.Stack()`) |
| Don't recover to hide bugs | Fix the root cause |

## Common Mistakes

| Mistake | Why Bad | Fix |
|---------|---------|-----|
| `_ = f.Close()` | Silently loses write errors | `if err := f.Close(); err != nil { ... }` |
| `panic` for expected errors | Crashes program | Return `error` |
| Not wrapping errors | Lost context | `fmt.Errorf("context: %w", err)` |
| Comparing errors with `==` | Doesn't check wrapped chain | `errors.Is(err, target)` |
| Type-asserting errors directly | Doesn't check wrapped chain | `errors.As(err, &target)` |
| Wrapping with `%v` when `%w` intended | Breaks `errors.Is` / `errors.As` | Use `%w` for wrapping |
| Error strings starting with capital | Convention violation | Lowercase: `"opening file: ..."` |
| Error strings ending with punctuation | Doesn't compose well | No period: `"not found"` not `"not found."` |
| `if err != nil { return err }` everywhere | Lost context | Add wrapping message |
| Returning error AND valid value | Confusing API | If `err != nil`, zero-value the result |
