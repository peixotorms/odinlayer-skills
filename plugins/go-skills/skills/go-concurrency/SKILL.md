---
name: go-concurrency
description: Use when writing concurrent Go code with goroutines, channels, select, sync primitives, or WaitGroup. Covers context.Context, cancellation, deadline, timeout, errgroup, semaphore, atomic, sync.Map, sync.Once, sync.Mutex, sync.RWMutex, sync.Pool, race condition, data race, -race flag, fan-out, fan-in, pipeline, share-by-communicating, channel patterns, buffered vs unbuffered channels, select with default, parallelization, worker pools, resource pools, done channel, and common concurrency mistakes.
---

# Go Concurrency

## Core Principle

> "Do not communicate by sharing memory; instead, share memory by communicating."

Pass data ownership through channels. Only one goroutine accesses a value at a time — data races cannot occur by design.

## Goroutines

```go
// Launch with go keyword — lightweight, cheap to create
go list.Sort()

// Anonymous goroutine with closure
go func() {
    time.Sleep(delay)
    fmt.Println(message)
}()
```

| Goroutine fact | Detail |
|----------------|--------|
| Cost | ~2KB initial stack, grows as needed |
| Scheduling | Multiplexed onto OS threads by Go runtime |
| No return value | Use channels or shared state for results |
| No handle/ID | Cannot kill externally — use context for cancellation |

## Channels

### Unbuffered vs Buffered

| Type | Creation | Sender blocks | Receiver blocks |
|------|----------|---------------|-----------------|
| Unbuffered | `make(chan T)` | Until receiver reads | Until sender sends |
| Buffered | `make(chan T, n)` | When buffer full | When buffer empty |

```go
// Unbuffered — synchronization point
done := make(chan struct{})
go func() {
    doWork()
    done <- struct{}{}  // signal completion
}()
<-done  // wait

// Buffered — decouple producer/consumer
jobs := make(chan Job, 100)
```

### Channel Direction

```go
// Send-only
func producer(out chan<- int) { out <- 42 }

// Receive-only
func consumer(in <-chan int) { v := <-in }
```

### Range Over Channel

```go
// Iterate until channel closed
for msg := range ch {
    process(msg)
}
// Loop exits when ch is closed and drained
```

### Closing Channels

| Rule | Detail |
|------|--------|
| Only sender should close | Sending to closed channel panics |
| Close is a broadcast | All receivers unblock |
| Receiving from closed returns zero value | Use comma-ok: `v, ok := <-ch` |
| Don't close to signal "no more data for now" | Close means "permanently done" |

## Select

```go
select {
case v := <-ch1:
    process(v)
case ch2 <- result:
    // sent
case <-time.After(5 * time.Second):
    // timeout
default:
    // non-blocking — runs if no channel ready
}
```

| Select rule | Detail |
|-------------|--------|
| Random choice when multiple ready | Not first-match |
| `default` makes it non-blocking | Use for try-send/try-receive |
| `time.After` for timeouts | Or `context.WithTimeout` |
| Nil channel blocks forever | Useful to disable a case dynamically |

## Common Patterns

### Semaphore (limit concurrency)

```go
var sem = make(chan struct{}, MaxConcurrent)

func handle(r *Request) {
    sem <- struct{}{}   // acquire
    defer func() { <-sem }()  // release
    process(r)
}
```

### Worker Pool

```go
func worker(id int, jobs <-chan Job, results chan<- Result) {
    for job := range jobs {
        results <- process(job)
    }
}

func main() {
    jobs := make(chan Job, 100)
    results := make(chan Result, 100)

    // Start workers
    for w := 0; w < numWorkers; w++ {
        go worker(w, jobs, results)
    }

    // Send jobs
    for _, job := range allJobs {
        jobs <- job
    }
    close(jobs)

    // Collect results
    for range allJobs {
        r := <-results
        handle(r)
    }
}
```

### Fan-out, Fan-in

```go
// Fan-out: multiple goroutines read from same channel
// Fan-in: single goroutine reads from multiple channels
func fanIn(channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    merged := make(chan int)

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                merged <- v
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}
```

### Resource Pool (Leaky Buffer)

```go
var pool = make(chan *Buffer, 100)

func getBuffer() *Buffer {
    select {
    case b := <-pool:
        return b  // reuse
    default:
        return new(Buffer)  // allocate new
    }
}

func putBuffer(b *Buffer) {
    b.Reset()
    select {
    case pool <- b:
        // returned to pool
    default:
        // pool full — GC will reclaim
    }
}
```

### Done Channel / Context Cancellation

```go
// Done channel pattern
func worker(done <-chan struct{}, jobs <-chan Job) {
    for {
        select {
        case <-done:
            return  // shutdown
        case job := <-jobs:
            process(job)
        }
    }
}

// Context pattern (preferred)
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return
        case job := <-jobs:
            process(job)
        }
    }
}
```

## sync Primitives

| Type | Use when |
|------|----------|
| `sync.Mutex` | Protect shared state — simple lock |
| `sync.RWMutex` | Read-heavy workloads — multiple readers, single writer |
| `sync.WaitGroup` | Wait for group of goroutines to finish |
| `sync.Once` | One-time initialization |
| `sync.Map` | Concurrent map (specialized — often regular map + mutex is better) |
| `sync.Pool` | Reuse temporary objects to reduce GC pressure |

```go
// WaitGroup — wait for all goroutines
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)
    go func(it Item) {
        defer wg.Done()
        process(it)
    }(item)
}
wg.Wait()

// Once — thread-safe lazy init
var once sync.Once
var instance *Config
func GetConfig() *Config {
    once.Do(func() {
        instance = loadConfig()
    })
    return instance
}
```

## Context Rules

| Rule | Detail |
|------|--------|
| Always first parameter | `func DoWork(ctx context.Context, ...)` |
| Never store in struct | Pass as parameter — storing hides lifecycle |
| Never use custom context types | Only `context.Context` |
| `context.Background()` in `main` | Entry point only |
| `req.Context()` in HTTP handlers | Already provided by framework |
| Propagate always | Pass to every function that accepts it |

```go
// BAD: context in struct
type Service struct {
    ctx context.Context  // Don't do this
}

// GOOD: context as first parameter
func (s *Service) Process(ctx context.Context, data Data) error {
    return s.db.QueryContext(ctx, query)
}
```

## Prefer Synchronous Functions

```go
// BAD: forcing async on callers
func FetchData() <-chan Result {
    ch := make(chan Result)
    go func() { ch <- doFetch() }()
    return ch
}

// GOOD: let caller add concurrency if needed
func FetchData(ctx context.Context) (Result, error) {
    return doFetch(ctx)
}

// Caller can easily wrap if they need async:
go func() { result, err := FetchData(ctx); ... }()
```

---

## Parallelization

```go
numCPU := runtime.NumCPU()
c := make(chan bool, numCPU)

for i := 0; i < numCPU; i++ {
    go func(start, end int) {
        processRange(data[start:end])
        c <- true
    }(i*len(data)/numCPU, (i+1)*len(data)/numCPU)
}

for i := 0; i < numCPU; i++ {
    <-c  // wait for all
}
```

## Common Mistakes

| Mistake | Why Bad | Fix |
|---------|---------|-----|
| Goroutine leak | No exit path, channels never closed | Use context, done channel, or WaitGroup |
| Closure over loop variable | All goroutines share same variable | Pass as argument: `go func(v T) {...}(v)` |
| Naked goroutine in production | No error handling, no shutdown | Use `errgroup`, context cancellation |
| Send on closed channel | Panics | Only sender closes; coordinate with done signal |
| Race on shared map | Runtime panic | Use `sync.Mutex` or `sync.Map` |
| Forgetting `wg.Add` before `go` | WaitGroup may finish early | Always `Add` before launching goroutine |
| Blocking main without wait | Program exits, goroutines killed | `WaitGroup.Wait()` or `select{}` |
| Buffered channel as queue | May hide backpressure | Size buffers intentionally |
| Goroutine with unclear lifetime | Leaks, can't shutdown | Document when/how it exits |
| Context stored in struct | Hides lifecycle | Pass as first parameter always |
| Async function by default | Harder to test, compose | Return sync; let caller wrap |
