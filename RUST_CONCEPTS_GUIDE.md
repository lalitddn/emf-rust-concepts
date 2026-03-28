# Rust Concepts Used in EXAScaler Management Framework

This guide explains the key Rust concepts and patterns used extensively throughout the EMF codebase, with practical examples from the project.

## Table of Contents

1. [Async/Await with Tokio](#1-asyncawait-with-tokio)
2. [Parser Combinators with Winnow](#2-parser-combinators-with-winnow)
3. [Trait-Based Design](#3-trait-based-design)
4. [Error Handling](#4-error-handling)
5. [Serialization with Serde](#5-serialization-with-serde)
6. [Channels for Concurrency](#6-channels-for-concurrency)
7. [Type Safety with Newtypes](#7-type-safety-with-newtypes)
8. [State Machines](#8-state-machines)
9. [Database with SQLx](#9-database-with-sqlx)
10. [gRPC with Tonic](#10-grpc-with-tonic)

---

## 1. Async/Await with Tokio

**Usage Level:** ⭐⭐⭐ Very Heavy

### What It Is
Asynchronous programming allows concurrent execution without blocking threads. Tokio is Rust's most popular async runtime, providing:
- Non-blocking I/O operations
- Task scheduling
- Timers and timeouts
- Async networking

### Why EMF Uses It
EMF manages distributed systems with many concurrent operations:
- HTTP/gRPC requests to multiple nodes
- Database queries
- File I/O operations
- Real-time metric collection

### Example from EMF

**Simple Async Query** (`emf-influx/src/lib.rs`):
```rust
#[cfg(feature = "with-db-client")]
pub async fn query(
    influx_client: &influx_db_client::Client,
    q: &str,
) -> Result<Option<Vec<influx_db_client::Node>>, Error> {
    let resp = influx_client.query(q, None).await?;
    Ok(resp)
}
```

**Key Points:**
- `async fn` - Function returns a Future
- `.await` - Suspend execution until Future completes
- Non-blocking - Other tasks can run while waiting

**Retry with Async** (`emf-influx/src/lib.rs`):
```rust
async fn write_points_retry(
    &self,
    points: &Points<'_>,
    precision: Option<Precision>,
    rp: Option<&str>,
) -> Result<(), error::Error> {
    retry_future(
        |_, _| self.write_points(points, precision, rp),
        |k, _, e| match (k, e) {
            (0, _) => RetryAction::RetryNow,
            (k, err) if k < 20 => match err {
                influx_db_client::Error::Communication(_)
                | influx_db_client::Error::Unknow(_) => {
                    RetryAction::WaitFor(Duration::from_secs(u64::from(2 * k)))
                }
                e => RetryAction::ReturnError(e),
            },
            (_, err) => {
                tracing::debug!("Retry error: {err}");
                RetryAction::ReturnError(err)
            }
        },
    )
    .await?;

    Ok(())
}
```

**Benefits:**
- Handle failures gracefully with exponential backoff
- Non-blocking delays between retries
- Maintains high concurrency

### Best Practices in EMF
1. ✅ Always use `async` for I/O operations
2. ✅ Use Tokio runtime (never mix runtimes)
3. ✅ Prefer channels over locks
4. ❌ Never block the async runtime with synchronous operations

---

## 2. Parser Combinators with Winnow

**Usage Level:** ⭐⭐⭐ Heavy

### What It Is
Parser combinators are small, composable parsing functions that can be combined to parse complex formats. Winnow is a modern, fast parser combinator library.

### Why EMF Uses It
EMF parses many command outputs and configuration formats:
- Lustre CLI output (`lctl`, `lnetctl`)
- Network configuration
- System information
- Device naming schemes

**Coding Guideline:** "Favor parser combinators over regular expressions"

### Example from EMF

**Parsing Device Names** (`emf-device-naming-scheme/src/lib.rs`):
```rust
pub fn fs_name<Input>() -> impl Parser<Input, Output = String>
where
    Input: Stream<Token = char>,
    Input::Error: ParseError<Input::Token, Input::Range, Input::Position>,
{
    combine::count_min_max(1, 8, alpha_num())
}

pub(crate) fn hex_index<Input>() -> impl Parser<Input, Output = u16>
where
    Input: Stream<Token = char>,
    Input::Error: ParseError<Input::Token, Input::Range, Input::Position>,
{
    combine::count(4, hex_digit()).then(|x: String| match x.parse::<u16>() {
        Ok(n) => value(n).left(),
        Err(e) => unexpected_any(Format(e)).right(),
    })
}
```

**Parsing Network QoS** (`emf-network-dispatchers/src/qos.rs`):
```rust
fn parse_dscp2prio(input: &mut &str) -> ModalResult<Dscp2Prio> {
    seq! (Dscp2Prio{
        _: (multispace0, "prio:"),
        prio: digit1.parse_to::<u32>(),
        
        _: (space1, "dscp:"),
        enabled: sized_array(','),
        
        _: (',', multispace0)
    })
    .parse_next(input)
}
```

**Complex Parser for TOML Paths** (`emf-toml/src/toml_path.rs`):
```rust
use winnow::{
    Parser as _,
    ascii::{digit1, multispace0, multispace1},
    combinator::{alt, delimited, preceded, repeat, separated_foldl1},
    token::{any, none_of, take_till},
};

// Parses paths like: "host.*.nic.mgmt0"
// Can navigate nested TOML structures
```

### Benefits of Parser Combinators
- ✅ **Composable**: Small parsers combine into complex ones
- ✅ **Type-Safe**: Compile-time guarantees about parsing logic
- ✅ **Maintainable**: Easier to understand than regex
- ✅ **Testable**: Each parser can be unit tested
- ✅ **Better Errors**: Detailed error messages with context

### Example: Parsing Lustre Target Names
```rust
// Parse: "fs-OST0001" or "fs-MDT0000"
fn target_name() -> impl Parser {
    (
        fs_name(),           // "fs" (1-8 alphanumeric)
        char('-'),           // "-"
        alt((
            string("OST"),   // "OST"
            string("MDT"),   // "MDT"
        )),
        index(),             // "0001" (4 digits -> u16)
    ).map(|(fs, _, type, idx)| TargetName { fs, type, idx })
}
```

---

## 3. Trait-Based Design

**Usage Level:** ⭐⭐⭐ Very Heavy

### What It Is
Traits define shared behavior across types. They're similar to interfaces but more powerful:
- Can have default implementations
- Can be used as bounds for generics
- Enable zero-cost abstractions

### Why EMF Uses It
EMF has many abstractions for:
- Different data sources (InfluxDB, PostgreSQL)
- Command execution (local, remote, gRPC)
- Component types (hosts, filesystems, targets)

### Example from EMF

**Data Transformation Trait** (`emf-influx/src/lib.rs`):
```rust
pub trait IntoItems {
    fn into_item<T: serde::de::DeserializeOwned>(self) -> Result<T, serde_json::Error>;
}

impl IntoItems for Vec<keys::Node> {
    fn into_item<T: serde::de::DeserializeOwned>(self) -> Result<T, serde_json::Error> {
        let xs: Vec<_> = self
            .into_iter()
            .map(|x| x.series.unwrap_or_default())
            .map(|xs| {
                let xs: Vec<_> = xs
                    .into_iter()
                    .map(|x| TagColVals(x.tags, x.columns, x.values.unwrap_or_default()))
                    .map(serde_json::Value::from)
                    .collect();
                serde_json::Value::Array(xs)
            })
            .collect();

        serde_json::from_value(serde_json::Value::Array(xs))
    }
}
```

**Usage:**
```rust
let query_result: Vec<Node> = influx_client.query("SELECT * FROM metrics").await?;

// Transform to strongly-typed structs
let metrics: Vec<MetricRecord> = query_result.into_item()?;
```

**Validation Traits** (`emf-sfa-lib/src/lib.rs`):
```rust
pub trait ClassName {
    const CLASS_NAME: &'static str;

    fn class_name(&self) -> &'static str {
        Self::CLASS_NAME
    }
}

pub trait ValidateSfaInstance: ClassName {
    fn validate_instance_class_name(x: &Instance) -> Result<(), EmfSfaLibError> {
        if x.class_name != Self::CLASS_NAME {
            return Err(EmfSfaLibError::UnexpectedInstance(
                Self::CLASS_NAME,
                x.class_name.to_string(),
            ));
        }
        Ok(())
    }
}
```

**Component Traits** (`emf-lib-state-machine/src/input_document/mod.rs`):
```rust
pub trait GetComponentType {
    fn component() -> ComponentType;
}

pub trait GetActionName {
    fn action_name() -> impl Into<ActionName>;
}

pub trait GetStepPair: GetComponentType + GetActionName {
    fn step_pair() -> StepPair {
        StepPair::new(Self::component(), Self::action_name())
    }
}

// Blanket implementation for all types implementing both traits
impl<T> GetStepPair for T where T: GetComponentType + GetActionName {}
```

### Benefits
- ✅ **Polymorphism**: One interface, many implementations
- ✅ **Zero-Cost**: No runtime overhead
- ✅ **Extensible**: Easy to add new types
- ✅ **Type-Safe**: Compiler enforces contracts

---

## 4. Error Handling

**Usage Level:** ⭐⭐⭐ Very Heavy

### What It Is
Rust's error handling uses `Result<T, E>` for recoverable errors. The `thiserror` crate makes creating custom error types easy.

### Why EMF Uses It
EMF handles many error types:
- Network failures
- Database errors
- Command execution errors
- Validation errors
- Parsing errors

**Coding Guideline:** "Avoid `.unwrap()` or `.expect()` in non-test target code"

### Example from EMF

**Custom Error Type** (`emf-influx/src/lib.rs`):
```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error(transparent)]
    InfluxDbError(#[from] influx_db_client::Error),

    #[error(transparent)]
    Serde(#[from] serde_json::Error),
}
```

**Key Features:**
- `#[error(transparent)]` - Display the wrapped error
- `#[from]` - Automatic conversion with `?` operator

**Complex Error Aggregation** (`emf-services/emf-device/src/error.rs`):
```rust
#[derive(Error, Debug)]
pub enum EmfDeviceError {
    #[error(transparent)]
    EmfManagerEnv(#[from] emf_manager_env::Error),

    #[error(transparent)]
    EmfPostgres(#[from] emf_postgres::Error),

    #[error(transparent)]
    SerdeJson(#[from] serde_json::Error),

    #[error(transparent)]
    SqlxCore(#[from] sqlx::Error),

    #[error("Could not find index in target name {0}")]
    TargetIndexNotFound(String),
}
```

**Using Errors** (`emf-device-naming-scheme/src/lib.rs`):
```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("Could not parse {0} as a valid DDN lustre target name. {SECTION}")]
    ParseLustreError(String),

    #[error("Could not parse {0} as a valid DDN LV name. {SECTION}")]
    ParseLvError(String),

    #[error(transparent)]
    Host(#[from] host::Error),
}

const SECTION: &str = "Refer to EXAScaler Installation and Administration Guide, Section Naming Formats.";
```

### Error Propagation with `?`
```rust
pub async fn complex_operation() -> Result<Data, Error> {
    let client = connect_to_database().await?;  // Returns early if error
    let data = client.query("SELECT ...").await?;
    let parsed = parse_data(data)?;
    Ok(parsed)
}
```

### Best Practices in EMF
1. ✅ Use `Result` for recoverable errors
2. ✅ Create domain-specific error types
3. ✅ Use `?` for error propagation
4. ✅ Provide context in error messages
5. ❌ Never use `.unwrap()` in production code
6. ❌ Never silently ignore errors

---

## 5. Serialization with Serde

**Usage Level:** ⭐⭐⭐ Very Heavy

### What It Is
Serde is Rust's de facto serialization framework. It supports JSON, YAML, TOML, and many other formats with zero-copy deserialization.

### Why EMF Uses It
EMF exchanges data in many formats:
- JSON APIs (GraphQL, OpenAPI)
- YAML configuration files
- TOML configuration (exascaler.toml)
- Database records
- gRPC messages

### Example from EMF

**Basic Serialization** (`emf-influx/src/lib.rs`):
```rust
#[derive(Debug, Clone, serde::Deserialize, PartialEq)]
pub struct Node {
    pub statement_id: Option<u64>,
    pub series: Option<Vec<Series>>,
}

#[derive(Debug, Clone, serde::Deserialize, PartialEq)]
pub struct Series {
    pub name: Option<String>,
    pub tags: Option<serde_json::Map<String, serde_json::Value>>,
    pub columns: Vec<String>,
    pub values: Option<Vec<Vec<serde_json::Value>>>,
}
```

**Custom Serialization** (`emf-wire-types/src/graphql/bigint.rs`):
```rust
#[serde_as]
#[derive(Debug, Clone, PartialEq, Eq, serde::Deserialize, serde::Serialize)]
#[serde(transparent)]
pub struct BigInt(#[serde_as(as = "serde_with::DisplayFromStr")] pub i64);

// Serializes as string: "12345" instead of number: 12345
// This prevents precision loss in JavaScript (which uses f64)
```

**Enum Serialization** (`emf-wire-types/src/lib.rs`):
```rust
#[derive(serde::Deserialize, serde::Serialize, Clone, Debug, PartialEq, Eq)]
#[cfg_attr(feature = "postgres-interop", derive(sqlx::Type))]
#[cfg_attr(feature = "graphql", derive(juniper::GraphQLEnum))]
#[serde(rename_all = "SCREAMING_SNAKE_CASE")]
#[repr(i16)]
pub enum MessageClass {
    Normal = 0,
    Lustre = 1,
    LustreError = 2,
}
```

**Usage:**
```json
{
  "message_class": "LUSTRE_ERROR"
}
```

**Complex Validation** (`emf-exa-config/src/exascaler_configuration/nic.rs`):
```rust
#[derive(Default, Debug, Clone, PartialEq, Eq, Serialize, Deserialize, Validate)]
#[serde(transparent)]
#[validatron(function = "valid_nics::errors")]
pub struct Nics(#[validatron] pub IndexMap<String, Nic>);
```

### Features Used in EMF
- **Derive macros**: `#[derive(Serialize, Deserialize)]`
- **Attributes**: `#[serde(rename_all = "...")]`
- **Transparent**: `#[serde(transparent)]` for wrapper types
- **Conditional**: `#[cfg_attr(...)]` for feature-gated derives
- **Custom logic**: `serde_with` for advanced scenarios

### Benefits
- ✅ **Type-Safe**: Compile-time guarantees
- ✅ **Zero-Copy**: Efficient deserialization
- ✅ **Format-Agnostic**: Same types for JSON, YAML, TOML
- ✅ **Extensible**: Custom serialization logic

---

## 6. Channels for Concurrency

**Usage Level:** ⭐⭐ Heavy

### What It Is
Channels provide message-passing between async tasks. Tokio provides several channel types:
- `mpsc`: Multi-producer, single-consumer
- `broadcast`: Multi-producer, multi-consumer
- `oneshot`: Single-value channel
- `watch`: Single-producer, multi-consumer with latest value

### Why EMF Uses It
**Coding Guideline:** "Use channels over locks (`Mutex`, `RwLock`)"

Channels are safer and easier to reason about:
- Agent services use tx/rx queues with deduplication
- State machine orchestration
- Real-time event streaming

### Example from EMF

**Executor Communication** (`emf-lib-state-machine/src/orchestrator/executor.rs`):
```rust
pub type ExecutorTx = UnboundedSender<(u32, CommandRow, InputDocument)>;

// Submit work to executor
executor_tx.send((command_id, row, input_doc)).await?;
```

**Command Plan Communication** (`emf-lib-state-machine/src/orchestrator/command_plan.rs`):
```rust
use tokio::sync::mpsc;

let (tx, mut rx) = mpsc::unbounded_channel();

// Producer task
tokio::spawn(async move {
    tx.send(data).unwrap();
});

// Consumer task
while let Some(msg) = rx.recv().await {
    process(msg).await;
}
```

**RPC Communication** (`emf-node-manager/src/utils/rpc.rs`):
```rust
let (tx, mut rx) = mpsc::unbounded_channel();

tokio::spawn(async move {
    let r = cmd.wait().await?;
    _ = tx.send(Ok(resp));
    Ok(())
});

// Stream results
Ok(Response::new(s))
```

### Channel Types in EMF

**Unbounded Channel** (unlimited capacity):
```rust
let (tx, mut rx) = mpsc::unbounded_channel();
tx.send(value)?;  // Never blocks
```

**Bounded Channel** (back-pressure):
```rust
let (tx, mut rx) = mpsc::channel(100);  // Max 100 messages
tx.send(value).await?;  // Waits if full
```

### Benefits
- ✅ **No Deadlocks**: Unlike mutexes
- ✅ **Composable**: Easy to create pipelines
- ✅ **Back-pressure**: Bounded channels prevent memory issues
- ✅ **Clear Ownership**: Message passing makes ownership explicit

---

## 7. Type Safety with Newtypes

**Usage Level:** ⭐⭐ Heavy

### What It Is
The Newtype pattern wraps a primitive type in a struct to add:
- Type safety (e.g., can't mix up UserId and HostId)
- Validation logic
- Domain-specific methods

### Why EMF Uses It
**Coding Guideline:** "Use newtypes for fields needing input validation"

EMF has many domain-specific types:
- Target names (MDT0001, OST0042)
- Filesystem names (must be 1-8 alphanumeric)
- IP addresses with validation
- BigInt for GraphQL (avoids JavaScript precision loss)

### Example from EMF

**GraphQL Duration** (`emf-wire-types/src/graphql/duration.rs`):
```rust
#[derive(serde::Deserialize, serde::Serialize, Clone, PartialEq, Eq, Debug)]
#[serde(try_from = "String", into = "String")]
#[cfg_attr(feature = "graphql", derive(juniper::GraphQLScalar))]
pub struct GraphQlDuration(pub Duration);

impl TryFrom<String> for GraphQlDuration {
    type Error = String;

    fn try_from(s: String) -> Result<Self, Self::Error> {
        humantime::parse_duration(&s)
            .map(GraphQlDuration)
            .map_err(|e| e.to_string())
    }
}

impl From<GraphQlDuration> for String {
    fn from(d: GraphQlDuration) -> String {
        humantime::format_duration(d.0).to_string()
    }
}
```

**Usage:**
```rust
// Input: "5m" or "2h30m"
// Validated at deserialization time
let duration: GraphQlDuration = serde_json::from_str(r#""5m""#)?;
```

**BigInt for JavaScript Compatibility** (`emf-wire-types/src/graphql/bigint.rs`):
```rust
#[serde_as]
#[derive(Debug, Clone, PartialEq, Eq)]
#[serde(transparent)]
pub struct BigInt(#[serde_as(as = "serde_with::DisplayFromStr")] pub i64);

// Prevents precision loss in JavaScript
// JSON: "12345678901234567" instead of 12345678901234567
```

**Filesystem Name Validation** (`emf-device-naming-scheme/src/lib.rs`):
```rust
pub fn fs_name<Input>() -> impl Parser<Input, Output = String>
where
    Input: Stream<Token = char>,
{
    // Must be 1-8 alphanumeric characters
    combine::count_min_max(1, 8, alpha_num())
}
```

### Benefits
- ✅ **Type Safety**: Can't accidentally mix types
- ✅ **Validation**: Enforce invariants at compile/runtime
- ✅ **Self-Documenting**: Code is clearer
- ✅ **Refactoring**: Easy to change underlying type

### Example: Preventing Type Confusion
```rust
// BAD: Easy to mix up
fn transfer(from: u64, to: u64, amount: u64) { }

// GOOD: Compiler prevents mistakes
struct UserId(u64);
struct AccountId(u64);
struct Amount(u64);

fn transfer(from: UserId, to: UserId, amount: Amount) { }

// This won't compile:
transfer(account_id, user_id, amount);  // ❌ Type mismatch
```

---

## 8. State Machines

**Usage Level:** ⭐⭐ Medium

### What It Is
State machines model systems that transition between discrete states. EMF uses async state machines for orchestration.

### Why EMF Uses It
The core orchestration engine converts declarative YAML into executable state machines:
- Input Document → Dependency Graph → Async State Machine
- Maximum parallelism while respecting dependencies
- Handles retries, failures, and cancellation

### Architecture

**Input Document** (YAML):
```yaml
jobs:
  - id: configure_filesystem
    steps:
      - action: filesystem.create
        id: create_fs
        inputs:
          name: lustre01

      - action: filesystem.mount
        id: mount_fs
        needs: [create_fs]
        inputs:
          name: lustre01
```

**Transformation Flow:**
```
YAML Input → InputDocument → Dependency Graph → Async Futures → Execution
```

### Example from EMF

**State Definition** (`emf-lib-state-machine/src/lib.rs`):
```rust
#[derive(Clone, Debug, serde::Serialize, serde::Deserialize, PartialEq)]
pub enum State {
    Pending,
    Running,
    Complete,
    Failed,
    Cancelled,
    Skipped,
}
```

**Orchestrator Execution** (`emf-lib-state-machine/src/orchestrator/mod.rs`):
```rust
pub async fn run_state_machine(
    input_doc: InputDocument,
    backend: impl Backend,
) -> Result<State, Error> {
    // Build dependency graph
    let graph = build_job_graphs(&input_doc)?;

    // Convert to async futures
    let futures = graph.into_futures();

    // Execute with maximum parallelism
    let state = execute_futures(futures).await?;

    Ok(state)
}
```

**Action Runner** (`emf-lib-state-machine/src/orchestrator/action_runner.rs`):
```rust
pub async fn invoke(
    pg_pool: PgPool,
    mut stdout_writer: OutputWriter,
    mut stderr_writer: OutputWriter,
    local_cancelation_tx: LocalCancelationTx,
    input: &Input,
    outputs: Outputs,
    config: emf_exa_config::subscriber::Config,
) -> Result<Option<serde_json::Value>, Error> {
    match input {
        Input::Host(x) => match x {
            host::Input::BashRpcCommand(x) => {
                let policy = exponential_backoff_policy_builder().build();

                let client = retry_future(
                    |_, _| RpcClient::connect(&x.host, &fqdn, ClientCertType::Emf),
                    policy,
                )
                .await?;

                let mut child = client
                    .command(&run)
                    .spawn();

                // Execute and handle state transitions
                child.wait().await
            }
            // ... more actions
        }
        // ... more input types
    }
}
```

### Benefits
- ✅ **Declarative**: Specify what, not how
- ✅ **Parallel**: Maximum concurrency automatically
- ✅ **Resilient**: Built-in retry and error handling
- ✅ **Observable**: Track state transitions

---

## 9. Database with SQLx

**Usage Level:** ⭐⭐ Medium

### What It Is
SQLx is an async SQL toolkit with compile-time checked queries. The compiler verifies:
- SQL syntax
- Column names and types
- Return type compatibility

### Why EMF Uses It
EMF stores configuration and state in PostgreSQL:
- Cluster configuration
- Command history
- Real-time updates via LISTEN/NOTIFY
- Migrations with rollback support

### Example from EMF

**Compile-Time Checked Query** (from EMF codebase):
```rust
// The query! macro checks this at compile time
let targets = sqlx::query!(
    r#"
    SELECT
        id,
        name,
        state as "state: TargetState",
        host_id,
        filesystem_id
    FROM target
    WHERE filesystem_id = $1
    "#,
    filesystem_id
)
.fetch_all(&pool)
.await?;

// Compiler knows:
// - targets[0].id is i64
// - targets[0].state is TargetState enum
// - targets[0].name is String
```

**Migrations** (`migrations/20200728203410_devices.sql`):
```sql
-- migrations/20241201000000_add_feature.up.sql
CREATE TABLE IF NOT EXISTS new_feature (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- migrations/20241201000000_add_feature.down.sql
DROP TABLE IF EXISTS new_feature;
```

**Running Migrations:**
```bash
# Apply migrations
sqlx migrate run

# Revert last migration
sqlx migrate revert
```

**Database Types** (`emf-wire-types/src/lib.rs`):
```rust
#[derive(serde::Deserialize, serde::Serialize, Clone, Debug)]
#[cfg_attr(feature = "postgres-interop", derive(sqlx::Type))]
pub enum MessageClass {
    Normal = 0,
    Lustre = 1,
    LustreError = 2,
}
```

### Real-Time Updates with LISTEN/NOTIFY

**Publisher:**
```sql
-- Trigger on table changes
CREATE TRIGGER target_notify
AFTER INSERT OR UPDATE OR DELETE ON target
FOR EACH ROW EXECUTE FUNCTION notify_change();
```

**Subscriber:**
```rust
let mut listener = PgListener::connect(&database_url).await?;
listener.listen("target_changes").await?;

while let Some(notification) = listener.try_recv().await? {
    let payload: TargetChange = serde_json::from_str(notification.payload())?;
    handle_change(payload).await?;
}
```

### Benefits
- ✅ **Compile-Time Safety**: Catch SQL errors before runtime
- ✅ **Type-Safe**: No manual string parsing
- ✅ **Async**: Non-blocking database access
- ✅ **Migrations**: Versioned schema changes with rollback

---

## 10. gRPC with Tonic

**Usage Level:** ⭐⭐ Medium

### What It Is
gRPC is a high-performance RPC framework using Protocol Buffers. Tonic is Rust's async gRPC implementation.

### Why EMF Uses It
V2 architecture uses gRPC for:
- Manager ↔ Node bidirectional communication
- Streaming command output
- Real-time metrics collection
- Type-safe API contracts

### Example from EMF

**Protocol Definition** (`proto/rpc/v1/rpc.proto`):
```protobuf
syntax = "proto3";

package emf.rpc.v1;

service Rpc {
  // Execute command and stream output
  rpc RequestRpc(RequestRpc) returns (stream ResponseStream);

  // Async command execution
  rpc RequestAsyncRpc(RequestAsync) returns (RequestId);

  // Check async command status
  rpc RequestAsyncStatus(RequestId) returns (ResponseAsync);
}

message RequestRpc {
  string command = 1;
  uint64 timeout_seconds = 2;
}

message ResponseStream {
  oneof message {
    bytes stdout = 1;
    bytes stderr = 2;
    int32 code = 3;
  }
}
```

**Generated Code:**
```bash
# Tonic generates Rust code from .proto files
cargo build  # Runs build.rs which invokes tonic-build
```

**Server Implementation** (`emf-node-manager/src/utils/rpc.rs`):
```rust
use tonic::{Request, Response, Status};

pub struct RpcService;

#[tonic::async_trait]
impl Rpc for RpcService {
    type RequestRpcStream = ResponseStream;

    async fn request_rpc(
        &self,
        request: Request<RequestRpc>,
    ) -> Result<Response<Self::RequestRpcStream>, Status> {
        let req = request.into_inner();
        let command = req.command;

        // Execute command
        let mut child = build_cmd(&command).spawn()?;

        let (tx, rx) = mpsc::unbounded_channel();

        // Stream stdout
        tokio::spawn(async move {
            let mut stdout = BufReader::new(child.stdout.take().unwrap());
            let mut buf = vec![0; 4096];

            loop {
                match stdout.read(&mut buf).await {
                    Ok(0) => break,
                    Ok(n) => {
                        let resp = ResponseStream {
                            message: Some(response_stream::Message::Stdout(
                                buf[..n].to_vec()
                            )),
                        };
                        _ = tx.send(Ok(resp));
                    }
                    Err(e) => break,
                }
            }
        });

        Ok(Response::new(ReceiverStream::new(rx)))
    }

    async fn request_async_rpc(
        &self,
        request: Request<RequestAsync>,
    ) -> Result<Response<RequestId>, Status> {
        let uuid = Uuid::new_v4().to_string();
        let command = request.into_inner().command;

        let child = build_cmd(&command).spawn()?;

        // Store child process for later status checks
        ASYNC_PROCESSES.lock().await.insert(uuid.clone(), child);

        Ok(Response::new(RequestId { id: uuid }))
    }
}
```

**Client Usage** (`emf-manager-client/src/node_manager.rs`):
```rust
pub struct NodeManagerGrpcClient {
    client: RpcClient,
}

impl NodeManagerGrpcClient {
    pub async fn connect(host: &str, port: u16) -> Result<Self, Error> {
        let endpoint = format!("https://{host}:{port}");
        let client = RpcClient::connect(endpoint).await?;
        Ok(Self { client })
    }

    pub async fn execute_command(
        &mut self,
        command: &str,
    ) -> Result<CommandOutput, Error> {
        let request = RequestRpc {
            command: command.to_string(),
            timeout_seconds: 300,
        };

        let mut stream = self.client
            .request_rpc(request)
            .await?
            .into_inner();

        let mut stdout = Vec::new();
        let mut stderr = Vec::new();
        let mut exit_code = None;

        while let Some(resp) = stream.message().await? {
            match resp.message {
                Some(Message::Stdout(data)) => stdout.extend(data),
                Some(Message::Stderr(data)) => stderr.extend(data),
                Some(Message::Code(code)) => exit_code = Some(code),
                None => {}
            }
        }

        Ok(CommandOutput {
            stdout,
            stderr,
            exit_code: exit_code.unwrap_or(-1),
        })
    }
}
```

### Benefits
- ✅ **Type-Safe**: Protocol Buffers enforce contracts
- ✅ **Efficient**: Binary protocol, faster than JSON
- ✅ **Streaming**: Bidirectional streams supported
- ✅ **Cross-Language**: Can interop with other languages
- ✅ **HTTP/2**: Multiplexing, flow control built-in

---

## Best Practices Summary

### EMF Coding Guidelines

#### ✅ DO:
1. **Always use async** for I/O operations
2. **Use Tokio** as the async runtime (never mix runtimes)
3. **Use channels** instead of locks (`Mutex`, `RwLock`)
4. **Use parser combinators** (`winnow`) instead of regex
5. **Create custom error types** with `thiserror`
6. **Use newtypes** for domain-specific validation
7. **Prefer early returns** over deep nesting
8. **Use traits** to model shared behavior
9. **Separate logic into libraries** (binaries should be thin)

#### ❌ DON'T:
1. **Never use `.unwrap()` or `.expect()`** in production code
2. **Never use indexing** operations (`arr[i]`) without bounds checking
3. **Never block** the async runtime with sync operations
4. **Never use regular expressions** for parsing (use winnow)
5. **Never introduce new dependencies** with same functionality as existing ones

### Architecture Principles

1. **Libraries over binaries**: Logic lives in libs, binaries are thin wrappers
2. **Zero-cost abstractions**: Use traits and generics for polymorphism
3. **Type-driven design**: Let the compiler prevent mistakes
4. **Fail fast**: Use `Result` and propagate errors with `?`
5. **Observable systems**: Use tracing, metrics, and structured logging

---

## Learning Resources

### Official Documentation
- [Tokio Tutorial](https://tokio.rs/tokio/tutorial)
- [Winnow Parser Guide](https://docs.rs/winnow/latest/winnow/)
- [Serde Guide](https://serde.rs/)
- [SQLx Documentation](https://docs.rs/sqlx/latest/sqlx/)
- [Tonic Guide](https://github.com/hyperium/tonic)

### Books
- [Rust Async Book](https://rust-lang.github.io/async-book/)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)

### EMF-Specific
- See `CLAUDE.md` in this repository for EMF coding standards
- Check `emf-lib-state-machine/doc/overview.md` for orchestrator details
- Review `docs/architecture.md` for system architecture

---

## Quick Reference

| Concept | When to Use | Example Crate |
|---------|-------------|---------------|
| Async/Await | I/O operations, network calls | `tokio` |
| Parser Combinators | Parsing CLI output, configs | `winnow` |
| Traits | Shared behavior, abstractions | Standard library |
| Error Handling | Every fallible operation | `thiserror` |
| Serialization | Data exchange, configs | `serde`, `serde_json` |
| Channels | Task communication | `tokio::sync::mpsc` |
| Newtypes | Domain validation | `emf-wire-types` |
| State Machines | Complex workflows | `emf-lib-state-machine` |
| Database | Persistent storage | `sqlx` |
| gRPC | Service communication | `tonic`, `prost` |
