# Community Rust SDK Analysis

## Executive Summary

This document analyzes [@eswolinsky3241/hatchet-rust-sdk](https://github.com/eswolinsky3241/hatchet-rust-sdk) v0.2.2 to inform our official Rust SDK development for `hatchet-dev/hatchet/sdks/rust/`.

**Overall Assessment:** ~70% feature complete vs. our architecture requirements
**Code Quality:** Good foundations, production-tested, some architectural gaps
**Recommendation:** Build official SDK from scratch, adopting proven patterns from community version

---

## Feature Comparison Matrix

| Feature | Community SDK | Our Architecture | Gap Analysis |
|---------|---------------|------------------|--------------|
| **Core Client** | | | |
| HatchetClient facade | ✅ | ✅ | Match - good pattern |
| Configuration system | ⚠️ Manual env | ✅ Config crate | Missing: multi-source config |
| JWT authentication | ✅ | ✅ | Match |
| TLS/mTLS | ✅ | ✅ | Match |
| **Workflow/Task Builders** | | | |
| Type-safe generics `<I, O>` | ✅ | ✅ | Match - excellent |
| Builder pattern | ✅ derive_builder | ✅ Builder | Match - their approach cleaner |
| DAG dependencies (parents) | ✅ | ✅ | Match |
| Task retries | ✅ | ✅ | Match |
| Execution timeout | ✅ | ✅ | Match |
| Event triggers | ✅ | ✅ | Match |
| Cron scheduling | ✅ | ✅ | Match |
| Priority | ✅ | ✅ | Match |
| Rate limiting | ⚠️ Hardcoded empty | ✅ Configurable | Missing: rate_limits always `vec![]` |
| Concurrency control | ⚠️ Exists, not exposed | ✅ Full support | Missing: not configurable |
| Conditions (waitFor, etc.) | ❌ Hardcoded None | ✅ Full support | Missing: critical feature |
| **Worker Runtime** | | | |
| Worker registration | ✅ | ✅ | Match |
| Task execution | ✅ | ✅ | Match |
| Worker labels | ✅ | ✅ | Match |
| Graceful shutdown | ✅ | ✅ | Match |
| Dual workers (durable/non) | ❌ | ✅ | Missing: single worker only |
| **Execution Context** | | | |
| Workflow metadata | ✅ | ✅ | Match |
| Parent outputs | ✅ | ✅ | Match |
| Logging | ✅ ctx.log() | ✅ putStream | Partial: no streaming |
| Child workflow spawning | ✅ | ✅ | Match - excellent example |
| Streaming (putStream) | ❌ | ✅ | Missing: no real-time streaming |
| **Durable Tasks** | | | |
| DurableContext | ❌ | ✅ | Missing: entire feature |
| Sleep/resume | ❌ | ✅ | Missing |
| Durable worker | ❌ | ✅ | Missing |
| **Protocol Clients** | | | |
| gRPC Dispatcher | ✅ | ✅ | Match |
| gRPC Admin | ✅ | ✅ | Match |
| gRPC Events | ✅ | ✅ | Match |
| gRPC Workflows | ✅ | ✅ | Match |
| REST API (full) | ✅ 19 modules | ✅ | Match - comprehensive |
| Runs client | ✅ | ✅ | Match |
| Metrics client | ❌ | ✅ | Missing |
| **Resilience** | | | |
| Retry logic | ⚠️ Basic | ✅ Tower middleware | Missing: no exponential backoff |
| Circuit breaker | ❌ | ✅ Tower | Missing |
| Timeout management | ✅ | ✅ | Match |
| **Observability** | | | |
| Logging | ✅ log crate | ✅ tracing | Different: log vs tracing |
| Structured logging | ❌ | ✅ | Missing: no span/fields |
| Metrics | ❌ | ✅ | Missing |
| **Testing** | | | |
| Unit tests | ⚠️ Some | ✅ 80%+ target | Partial |
| Integration tests | ✅ | ✅ | Match |
| Examples | ✅ 5 examples | ✅ 17+ planned | Good start |

---

## Architecture Patterns Analysis

### ✅ **Patterns to Adopt**

#### 1. ExecutableTask Trait with Type Erasure
**Location:** `src/runnables/task.rs:50-56`

```rust
pub trait ExecutableTask: Send + Sync + dyn_clone::DynClone {
    fn execute(&self, input: serde_json::Value, ctx: Context) -> TaskResult;
    fn name(&self) -> &str;
}

dyn_clone::clone_trait_object!(ExecutableTask);
```

**Why Adopt:**
- Allows storing heterogeneous tasks (different `<I, O>` types) in same collection
- Uses `dyn_clone` crate for trait object cloning
- Clean abstraction for worker execution loop

**Our Implementation:**
- Use this exact pattern for `WorkerRuntime::register_task()`
- Add to architecture as "Type Erasure Pattern" section

#### 2. derive_builder Pattern
**Location:** `src/runnables/task.rs:59-87`

```rust
#[derive(Clone, derive_builder::Builder)]
#[builder(pattern = "owned")]
pub struct Task<I, O> {
    client: Hatchet,
    pub(crate) name: String,
    handler: Arc<dyn Fn(I, Context) -> Pin<Box<...>>>,
    #[builder(default = vec![])]
    parents: Vec<String>,
    // ... more fields with defaults
}
```

**Why Adopt:**
- Cleaner than manual builder implementation
- Compile-time validation of required fields
- Default values via attributes

**Our Implementation:**
- Add `derive_builder = "0.20"` to dependencies
- Use for TaskBuilder, WorkflowBuilder

#### 3. OpenAPI-Generated REST Client
**Location:** `src/clients/rest/apis/*` (19 modules)

**Why Adopt:**
- Comprehensive API coverage (all endpoints)
- Auto-generated from OpenAPI spec
- Type-safe models

**Our Implementation:**
- Use same OpenAPI generator approach
- Ensure we have latest spec from server

### ⚠️ **Patterns to Improve**

#### 1. Configuration System
**Current:** Manual environment variable parsing
**Better:** Use `config` crate for multi-source configuration

```rust
// Theirs: Manual
let token = std::env::var("HATCHET_CLIENT_TOKEN")?;

// Ours: config crate
let config = Config::builder()
    .add_source(config::Environment::with_prefix("HATCHET"))
    .add_source(config::File::with_name(".hatchet/config"))
    .build()?;
```

#### 2. Logging Strategy
**Current:** `log` crate (simple)
**Better:** `tracing` crate (structured, async-aware)

```rust
// Theirs: log
log::info!("Starting task {}", task_name);

// Ours: tracing
tracing::info!(
    task_name = %task_name,
    workflow_id = %workflow_id,
    "Starting task"
);
```

**Benefits:**
- Structured fields for better querying
- Async-aware (correct context propagation)
- Span-based tracing (task duration, parent-child relationships)

#### 3. Resilience Patterns
**Current:** Basic retry, no circuit breaker
**Better:** Tower middleware for comprehensive resilience

**Missing:**
- Exponential backoff
- Circuit breaker for gRPC connection
- Rate limit detection (RESOURCE_EXHAUSTED handling)

---

## Critical Missing Features

### 1. Durable Tasks (HIGH PRIORITY)
**Impact:** Cannot implement long-running workflows with sleep/resume

**What's Needed:**
```rust
pub trait DurableContext: Context {
    async fn sleep_for(&self, duration: Duration) -> Result<()>;
    fn step_number(&self) -> u32;
}
```

**Implementation Complexity:** Medium (2 weeks)

### 2. Streaming Support (MEDIUM PRIORITY)
**Impact:** No real-time output streaming

**What's Needed:**
```rust
impl Context {
    pub async fn put_stream(&self, data: impl Serialize) -> Result<()>;
}
```

**Implementation Complexity:** Low (3 days)

### 3. Conditions (MEDIUM PRIORITY)
**Impact:** Cannot implement waitFor, cancelIf, skipIf patterns

**What's Needed:**
```rust
pub struct TaskBuilder<I, O> {
    pub fn wait_for(&mut self, condition: Condition) -> &mut Self;
    pub fn cancel_if(&mut self, condition: Condition) -> &mut Self;
    pub fn skip_if(&mut self, condition: Condition) -> &mut Self;
}
```

**Implementation Complexity:** Medium (1 week)

### 4. Rate Limiting Configuration (LOW PRIORITY)
**Impact:** Cannot configure rate limits (hardcoded empty)

**Current Code:**
```rust
// src/runnables/task.rs:131
rate_limits: vec![], // Always empty!
```

**What's Needed:**
```rust
pub struct TaskBuilder<I, O> {
    pub fn rate_limit(&mut self, limit: RateLimitConfig) -> &mut Self;
}
```

**Implementation Complexity:** Low (3 days)

---

## Dependency Comparison

### Community SDK (Cargo.toml)
```toml
[dependencies]
tonic = "0.13"           # Ours: 0.14
prost = "0.13"           # Ours: 0.14
tokio = "1.46"           # Ours: 1.47
reqwest = "0.12"         # Match ✅
serde = "1"              # Match ✅
thiserror = "2"          # Match ✅
anyhow = "1.0"           # Match ✅
derive_builder = "0.20"  # We don't have (should add)
dyn-clone = "1.0"        # We don't have (should add)
# Missing: config, jsonwebtoken, tower, tracing
```

### Our Architecture
```toml
[dependencies]
tonic = "0.14"
prost = "0.14"
tokio = { version = "1.47", features = ["full"] }
reqwest = "0.12"
serde = "1.0"
thiserror = "2.0"
anyhow = "1.0"
config = "0.14"          # They don't have
jsonwebtoken = "9"       # They don't have
tower = "0.5"            # They don't have (resilience)
tracing = "0.1"          # They don't have (structured logging)
tracing-subscriber = "0.3"
```

**Recommendation:** Merge dependency lists:
- Add `derive_builder` and `dyn-clone` from theirs
- Keep our additions (config, tower, tracing)

---

## Code Quality Assessment

### Strengths ✅
1. **Type Safety:** Excellent use of generics throughout
2. **API Coverage:** Comprehensive REST client (19 modules)
3. **Real-world Tested:** 3 weeks in production use
4. **Clean Patterns:** ExecutableTask trait is clever
5. **Good Examples:** 5 working examples covering key patterns

### Weaknesses ⚠️
1. **Test Coverage:** Limited unit tests
2. **Resilience:** No circuit breaker, basic retry
3. **Observability:** Basic logging, no structured tracing
4. **Missing Features:** Durable tasks, streaming, conditions
5. **Configuration:** Manual env parsing, no multi-source

### Security 🔒
- ✅ Uses rustls (no OpenSSL dependency)
- ✅ JWT tokens handled properly
- ⚠️ No token redaction in logs
- ⚠️ No audit trail documentation

---

## Learning Opportunities

### From Their Examples

#### 1. Child Workflow Spawning
**File:** `examples/dynamic_child_spawning.rs`

**Key Pattern:**
```rust
// They spawn N child workflows in parallel
let mut child_tasks = vec![];
for i in 0..input.n {
    let workflow_clone = child_workflow.clone();
    let handle = async move {
        workflow_clone.run(&input, None).await
    };
    child_tasks.push(handle);
}
let results = futures::future::join_all(child_tasks).await;
```

**Lesson:** Use `futures::join_all` for parallel child execution
**Adopt:** Yes - this pattern works well

#### 2. Error Handling
**File:** `examples/error.rs`

**Key Pattern:**
```rust
// Custom error type with Display for user-friendly messages
#[derive(Debug)]
pub enum TaskError {
    InputDeserialization(serde_json::Error),
    OutputSerialization(serde_json::Error),
    Execution(anyhow::Error),
}
```

**Lesson:** Separate deserialization errors from execution errors
**Adopt:** Yes - this is good error granularity

### From Their Architecture

#### 1. Worker Task Dispatching
**File:** `src/worker/task_dispatcher.rs`

**Pattern:** Separate task dispatcher from worker
**Lesson:** Clean separation between listening and execution
**Adopt:** Yes - aligns with our WorkerRuntime design

#### 2. Action Listener
**File:** `src/worker/action_listener.rs`

**Pattern:** Dedicated struct for gRPC streaming
**Lesson:** Isolate streaming logic from main worker
**Adopt:** Yes - good modularity

---

## Recommendation: Build Fresh

### Why Not Fork?

1. **Missing 30% of features** - Durable tasks, streaming, conditions
2. **Architectural gaps** - No resilience patterns, basic logging
3. **Upstream contribution** - Easier to contribute clean implementation to official repo
4. **Complete feature parity** - Need all Go/Python/TypeScript features

### What to Adopt from Community SDK

1. ✅ **ExecutableTask trait pattern** - Exact implementation
2. ✅ **derive_builder** - Add to our dependencies
3. ✅ **dyn-clone for trait objects** - Add to our dependencies
4. ✅ **OpenAPI-generated REST clients** - Same approach
5. ✅ **Child workflow spawning pattern** - From their examples
6. ✅ **Error type separation** - Deserialization vs execution

### What to Build Fresh

1. **DurableContext + durable worker** - Not in community version
2. **Streaming support** - Not in community version
3. **Condition system** - Not in community version
4. **Tower middleware** - Resilience patterns
5. **Tracing** - Structured logging
6. **Config crate** - Multi-source configuration

---

## Timeline Estimate

### If Building Fresh (Recommended)
- **Weeks 1-2:** Core client + config (adopt ExecutableTask pattern)
- **Weeks 3-4:** Workflow/Task builders + Worker runtime
- **Weeks 5-6:** Durable tasks + streaming + conditions
- **Weeks 7-8:** Metrics, resilience, polish

**Total:** 6-8 weeks to feature-complete official SDK

### If Forking Community SDK
- **Weeks 1-2:** Add durable tasks
- **Weeks 3-4:** Add streaming, conditions, rate limits
- **Weeks 5:** Refactor logging, add resilience
- **Week 6:** Polish and testing

**Total:** 4-6 weeks, but harder to upstream

---

## Conclusion

The community Rust SDK is a **solid foundation** with ~70% feature coverage. However, for an **official SDK in the upstream repository**, we recommend:

1. **Build fresh** following our architecture
2. **Adopt proven patterns** from community version (ExecutableTask, derive_builder)
3. **Implement missing 30%** from day one (durable tasks, streaming, conditions)
4. **Add architectural improvements** (resilience, structured logging)

**Result:** Complete, official Rust SDK with feature parity to Go/Python/TypeScript SDKs, ready for `hatchet-dev/hatchet/sdks/rust/` contribution.

---

**Analysis Date:** October 1, 2025
**Analyzed Version:** v0.2.2
**Analyst:** Winston (Architect Agent)
