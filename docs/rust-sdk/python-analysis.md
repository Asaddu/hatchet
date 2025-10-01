# Hatchet Python SDK: Architectural Analysis for Rust Implementation

## Executive Summary

The Hatchet Python SDK (~129k LOC) is a distributed task queue client with sophisticated workflow orchestration capabilities. It uses a layered architecture with clear separation between API clients, workflow definitions, execution runtime, and communication protocols.

## 1. Directory Organization

```
hatchet_sdk/
├── client.py                    # Main client orchestrator
├── hatchet.py                   # High-level API facade
├── config.py                    # Configuration with Pydantic
├── connection.py                # gRPC connection management
├── exceptions.py                # Error hierarchy
├── metadata.py, token.py        # Auth utilities
├── clients/                     # API client implementations
│   ├── admin.py                 # Workflow registration & triggering
│   ├── events.py                # Event publishing
│   ├── dispatcher/              # Worker<->server communication
│   │   ├── dispatcher.py        # gRPC dispatcher client
│   │   └── action_listener.py   # Streaming action listener
│   ├── listeners/               # Various event listeners
│   └── rest/                    # REST API (OpenAPI generated)
├── features/                    # Feature-specific clients
│   ├── cel.py, cron.py, filters.py, logs.py
│   ├── metrics.py, rate_limits.py, runs.py
│   ├── scheduled.py, workflows.py, workers.py
│   └── stubs.py, tenant.py
├── runnables/                   # Workflow & task definitions
│   ├── workflow.py              # Workflow builder/DSL
│   ├── task.py                  # Task decorator & execution
│   ├── action.py                # Runtime action representation
│   ├── types.py                 # Type definitions & configs
│   └── contextvars.py           # Context variable management
├── context/                     # Execution context
│   ├── context.py               # Regular & durable context
│   └── worker_context.py        # Worker-level context
├── worker/                      # Worker runtime
│   ├── worker.py                # Worker orchestrator (multiprocess)
│   ├── action_listener_process.py  # Subprocess for gRPC listening
│   └── runner/                  # Task execution engine
│       └── run_loop_manager.py  # Task scheduling & execution
├── contracts/                   # Protobuf generated code
│   ├── dispatcher_pb2.py, events_pb2.py
│   ├── workflows_pb2.py
│   └── v1/                      # Versioned contracts
└── utils/                       # Utility functions
    ├── aio.py, backoff.py, serde.py
    ├── opentelemetry.py, typing.py
    └── timedelta_to_expression.py
```

## 2. Key Architectural Patterns

### 2.1 Layered Architecture

**Three-tier design:**
1. **High-level API** (`Hatchet` class) - Decorator-based workflow DSL
2. **Client Layer** (`Client` class) - Protocol abstraction (gRPC/REST)
3. **Protocol Layer** - Protobuf contracts + gRPC stubs

**Rust Translation:**
- Use trait-based abstraction for client interfaces
- Builder pattern for complex configurations
- Macro-based decorators could replace Python decorators

### 2.2 Configuration Management

**Pattern: Pydantic Settings with Environment Variables**
```python
# Python
class ClientConfig(BaseSettings):
    model_config = create_settings_config(env_prefix="HATCHET_CLIENT_")
    token: str = ""
    host_port: str = "localhost:7070"
    tls_config: ClientTLSConfig = Field(default_factory=...)
```

**Rust Equivalent:**
- Use `serde` + `config` crate or `figment`
- Derive-based validation with custom validators
- Environment variable parsing with prefix support

### 2.3 Workflow Definition DSL

**Pattern: Decorator-based Builder with Type Safety**
```python
@hatchet.workflow(name="my-workflow", on_events=["user.created"])
def workflow(input_validator: UserInput):
    @workflow.task(timeout=60, retries=3)
    def step1(input: UserInput, ctx: Context) -> StepOutput:
        return {"result": "..."}
```

**Key Components:**
- **Workflow** - Container for tasks with metadata
- **Task** - Executable unit with timeout/retry config
- **Context** - Runtime execution context with DAG access
- **Input Validation** - Pydantic models for type safety

**Rust Translation:**
- Procedural macros for workflow/task decorators
- Generic trait bounds for input/output validation
- Builder pattern for workflow configuration
- Use `serde` for serialization

### 2.4 Worker Architecture

**Pattern: Multi-process Worker Pool**
```python
Worker
├── Main Process (orchestration)
├── Action Listener Process (gRPC streaming)
│   └── Receives work from dispatcher
└── Action Runner (thread/async pool)
    └── Executes tasks
```

**Critical Features:**
- Multiprocess isolation for crash resilience
- Async gRPC streaming for action listening
- Queue-based communication between processes
- Graceful shutdown with signal handling
- Health check HTTP server

**Rust Translation:**
- Use `tokio` runtime with multi-threaded scheduler
- `tokio::sync::mpsc` for inter-task communication
- Consider actor pattern (`actix` or custom) for isolation
- `tower` for service abstraction
- Signal handling with `tokio::signal`

### 2.5 Error Handling

**Pattern: Custom Exception Hierarchy**
```python
class NonRetryableException(Exception): pass
class DedupeViolationError(Exception): pass
class TaskRunError(Exception):
    # Serializable error with trace
    def serialize(self, include_metadata: bool) -> str
```

**Features:**
- Retryable vs non-retryable distinction
- Error serialization for remote communication
- Stack trace preservation
- Exception groups for parallel failures

**Rust Translation:**
- `thiserror` for error types
- `anyhow` for context-rich errors
- Custom `Result<T, E>` types
- Enum-based error variants:
  ```rust
  #[derive(Error, Debug)]
  pub enum HatchetError {
      #[error("non-retryable: {0}")]
      NonRetryable(String),
      #[error("dedupe violation")]
      DedupeViolation,
      // ...
  }
  ```

### 2.6 Context & Dependency Injection

**Pattern: Context as State Container + DI**
```python
class Context:
    def task_output(self, task: Task[T, R]) -> R
    def spawn_workflow(self, name: str, input: dict) -> WorkflowRunRef
    @property
    def workflow_input(self) -> JSONSerializableMapping
```

**Rust Translation:**
- Struct-based context with lifetime management
- Arc/Mutex for shared state
- Trait objects for dynamic dispatch where needed
- Builder pattern for context construction

### 2.7 Communication Protocols

**Dual Protocol Design:**
1. **gRPC** - Worker↔Dispatcher (bidirectional streaming)
   - Action listening (server→worker)
   - Event reporting (worker→server)
   - Protobuf contracts

2. **REST API** - Admin operations (OpenAPI generated)
   - Workflow management
   - Metrics/logs retrieval
   - Manual triggers

**Rust Translation:**
- `tonic` for gRPC (excellent Rust support)
- `reqwest` or `hyper` for REST
- Generate code from `.proto` files with `prost`
- Consider `tonic-build` for build-time codegen

## 3. Core Client Architecture

### 3.1 Client Composition Pattern

```python
class Client:
    def __init__(self, config: ClientConfig):
        self.dispatcher = DispatcherClient(config)
        self.event = EventClient(config)
        self.listener = RunEventListenerClient(config)
        self.cel = CELClient(config)
        self.cron = CronClient(config)
        # ... more feature clients
```

**Pattern:** Composition over inheritance with feature-specific clients

**Rust Translation:**
```rust
pub struct Client {
    config: Arc<ClientConfig>,
    dispatcher: DispatcherClient,
    event: EventClient,
    // ... trait objects or concrete types
}

impl Client {
    pub fn new(config: ClientConfig) -> Self {
        let config = Arc::new(config);
        Self {
            dispatcher: DispatcherClient::new(config.clone()),
            event: EventClient::new(config.clone()),
            // ...
            config,
        }
    }
}
```

### 3.2 Connection Management

**Pattern: Lazy gRPC Channel Creation**
```python
class DispatcherClient:
    def __init__(self, config):
        self.aio_client: DispatcherStub | None = None

    def _get_or_create_client(self):
        if self.client is None:
            conn = new_conn(self.config, False)
            self.client = DispatcherStub(conn)
        return self.client
```

**Features:**
- TLS/mTLS support
- Keepalive configuration
- Connection pooling
- Separate sync/async clients

**Rust Translation:**
- `OnceCell` or `std::sync::Once` for lazy init
- `tonic::transport::Channel` with builder pattern
- Connection pooling via `tower` middleware

### 3.3 Authentication Pattern

**JWT Token in Metadata:**
```python
def get_metadata(token: str):
    return (("authorization", f"Bearer {token}"),)
```

**Rust Translation:**
```rust
fn get_metadata(token: &str) -> tonic::metadata::MetadataMap {
    let mut map = MetadataMap::new();
    map.insert("authorization", format!("Bearer {}", token).parse().unwrap());
    map
}
```

## 4. Critical Files & Purposes

| File | Purpose | Rust Equivalent |
|------|---------|-----------------|
| `hatchet.py` | User-facing API facade | `lib.rs` main entry point |
| `client.py` | Client orchestrator | `client.rs` |
| `config.py` | Configuration schema | `config.rs` with `serde` |
| `worker/worker.py` | Worker runtime | `worker.rs` with `tokio` |
| `runnables/workflow.py` | Workflow builder | `workflow.rs` with proc macros |
| `runnables/task.py` | Task decorator & execution | `task.rs` |
| `context/context.py` | Execution context | `context.rs` |
| `connection.py` | gRPC connection factory | `connection.rs` |
| `clients/dispatcher/dispatcher.py` | gRPC dispatcher client | `dispatcher.rs` |
| `clients/events.py` | Event publishing | `events.rs` |
| `exceptions.py` | Error types | `error.rs` with `thiserror` |

## 5. Testing Patterns

**Structure:**
```
tests/
├── test_client.py              # Unit tests
├── test_serde.py               # Serialization tests
├── test_task_default_fallbacks.py
├── worker_fixture.py           # Shared test fixtures
└── <feature>/                  # Integration tests per feature
    ├── test_<feature>.py
    └── worker.py
```

**Key Patterns:**
- Pytest with async support (`pytest-asyncio`)
- Fixture-based setup/teardown
- Integration tests alongside unit tests
- Environment variable mocking

**Rust Translation:**
- Unit tests with `#[test]` and `#[tokio::test]`
- Integration tests in `tests/` directory
- `mockall` for mocking
- `testcontainers` for integration tests

## 6. Documentation Structure

**Approach: Code-generated API docs + examples**
```
docs/
├── client.md                   # High-level guides
├── context.md
├── feature-clients/            # Per-feature documentation
│   ├── cron.md, logs.md, etc.
└── generator/                  # LLM-based doc generation
```

**Pattern:** MkDocs + docstrings → markdown

**Rust Translation:**
- `rustdoc` for API documentation
- Doc comments with examples
- `mdBook` for guides
- Integration with `docs.rs`

## 7. Recommendations for Rust Implementation

### 7.1 High Priority Patterns

✅ **Adopt:**
1. **Layered architecture** - Clean separation of concerns
2. **Builder pattern** - For workflow/task configuration
3. **Type-safe DSL** - Use macros for ergonomic API
4. **Dual protocol design** - gRPC + REST
5. **Feature flags** - For optional dependencies
6. **Configuration from env** - Using `config` crate

✅ **Adapt:**
1. **Multiprocess → Multi-threaded** - Rust's memory safety allows safe concurrency
2. **Decorators → Macros** - Procedural macros for `#[workflow]`, `#[task]`
3. **Pydantic → Serde** - Type-safe serialization/validation
4. **Exception → Result** - Idiomatic error handling

### 7.2 Architecture Recommendations

```rust
// Proposed structure
hatchet-sdk/
├── src/
│   ├── lib.rs                  // Public API
│   ├── client.rs               // Client orchestrator
│   ├── config.rs               // Configuration
│   ├── worker.rs               // Worker runtime
│   ├── workflow/               // Workflow DSL
│   │   ├── mod.rs
│   │   ├── builder.rs
│   │   └── macros.rs
│   ├── task.rs                 // Task execution
│   ├── context.rs              // Execution context
│   ├── error.rs                // Error types
│   ├── clients/                // Protocol clients
│   │   ├── dispatcher.rs
│   │   ├── events.rs
│   │   └── rest.rs
│   └── proto/                  // Generated protobuf code
├── macros/                     // Procedural macros crate
└── examples/
```

### 7.3 Key Design Decisions

**1. Async Runtime:** Use `tokio` exclusively
- Mature ecosystem
- Excellent gRPC support via `tonic`
- Multi-threaded scheduler

**2. Type Safety:** Leverage Rust's type system
```rust
// Generic workflow with input/output types
pub struct Workflow<I, O> {
    tasks: Vec<Task<I, O>>,
    config: WorkflowConfig,
}

// Type-safe task chaining
impl<I, O> Workflow<I, O> {
    pub fn task<F, T>(&mut self, f: F) -> &mut Self
    where
        F: Fn(I, Context) -> Result<T>,
        T: Serialize,
    {
        // ...
    }
}
```

**3. Macro Design:** Progressive disclosure
```rust
// Simple case
#[task]
fn simple_task(input: MyInput, ctx: Context) -> Result<Output> {
    Ok(Output { result: "done" })
}

// Advanced case
#[task(
    timeout = "60s",
    retries = 3,
    rate_limit = "10/s"
)]
async fn advanced_task(input: MyInput, ctx: Context) -> Result<Output> {
    // ...
}
```

**4. Error Handling:** Explicit Result types
```rust
pub type HatchetResult<T> = Result<T, HatchetError>;

#[derive(Error, Debug)]
pub enum HatchetError {
    #[error("task failed: {0}")]
    TaskFailed(#[from] TaskError),

    #[error("connection error: {0}")]
    Connection(#[from] tonic::Status),

    #[error(transparent)]
    Other(#[from] anyhow::Error),
}
```

**5. Concurrency Model:** Actor-like pattern
```rust
// Worker as actor
pub struct Worker {
    rx: mpsc::Receiver<Action>,
    executor: TaskExecutor,
}

impl Worker {
    pub async fn run(mut self) {
        while let Some(action) = self.rx.recv().await {
            self.executor.execute(action).await;
        }
    }
}
```

### 7.4 Dependencies Recommendation

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
tonic = "0.12"
prost = "0.13"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
thiserror = "2.0"
anyhow = "1.0"
config = "0.14"
reqwest = { version = "0.12", features = ["json"] }
tower = "0.5"
tracing = "0.1"
tracing-subscriber = "0.3"

[build-dependencies]
tonic-build = "0.12"
```

### 7.5 What Doesn't Translate Well

⚠️ **Challenges:**
1. **Dynamic typing** - Python's dict-based payloads require careful Rust design
   - Solution: Use `serde_json::Value` or strongly-typed enums
2. **Decorator syntax** - Less ergonomic in Rust
   - Solution: Proc macros (compile-time cost)
3. **Multi-process isolation** - Python uses for crash safety
   - Solution: Thread-based with panic recovery + supervision
4. **OpenAPI client generation** - Python has better tooling
   - Solution: `openapi-generator` or manual implementation

## 8. Implementation Priorities

**Phase 1: Core Foundation**
1. Configuration system
2. gRPC connection management
3. Basic client structure
4. Error types

**Phase 2: Workflow DSL**
1. Workflow builder API
2. Task macros
3. Context implementation
4. Type-safe input/output

**Phase 3: Worker Runtime**
1. Worker orchestration
2. Task execution engine
3. Action listener
4. Signal handling

**Phase 4: Feature Clients**
1. Dispatcher client
2. Event client
3. REST API client
4. Additional features (cron, metrics, etc.)

---

## Conclusion

The Python SDK demonstrates a well-architected, production-ready distributed system client. Its patterns translate well to Rust with adaptations for ownership, type safety, and async/await. The macro-based DSL, type-safe serialization, and actor model will provide a more robust, performant SDK while maintaining API ergonomics.

**Key Takeaway:** Focus on leveraging Rust's strengths (zero-cost abstractions, fearless concurrency, type safety) while preserving the Python SDK's intuitive workflow definition patterns.
