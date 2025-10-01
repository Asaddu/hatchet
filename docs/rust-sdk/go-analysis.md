# Hatchet Go SDK: Architectural Analysis for Rust Implementation

## Executive Summary

The Hatchet Go SDK (~3,932 lines of code) is a well-architected, reflection-based workflow orchestration SDK. It provides a clean, type-safe API for defining workflows, tasks, and workers with sophisticated features like DAG execution, concurrency control, rate limiting, and durable task execution.

## Directory Structure

```
/home/mikelasaddu/Projects/hatchet/sdks/go/
├── hatchet.go           # Main package documentation & exported types
├── client.go            # Client facade with feature clients
├── worker.go            # Worker configuration and options
├── workflow.go          # Workflow and task definitions
├── internal/
│   ├── declaration.go   # Core workflow declaration logic (~900 lines)
│   └── task/            # Task-specific implementations
├── features/            # Feature-specific clients (12 modules)
│   ├── workflows.go
│   ├── runs.go
│   ├── metrics.go
│   ├── ratelimits.go
│   ├── crons.go
│   ├── schedules.go
│   └── ...
└── examples/            # 17 comprehensive examples
    ├── simple/
    ├── dag/
    ├── streaming/
    ├── durable/
    └── ...
```

## Core Design Patterns

### 1. **Builder Pattern with Functional Options**

The SDK extensively uses functional options for configuration:

```go
// Workflow options
client.NewWorkflow("name",
    hatchet.WithWorkflowCron("*/5 * * * *"),
    hatchet.WithWorkflowEvents("event.created"),
    hatchet.WithWorkflowConcurrency(types.Concurrency{...}))

// Task options
workflow.NewTask("task-name", fn,
    hatchet.WithRetries(3),
    hatchet.WithParents(task1, task2),
    hatchet.WithExecutionTimeout(30*time.Second))

// Worker options
client.NewWorker("worker",
    hatchet.WithWorkflows(workflow),
    hatchet.WithSlots(100),
    hatchet.WithLabels(map[string]any{...}))
```

**Rust Translation:** Use the builder pattern with typed builders and `impl Into<Option<T>>` for optional parameters.

### 2. **Reflection-Based Type Conversion**

The SDK uses Go's reflection extensively for:
- Converting `map[string]interface{}` to strongly-typed structs
- Validating function signatures at runtime
- Dynamic task registration

**Key Pattern:**
```go
func convertInputToType(input any, expectedType reflect.Type) reflect.Value {
    // JSON marshal/unmarshal for type conversion
    jsonData, _ := json.Marshal(input)
    result := reflect.New(expectedType)
    json.Unmarshal(jsonData, result.Interface())
    return result.Elem()
}
```

**Rust Translation:** Use serde for serialization with strong typing. Leverage procedural macros for compile-time validation instead of runtime reflection.

### 3. **Two-Tier Client Architecture**

```
┌─────────────────────────────────────┐
│   SDK Client (sdks/go/client.go)   │  <- High-level, user-facing API
│   - Feature clients (lazy-loaded)   │
│   - Simplified workflows            │
└──────────────┬──────────────────────┘
               │
               v
┌─────────────────────────────────────┐
│  Legacy Client (pkg/client/client.go)│  <- Low-level, internal API
│  - gRPC connection management       │
│  - Protocol buffer handling         │
│  - Admin/Event/Dispatcher clients   │
└─────────────────────────────────────┘
```

**Rust Translation:** Similar layering with:
- High-level SDK crate for users
- Low-level client crate for protocol details
- Use traits for client interfaces

### 4. **Workflow Declaration System**

The workflow system uses a sophisticated declaration pattern:

```go
type WorkflowDeclaration[I, O any] interface {
    Task(opts, fn func(ctx, input I) (interface{}, error))
    Run(ctx, input I) (*O, error)
    Dump() (*protobuf.Request, []NamedFunction, ...)
}
```

Key features:
- **Generic input/output types** (I, O)
- **Task registration** with reflection-based wrapping
- **Dump method** serializes to protobuf for server transmission
- **Separation of concerns**: declaration vs. execution

**Rust Translation:**
- Use trait objects or enums for dynamic dispatch
- Leverage async traits for execution
- Proc macros for workflow DSL
- Strong typing with generics

### 5. **Dual Worker Pattern: Durable vs. Non-Durable**

```go
type Worker struct {
    nonDurable *worker.Worker  // Regular tasks
    durable    *worker.Worker  // Long-running, resumable tasks
}
```

Workers are created lazily based on workflow composition:
- If workflow has durable tasks → create separate durable worker
- Durable workers have different slot limits (1000 vs 100 default)
- Both start concurrently using `errgroup`

**Rust Translation:**
- Use enum variants or trait objects
- Leverage tokio for concurrent worker execution
- Consider actor model (e.g., `actix` or custom channels)

### 6. **Context Abstraction**

Two context types with different capabilities:

```go
type Context = pkgWorker.HatchetContext
type DurableContext = pkgWorker.DurableHatchetContext

// Regular context
ctx.WorkflowInput(&input)
ctx.ParentOutput(task, &output)
ctx.RetryCount()
ctx.PutStream("message")

// Durable context (extends regular)
ctx.SleepFor(30 * time.Second)  // Persists across restarts
```

**Rust Translation:**
- Use traits for context capabilities
- `DurableContext: Context` trait inheritance
- Async methods for operations

## Error Handling Patterns

### Current Approach:
```go
// Panic for programmer errors
if name == "" {
    panic("task name cannot be empty")
}

// Return errors for runtime failures
result, err := workflow.Run(ctx, input)
if err != nil {
    return nil, err
}
```

### Rust Approach:
- Use `Result<T, E>` for all fallible operations
- Custom error types with `thiserror`
- `anyhow` for application-level error handling
- Builder validation at compile-time when possible

## Connection & Authentication

The SDK delegates to the pkg/client layer which handles:
1. **Config Loading**: Environment variables, config files, or explicit options
2. **gRPC Connection**: TLS, retries, keepalive configuration
3. **REST API Client**: Generated from OpenAPI specs
4. **Token Management**: Bearer token authentication

**Key Configuration Sources:**
- `HATCHET_CLIENT_TOKEN` environment variable
- Config file at `~/.hatchet/config.yaml`
- Programmatic options via `ClientOpt` functions

**Rust Translation:**
- Use `tonic` for gRPC
- `reqwest` for REST API
- Config crate for environment/file loading
- Builder pattern for client construction

## Testing Patterns

From `/home/mikelasaddu/Projects/hatchet/sdks/go/workflow_test.go`:

```go
func TestConvertInputToType_MapToStruct(t *testing.T) {
    input := map[string]interface{}{
        "name": "Alice",
        "age": 25,
    }
    expectedType := reflect.TypeOf(TestStruct{})

    result := convertInputToType(input, expectedType)

    assert.Equal(t, expected, result.Interface())
}
```

**Testing Strategy:**
- Unit tests for type conversion logic
- Integration tests against real Hatchet server
- Example-based documentation (17 runnable examples)

**Rust Translation:**
- Unit tests with `#[test]` and `assert!`
- Doc tests in documentation
- Integration tests in `tests/` directory
- Property-based testing with `proptest` or `quickcheck`

## Documentation Structure

The Go SDK uses **literate programming** principles:

1. **Package-level docs** (`hatchet.go`):
   - Complete usage example
   - Links to 17 different example scenarios
   - API surface documentation

2. **17 Runnable Examples** covering:
   - Simple tasks
   - DAG workflows
   - Event-driven workflows
   - Cron scheduling
   - Rate limiting
   - Concurrency control
   - Child workflows
   - Durable tasks
   - Streaming
   - Error handling
   - Timeouts
   - Cancellations
   - Retries
   - Priority
   - Bulk operations
   - Sticky workers
   - Conditions

3. **Inline Documentation**: Extensive godoc comments

**Rust Translation:**
- Use `rustdoc` with extensive examples
- Create examples/ directory with runnable code
- README with quickstart guide
- API documentation with doctests

## Feature Clients (Lazy-Loaded Pattern)

```go
type Client struct {
    legacyClient v0Client.Client

    // Feature clients (lazy loaded)
    metrics    *features.MetricsClient
    rateLimits *features.RateLimitsClient
    crons      *features.CronsClient
    // ... 9 total feature clients
}

func (c *Client) Metrics() *features.MetricsClient {
    if c.metrics == nil {
        c.metrics = features.NewMetricsClient(...)
    }
    return c.metrics
}
```

**Benefits:**
- Deferred initialization
- Clear API surface
- Easy to extend

**Rust Translation:**
- Use `OnceCell` or `lazy_static!` for lazy initialization
- Consider using `Arc<Mutex<Option<T>>>` for thread-safe lazy init
- Or expose constructors and let users compose

## Key Idioms & Conventions

### 1. **Standalone Tasks**
Simplified API for single-task workflows:
```go
task := client.NewStandaloneTask("name", fn)
result, err := task.Run(ctx, input)
```

### 2. **Child Workflows**
Type-safe child workflow execution:
```go
result, err := workflow.RunAsChild(ctx, input, RunAsChildOpts{
    Sticky: &true,
    Key: &"unique-key",
})
```

### 3. **Type-Safe Result Extraction**
```go
workflowResult, _ := workflow.Run(ctx, input)
taskResult := workflowResult.TaskOutput("task-name")
var output MyType
taskResult.Into(&output)  // JSON unmarshal
```

### 4. **Condition System**
Declarative conditions for task execution:
```go
hatchet.WithWaitFor(
    hatchet.OrCondition(
        hatchet.SleepCondition(5 * time.Minute),
        hatchet.UserEventCondition("approval", "approved == true"),
    ),
)
```

**Rust Translation:**
- Use enum for condition types
- Builder pattern for complex conditions
- Trait for condition evaluation

## Critical Files & Their Purposes

| File | Lines | Purpose |
|------|-------|---------|
| `internal/declaration.go` | ~900 | Core workflow declaration, type conversion, task registration |
| `client.go` | ~600 | Main SDK client facade, feature client management |
| `workflow.go` | ~550 | Workflow/task builder API, functional options |
| `hatchet.go` | ~100 | Package docs, exported types, condition helpers |
| `worker.go` | ~65 | Worker configuration and lifecycle |
| `features/*.go` | ~300 | Feature-specific REST API clients (12 files) |

## Rust Implementation Recommendations

### 1. **Leverage Strong Typing**
```rust
// Instead of reflection, use generics + serde
trait Task<I, O>
where
    I: DeserializeOwned,
    O: Serialize,
{
    async fn execute(&self, ctx: Context, input: I) -> Result<O>;
}
```

### 2. **Async/Await Instead of Goroutines**
```rust
// Worker lifecycle
async fn start(&self) -> Result<JoinHandle<()>> {
    let (durable, non_durable) = tokio::join!(
        self.durable_worker.start(),
        self.non_durable_worker.start()
    );
    // ...
}
```

### 3. **Builder Pattern with Type States**
```rust
// Enforce required fields at compile time
WorkflowBuilder::new("name")
    .with_cron("*/5 * * * *")
    .build()  // Returns Result if validation fails
```

### 4. **Proc Macros for Workflow DSL**
```rust
#[workflow(name = "my-workflow")]
async fn my_workflow(ctx: Context, input: MyInput) -> Result<MyOutput> {
    #[task(retries = 3, timeout = "30s")]
    async fn step1(ctx: Context, input: MyInput) -> Result<Step1Output> {
        // ...
    }

    let result = step1.run(ctx, input).await?;
    // ...
}
```

### 5. **Error Handling with Context**
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum HatchetError {
    #[error("workflow not found: {name}")]
    WorkflowNotFound { name: String },

    #[error("task execution failed: {source}")]
    TaskFailed {
        #[from]
        source: Box<dyn std::error::Error + Send + Sync>
    },

    #[error(transparent)]
    Transport(#[from] tonic::Status),
}
```

### 6. **Trait-Based Context**
```rust
#[async_trait]
pub trait Context: Send + Sync {
    async fn workflow_input<T: DeserializeOwned>(&self) -> Result<T>;
    async fn parent_output<T: DeserializeOwned>(&self, task: &Task) -> Result<T>;
    fn retry_count(&self) -> u32;
    async fn put_stream(&self, msg: impl Into<String>);
}

#[async_trait]
pub trait DurableContext: Context {
    async fn sleep_for(&self, duration: Duration) -> Result<()>;
}
```

### 7. **Feature Modules**
```rust
// src/client.rs
pub struct Client {
    inner: Arc<InnerClient>,
}

impl Client {
    pub fn workflows(&self) -> WorkflowsClient {
        WorkflowsClient::new(self.inner.clone())
    }

    pub fn runs(&self) -> RunsClient {
        RunsClient::new(self.inner.clone())
    }
}
```

## What Translates Well to Rust

✅ **Excellent Translation:**
- Builder patterns with functional options → Type-state builders
- Feature client separation → Module structure
- Dual worker pattern → Enum variants or trait objects
- Context abstraction → Trait inheritance
- REST/gRPC clients → `reqwest` and `tonic`

✅ **Good Translation (with adaptations):**
- Type conversion → Serde instead of reflection
- Workflow declaration → Proc macros + traits
- Error handling → Result types with custom errors
- Testing patterns → Similar structure with Rust idioms

⚠️ **Requires Careful Design:**
- Runtime reflection → Compile-time proc macros + generics
- Dynamic task registration → Static type system requires different approach
- Nil/null handling → Option types and explicit handling

## Conclusion

The Go SDK provides a mature, well-designed foundation for a Rust implementation. The key challenge is replacing reflection-based patterns with Rust's compile-time type system, which can be achieved through:

1. **Procedural macros** for workflow/task definitions
2. **Trait-based abstractions** for contexts and clients
3. **Strong typing with generics** for type-safe workflows
4. **Async/await** for concurrent execution
5. **Builder patterns** for configuration

The result will be a more type-safe, performant SDK with compile-time guarantees that the Go version achieves at runtime.
