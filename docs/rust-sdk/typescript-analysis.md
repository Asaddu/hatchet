# Hatchet TypeScript SDK: Architectural Analysis for Rust Implementation

## Executive Summary

The Hatchet TypeScript SDK provides a type-safe workflow orchestration client with sophisticated features including DAG execution, durable tasks, conditional execution, and real-time streaming. It leverages TypeScript's type system to provide compile-time safety while maintaining runtime flexibility.

## Directory Structure

```
sdks/typescript/
├── src/
│   ├── clients/          # Core client implementations
│   │   ├── admin/        # Admin operations (workflow triggers)
│   │   ├── dispatcher/   # gRPC dispatcher client (worker-server communication)
│   │   ├── event/        # Event client (pub/sub triggers)
│   │   ├── hatchet-client/ # Legacy client wrapper
│   │   ├── listeners/    # Event listeners (run, durable)
│   │   ├── rest/         # HTTP REST API client
│   │   └── worker/       # Worker runtime
│   ├── v1/               # V1 API (current, recommended)
│   │   ├── client/       # Main client and worker implementations
│   │   ├── conditions/   # Conditional execution logic
│   │   ├── declaration.ts # Workflow/task declaration types
│   │   └── task.ts       # Task configuration types
│   ├── protoc/           # Generated gRPC/protobuf code
│   ├── util/             # Utilities (config, errors, logging)
│   ├── examples/         # Usage examples
│   ├── workflow.ts       # V0 workflow definitions (deprecated)
│   ├── step.ts           # V0 step definitions (deprecated)
│   └── index.ts          # Public API exports
```

## Core Client Architecture

### 1. **HatchetClient (Main Entry Point)**

**File:** `src/v1/client/client.ts`

**Key Patterns:**
- **Builder Pattern**: Factory method `HatchetClient.init()` for initialization
- **Lazy Initialization**: Feature clients (metrics, crons, schedules) created on-demand via getters
- **Composition**: Wraps multiple specialized clients (admin, events, runs, workers)
- **Configuration Cascade**: Config from params → YAML file → environment variables

**Core Structure:**
```typescript
class HatchetClient {
  _v0: LegacyHatchetClient;        // Backwards compatibility
  _api: Api;                        // REST client
  _listener: RunListenerClient;    // gRPC listener

  // Lazy-loaded feature clients
  get metrics() { ... }
  get crons() { ... }
  get scheduled() { ... }
  get events() { ... }
  get runs() { ... }
  get workflows() { ... }
  get workers() { ... }

  // Factory methods
  workflow<I, O>(opts): WorkflowDeclaration<I, O>
  task<I, O>(opts): TaskWorkflowDeclaration<I, O>
  durableTask<I, O>(opts): TaskWorkflowDeclaration<I, O>

  // Execution methods
  run<I, O>(workflow, input, opts): Promise<O>
  runNoWait<I, O>(workflow, input, opts): Promise<WorkflowRunRef<O>>
  worker(name, opts): Promise<Worker>
}
```

**Rust Translation:**
- Use trait-based composition instead of inheritance
- Builder pattern with `ClientBuilder::new().token(...).build()`
- Use `Arc<Mutex<>>` or `OnceCell` for lazy initialization
- Strong typing with generics for workflow input/output

### 2. **Configuration System**

**File:** `src/util/config-loader/config-loader.ts`

**Key Patterns:**
- **Config Priority**: Explicit config > YAML file > Environment variables
- **Validation**: Uses Zod schema validation (maps to `serde` + custom validation in Rust)
- **JWT Token Parsing**: Extracts tenant ID and addresses from JWT claims
- **TLS Strategy**: Supports `tls`, `mtls`, `none` modes

**Configuration Schema:**
```typescript
{
  token: string,
  tls_config: {
    tls_strategy: 'tls' | 'mtls' | 'none',
    cert_file?: string,
    ca_file?: string,
    key_file?: string,
    server_name?: string
  },
  host_port: string,      // gRPC address
  api_url: string,        // REST API address
  log_level: 'OFF' | 'DEBUG' | 'INFO' | 'WARN' | 'ERROR',
  tenant_id: string,
  namespace?: string
}
```

**Rust Translation:**
- Use `serde` for JSON/YAML deserialization
- `config` crate for multi-source configuration
- `jsonwebtoken` crate for JWT parsing
- `rustls` for TLS configuration

### 3. **Workflow Definition Pattern**

**File:** `src/v1/declaration.ts`

**Key Patterns:**
- **Type-Safe Builder**: Fluent API for workflow construction
- **Generic Constraints**: Input/output types enforced at compile time
- **Task Graph**: DAG construction via `parents` array
- **Conditional Execution**: `waitFor`, `cancelIf`, `skipIf` conditions

**Workflow Declaration:**
```typescript
class WorkflowDeclaration<I, O> {
  definition: {
    name: string,
    description?: string,
    _tasks: CreateWorkflowTaskOpts[],
    _durableTasks: CreateWorkflowDurableTaskOpts[],
    onFailure?: TaskOpts,
    onSuccess?: TaskOpts
  }

  task<Name extends string, Fn>(opts): CreateWorkflowTaskOpts
  durableTask<Name, Fn>(opts): CreateWorkflowDurableTaskOpts
  onFailure<Name, L>(opts): CreateWorkflowTaskOpts
  onSuccess<Name, L>(opts): CreateWorkflowTaskOpts

  run(input: I): Promise<O>
  runNoWait(input: I): Promise<WorkflowRunRef<O>>
  schedule(enqueueAt: Date, input: I): Promise<ScheduledWorkflows>
  cron(name, expression, input): Promise<CronWorkflows>
}
```

**Rust Translation:**
- Use builder pattern with phantom types for compile-time type safety
- Implement `WorkflowBuilder<I, O>` with methods returning `Self`
- Use `async_trait` for async methods
- Macro for ergonomic workflow definition (similar to `#[derive]`)

### 4. **Worker Runtime**

**File:** `src/v1/client/worker/worker.ts`

**Key Patterns:**
- **Dual Workers**: Separate workers for regular and durable tasks
- **Action Listener**: gRPC streaming for receiving work
- **Graceful Shutdown**: Handles `SIGTERM`/`SIGINT` signals
- **Label-Based Routing**: Worker affinity via labels

**Worker Structure:**
```typescript
class Worker {
  nonDurable: V1Worker;     // Regular tasks
  durable?: V1Worker;        // Durable tasks (lazy-created)

  async registerWorkflows(workflows[])
  start(): Promise<void>
  stop(): Promise<void>
  upsertLabels(labels): Promise<void>
  pause/unpause(): Promise<void>
}

class V1Worker {
  async registerWorkflowV1(workflow)
  async start() {
    // Listen to gRPC action stream
    // Execute tasks in thread pool
    // Send results back
  }
}
```

**Rust Translation:**
- Use `tokio` for async runtime
- `tokio::select!` for graceful shutdown
- `tonic` for gRPC streaming
- Thread pool via `rayon` or `tokio::task::spawn`
- Use `Arc` for shared state between workers

### 5. **Context and Execution**

**File:** `src/step.ts`

**Key Patterns:**
- **Context Object**: Provides task metadata and operations
- **Child Workflow Spawning**: Tasks can trigger other workflows
- **Streaming**: Real-time output streaming via `putStream()`
- **Timeout Management**: Dynamic timeout refresh

**Context API:**
```typescript
class Context<I> {
  // Metadata
  workflowName(): string
  taskName(): string
  workflowRunId(): string
  taskRunId(): string
  retryCount(): number

  // Parent task outputs
  parentOutput<L>(task): Promise<L>

  // Child workflow execution
  runChild<Q, P>(workflow, input): Promise<P>
  runNoWaitChild<Q, P>(workflow, input): Promise<WorkflowRunRef<P>>
  bulkRunChildren<Q, P>(children[]): Promise<P[]>

  // Operations
  log(message, level)
  putStream(data)
  refreshTimeout(duration)
  releaseSlot()

  // Utilities
  worker: ContextWorker
  abortController: AbortController
}
```

**Rust Translation:**
- Use struct with lifetime parameters for context
- `async fn` for async operations
- `tokio::sync::mpsc` for streaming
- `tokio::time::timeout` for timeout management

## Communication Layer

### 6. **gRPC Client (Dispatcher)**

**File:** `src/clients/dispatcher/dispatcher-client.ts`

**Key Patterns:**
- **nice-grpc Library**: TypeScript gRPC framework
- **Middleware**: Token injection, retry logic
- **Bidirectional Streaming**: Worker-server communication
- **Retrier**: Automatic retry with exponential backoff

**Rust Translation:**
- Use `tonic` for gRPC
- Implement `tonic::service::Interceptor` for middleware
- `tonic::codegen::tokio_stream` for streaming
- Custom retry logic with `tokio::time::sleep`

### 7. **REST API Client**

**File:** `src/clients/rest/api.ts`

**Key Patterns:**
- **Code Generation**: OpenAPI → TypeScript via `swagger-typescript-api`
- **Axios**: HTTP client with interceptors
- **Bearer Auth**: Token in Authorization header

**Rust Translation:**
- Use `reqwest` for HTTP client
- Generate client from OpenAPI spec using `openapi-generator`
- Middleware for auth header injection

## Error Handling

**File:** `src/util/errors/hatchet-error.ts`

**Key Patterns:**
- **Custom Error Class**: `HatchetError` extends `Error`
- **Non-Retryable Errors**: `NonRetryableError` for permanent failures
- **Zod Validation Errors**: Config validation errors

**Rust Translation:**
- Use `thiserror` for error definitions
- Enum-based errors with context
- `anyhow` for error propagation in applications

```rust
#[derive(Debug, thiserror::Error)]
pub enum HatchetError {
    #[error("Configuration error: {0}")]
    Config(String),

    #[error("gRPC error: {0}")]
    Grpc(#[from] tonic::Status),

    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),

    #[error("Non-retryable: {0}")]
    NonRetryable(String),
}
```

## Testing Strategy

**Files:** `*.test.ts`, `*.e2e.ts`

**Key Patterns:**
- **Unit Tests**: Jest for isolated component testing
- **E2E Tests**: Full workflow execution tests
- **Test Fixtures**: Mock configurations and data
- **Async Testing**: Proper async/await handling

**Rust Translation:**
- Use `tokio::test` for async tests
- `mockito` or `wiremock` for HTTP mocking
- `tonic-mock` for gRPC mocking
- Integration tests in `tests/` directory

## Documentation Structure

**Files:** `README.md`, TypeDoc comments

**Key Patterns:**
- **JSDoc**: Inline documentation for API methods
- **TypeDoc**: Generated API documentation
- **Examples**: Inline examples in docstrings
- **Quick Start**: Minimal example in README

**Rust Translation:**
- Use `///` doc comments for public API
- `cargo doc` for documentation generation
- `examples/` directory for runnable examples
- `README.md` with quick start

## Key Architectural Decisions for Rust

### What Translates Well:

1. **Type Safety**: Rust's type system is even stronger - leverage it
2. **Builder Pattern**: Natural fit for Rust
3. **Async/Await**: `tokio` provides similar capabilities
4. **Error Handling**: Rust's `Result<T, E>` is superior
5. **Configuration System**: `serde` + `config` crate work well
6. **gRPC Communication**: `tonic` is excellent

### What Needs Adaptation:

1. **Lazy Initialization**: Use `OnceCell` or `Arc<Mutex<Option<T>>>`
2. **Class Inheritance**: Use composition with traits
3. **Dynamic Typing**: Strong typing everywhere in Rust
4. **Prototype Chain**: Not applicable - use trait implementations
5. **Method Overloading**: Use different method names or builder pattern

## Recommended Rust SDK Structure

```
hatchet-sdk/
├── src/
│   ├── client.rs           # Main HatchetClient
│   ├── config.rs           # Configuration types
│   ├── workflow/
│   │   ├── mod.rs
│   │   ├── builder.rs      # Workflow builder
│   │   ├── declaration.rs  # Workflow types
│   │   └── context.rs      # Execution context
│   ├── worker/
│   │   ├── mod.rs
│   │   ├── runtime.rs      # Worker runtime
│   │   └── executor.rs     # Task execution
│   ├── transport/
│   │   ├── grpc/          # gRPC clients
│   │   └── rest/          # REST client
│   ├── error.rs           # Error types
│   ├── util/              # Utilities
│   └── lib.rs
├── examples/              # Usage examples
├── tests/                 # Integration tests
└── Cargo.toml
```

## Critical Implementation Patterns

### 1. **Generic Workflow Types:**
```rust
pub struct Workflow<I, O>
where
    I: Serialize + DeserializeOwned,
    O: Serialize + DeserializeOwned,
{
    definition: WorkflowDefinition,
    _marker: PhantomData<(I, O)>,
}
```

### 2. **Builder Pattern:**
```rust
impl<I, O> Workflow<I, O> {
    pub fn task<F>(&mut self, name: &str, func: F) -> &mut Self
    where
        F: Fn(I, Context<I>) -> BoxFuture<'static, Result<O>>
    {
        // Add task to definition
        self
    }
}
```

### 3. **Async Context:**
```rust
pub struct Context<I> {
    metadata: Arc<TaskMetadata>,
    client: Arc<HatchetClient>,
    _marker: PhantomData<I>,
}

impl<I> Context<I> {
    pub async fn run_child<Q, P>(
        &self,
        workflow: Workflow<Q, P>,
        input: Q,
    ) -> Result<P> {
        // Implementation
    }
}
```

## Conclusion

The TypeScript SDK provides an excellent blueprint for a type-safe, ergonomic workflow orchestration client. Translating to Rust will leverage even stronger compile-time guarantees while maintaining similar API patterns through builders, generics, and async/await.

**Key Advantages of Rust Implementation:**
- Compile-time type checking (no runtime reflection needed)
- Zero-cost abstractions
- Memory safety without garbage collection
- Superior concurrency primitives via `tokio`
- Better performance characteristics

The result will be a more robust, performant SDK that maintains the ergonomic API design of the TypeScript version while providing stronger safety guarantees.
