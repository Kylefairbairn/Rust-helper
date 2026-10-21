# Kyle’s Rust Relearning Guide

## Purpose

This is a practical Rust refresh for an experienced software engineer\. The aim is not to memorize syntax\. Build small programs, predict what the compiler and runtime will do, deliberately trigger failures, then explain what Rust is protecting and how the design should change\.

**Constraint:** use only `std` and the Rust toolchain &#40;`rustc`, `cargo`, `rustfmt`, and `rustdoc`&#41;\. Do not add crates\. The examples use `cargo` for building and testing, but every program uses only the standard library\.

**Pacing:** treat each numbered session as one sitting of roughly 45–90 minutes\. Do one or two sessions per week, repeat a session when the compiler behavior is not yet predictable, and keep the experiments in separate files or Git commits so broken examples remain easy to revisit\.

## The learning loop

For every feature, follow this sequence:

1. **Predict:** write down what you think will compile, what value will print, or what test will fail\.
2. **Build the smallest example:** one concept at a time; avoid fixing unrelated warnings\.
3. **Break it on purpose:** make one edit from the experiment list below\.
4. **Read the whole diagnostic:** identify the conflicting access, lifetime, or ownership transfer\.
5. **Fix the design:** prefer a narrower borrow, owned value, or explicit shared owner over adding `.clone()` automatically\.
6. **Explain it back:** write a few sentences in `NOTES.md`: what happened, why, and what change fixed it\.
7. **Commit:** one concept per commit, including useful compiler errors in notes rather than leaving the branch uncompilable\.

Keep intentional failures in `experiments/fail/` and working versions in `src/` &#40;or separate tiny Cargo packages&#41;\. For an expected compile failure, run `cargo check` and record the key diagnostic; don’t make the normal project fail to build\.

## Quick review: ownership, borrowing, and lifetimes

### Ownership

Each Rust value has one owner\. When the owner goes out of scope, the value is dropped\. Assigning or passing a non\-`Copy` value usually **moves** ownership; the old binding can no longer be used\. This gives deterministic cleanup without a garbage collector and makes resource ownership visible in the program\.

```rust
let first = String::from("hello");
let second = first;        // ownership moves to second
println!("{second}");
// println!("{first}");   // uncomment: use after move
```

`Copy` values such as integers and booleans are copied by assignment\. `String` is not `Copy`: it owns a heap buffer, and duplicating its bits would create two owners that might both free the same allocation\. Use `clone()` when you truly want a deep copy; use borrowing when the callee only needs temporary access\.

### Borrowing

References let code use a value without taking ownership\. At a given time, Rust permits either many shared references &#40;`&T`&#41; or one exclusive mutable reference &#40;`&mut T`&#41; to a value\. A mutable borrow means the borrower has exclusive access for that borrow’s lifetime\. This prevents data races and invalidation from mutation while another reference is being used\.

```rust
fn byte_count(text: &String) -> usize { text.len() }
fn add_exclamation(text: &mut String) { text.push('!'); }
```

Prefer `&str` for read\-only text parameters when you don’t need `String`\-specific ownership\. Prefer `&mut T` only when the function must mutate the caller’s value\. A borrow does not extend the owned value’s life; it is valid only while the owner remains alive and the borrowing rules are respected\.

### Lifetimes

A lifetime is a relationship between references: it tells the compiler how long references must remain valid relative to one another\. It does **not** keep data alive, and most local lifetimes are inferred\. Explicit lifetime syntax is most often needed when a function returns a reference borrowed from an input, or a struct stores references\.

```rust
fn longer<'a>(left: &'a str, right: &'a str) -> &'a str {
    if left.len() >= right.len() { left } else { right }
}
```

The signature says the returned slice is valid for a duration shared by both inputs\. It does not promise that either input lives forever\. If the function instead creates a local `String` and tries to return `&local`, compilation fails because the local owner is dropped at return\.

### Move is not a synonym for heap allocation

Moves happen for values on the stack and heap\. A move transfers the value’s ownership semantics; the compiler may optimize the machine\-level operation\. `String` is a useful example because its buffer ownership is visible, but a struct containing only integers is also moved unless it implements `Copy`\.

### Why Rust is strict here

Rust’s rules make aliasing and mutation predictable\. They prevent use\-after\-free, double\-free, iterator invalidation, and unsynchronized shared mutation in safe code\. The compiler is enforcing invariants that otherwise need runtime checks, conventions, or careful manual review\.

## Suggested workspace

```text
rust-refresh/
  Cargo.toml
  README.md
  NOTES.md
  src/
    main.rs
    lib.rs
  experiments/
    fail/
    fixed/
```

Create it with `cargo new rust-refresh`, initialize Git with `git init`, and confirm the baseline with `cargo run`, `cargo test`, and `cargo fmt --check`\. Keep the package dependency\-free: check that `Cargo.toml` has no `[dependencies]` entries\. Use `std::env`, `std::fs`, `std::io`, `std::collections`, `std::net`, `std::thread`, `std::sync`, and `std::time` as the projects need them\.

## Session plan

### Session 1 — Refresh the value model

**Build:** a command\-line temperature converter with functions for parsing, conversion, and formatted output\. Start with hard\-coded values, then read one argument using `std::env::args`\.

**Review:** bindings, mutability, scalar types, `String` vs `&str`, expression returns, `Option`, and `Result`\.

**Break it:**

- Assign a `String` to another variable, then use the original\. Observe move semantics\.
- Repeat with `i32`\. Observe that it remains usable because it is `Copy`\.
- Change a function parameter from `&str` to `String`; see what the caller must now do\.
- Parse invalid input with `.parse::<f64>().unwrap()`, then replace the panic with a returned `Result` and a useful error message\.

**Why:** ownership and error values make resource transfer and failure paths explicit\. Rust favors explicit, typed failure over silently continuing with invalid data\.

**Done when:** invalid input is handled without panic, and you can explain which values move and which copy\.

### Session 2 — Structs, enums, and methods

**Build:** a small sensor reading model: `SensorReading { id, celsius, timestamp }`, an enum for sensor status, and methods to format a reading and determine whether it is out of range\. Use `std::time::SystemTime` if a timestamp is useful\.

**Break it:**

- Make fields private, then try to update one from outside the module\.
- Replace a status `String` with an enum and try to construct an unsupported status\.
- Add a method taking `&mut self`; call it while an immutable reference to the same value is still in use\.

**Why:** structs group related state; enums model a closed set of valid cases\. Privacy protects invariants at module boundaries\. Methods are functions with an explicit receiver &#40;`&self`, `&mut self`, or `self`&#41; that documents how they use the value\.

**Done when:** invalid statuses are unrepresentable and the receiver choice for each method makes sense\.

### Session 3 — Collections and borrowing

**Build:** an in\-memory sensor registry using `Vec<SensorReading>` and `HashMap<u32, SensorReading>`\. Add, list, find, and update readings\. Add unit tests in `src/lib.rs`\.

**Break it:**

- Hold a reference to a vector element, then push another element and use the reference\. Observe why vector reallocation could invalidate it\.
- Iterate immutably and try to modify the same collection inside the loop\.
- Change a function taking `&[SensorReading]` to take `Vec<SensorReading>` and note the new ownership requirement\.
- Use `.clone()` to silence a move error, then redesign the function to borrow and compare the trade\-off\.

**Why:** `Vec` may relocate its elements as it grows\. References cannot outlive or be invalidated by the collection they point into\. Borrowing lets APIs state whether they inspect, mutate, or consume a collection\.

**Done when:** your read\-only functions take slices or references, and updates are isolated to clear mutable operations\.

### Session 4 — Borrow checker and lifetime lab

**Build:** functions that find the longest line in a `&str`, return a slice of the input, and parse a line into an owned reading\. Add tests for empty input, one line, and ties\.

**Break it:**

- Return a reference to a local `String`\. Read the error and explain why returning an owned `String` works\.
- Write `fn first<'a>(x: &'a str, y: &str) -> &'a str { x }`, then return `y`\. Note why the declared relationship is too narrow\.
- Create a struct `Label<'a> { text: &'a str }`; attempt to store a reference to a temporary local string\.
- Keep a shared borrow alive, then attempt a mutation; then move the last use of the shared reference before the mutation and observe non\-lexical lifetimes\.

**Why:** the compiler must prove every reference points to a live owner\. Lifetime parameters describe those proven relationships; they do not extend storage duration\. Non\-lexical lifetimes let a borrow end after its final use rather than only at the closing brace\.

**Done when:** you can distinguish a dangling\-reference error from a conflicting\-borrow error, and know when returning an owned value is the cleanest API\.

### Session 5 — Error handling and file I/O

**Build:** a log\-file summary tool using `std::fs::read_to_string`\. Count lines, warnings, errors, and malformed records\. Return `Result<Summary, Error>` from core functions\. For a custom error, start with a simple enum and implement `Display` and `std::error::Error` using only `std`\.

**Break it:**

- Use `unwrap()` for a missing file, then propagate the error with `?`\.
- Replace a parse error with a default value; verify the test catches silent data loss\.
- Return `Result` from a helper but forget to propagate or handle it\.

**Why:** `Result` carries success or failure in the type\. `?` propagates an error while preserving the call chain; it does not mean “ignore errors\.”

**Done when:** missing files and malformed lines are reported distinctly, and core logic is unit\-tested without depending on a real file\.

### Session 6 — Ownership across threads

**Build:** a parallel log counter\. Read a file into owned lines, split work into chunks, spawn a few worker threads, and collect counts with `JoinHandle::join`\. Keep the first version simple: move each worker’s chunk into its thread and return a result\.

**Break it:**

- Spawn a closure that borrows a local `String`, then let the function return\. Observe the `'static` bound failure\.
- Add `move` to the closure\. Check which values are now transferred into it and which are still usable outside\.
- Try sharing `Rc<String>` between threads; observe why `Rc` is not `Send`\.
- Replace `Rc` with `Arc` for shared ownership, then try mutating through it\. Add `Mutex` only when shared mutation is actually required\.
- Drop a `JoinHandle` without joining, then compare the program’s behavior to explicitly joining workers\.

**Why:** a spawned thread may outlive the stack frame that created it, so its closure must own captured data or hold references guaranteed to be `'static`\. `Send` means a value can safely move to another thread; `Sync` means shared references can safely cross threads\. `Arc` provides atomic shared ownership, while `Mutex` provides synchronized mutable access\.

**Done when:** you can explain the difference between `move` closure capture, `Send`, `Sync`, `Arc`, and `Mutex` without treating them as interchangeable fixes\.

### Session 7 — Shared state and synchronization

**Build:** a bounded work queue using `std::sync::mpsc`, with one producer and multiple worker threads &#40;clone the receiver only if your chosen design allows it; the standard receiver is not cloneable, so consider a single dispatcher or `Arc<Mutex<Receiver<_>>>`&#41;\. Track completed jobs with `Arc<Mutex<usize>>`, or have workers return counts and aggregate them in the main thread\.

**Break it:**

- Attempt to update a shared integer from several threads without synchronization\.
- Hold a mutex guard while calling code that might lock the same mutex again; use a timeout or a deliberately controlled test harness to avoid hanging your main work\. Record how lock\-ordering mistakes can deadlock\.
- Forget to send the termination signal or close all senders; observe workers waiting forever\.
- Make the critical section larger than necessary, then reduce it\.

**Why:** safe shared mutation needs an explicit synchronization policy\. Message passing can reduce shared mutable state; locks are useful when shared state is the simpler design\. Lock scope and shutdown behavior are part of correctness\.

**Done when:** every worker has a clear termination path and shared data has the smallest practical lock scope\.

### Session 8 — TCP and a tiny HTTP server

**Build:** use `std::net::TcpListener` and `TcpStream` to serve one fixed HTTP response\. Read request bytes with `std::io::Read`, write a valid status line, headers, blank line, and body with `std::io::Write`\. Then add a thread per connection and a small request counter\.

**Break it:**

- Omit the blank line between headers and body\.
- Send a wrong `Content-Length` and see what a simple client does\.
- Read only once and assume the full request arrived\. Instead, read until a delimiter or a sensible size limit\.
- Handle every connection sequentially, then compare with a thread\-per\-connection version\.
- Add unbounded thread creation, then add a maximum connection limit or fixed worker pool\.

**Why:** TCP is a byte stream, not a message protocol; reads may return partial data\. HTTP framing is explicit\. Threads make blocking work concurrent, but unconstrained thread creation consumes resources\.

**Done when:** a browser or `curl` can retrieve the fixed response, and you can describe at least two limitations of this intentionally small server\. Do not use it as a production server\.

### Session 9 — Testable design and a mini thread pool

**Build:** extract the log analyzer and request parser into a library module\. Add table\-driven unit tests and a small thread pool using `std::thread`, `std::sync::mpsc`, and `std::sync::Mutex`\. Prefer a tested, minimal interface over trying to recreate a production crate\.

**Break it:**

- Cause a worker task to panic and inspect what `join()` returns\.
- Drop the pool while jobs remain queued; define and test the shutdown behavior\.
- Change one worker to process tasks slowly and observe queue growth or ordering\.
- Write a test that relies on thread timing, then remove the timing assumption by using channels/barriers or by testing a pure function\.

**Why:** concurrency tests should verify outcomes and synchronization, not hope that a scheduler produces one particular timing\. Separating pure logic from I/O and threads makes most behavior easy to test deterministically\.

**Done when:** ordinary logic has fast unit tests, and thread tests use explicit synchronization instead of arbitrary sleeps where possible\.

### Session 10 — Consolidation project

**Build:** a std\-only “device event monitor” command\-line tool\. It reads a text log, parses timestamp/severity/device/message records, summarizes counts, and optionally processes files concurrently\. Keep the first version single\-threaded; add concurrency only after tests establish the expected output\.

Suggested milestones:

1. Parse one line into an enum\-backed event\.
2. Report malformed lines with line numbers\.
3. Add summary counts and unit tests\.
4. Read a file and handle command\-line arguments\.
5. Add a parallel mode that returns owned summaries from workers\.
6. Document ownership decisions and known limitations in `README.md`\.

**Final review:** explain which APIs borrow and which consume values; identify every thread boundary and why captured data is `Send + 'static`; explain where lifetimes are inferred and where you wrote one explicitly; and justify any `clone`, `Arc`, or `Mutex` use\.

## Compiler experiment catalog

Use these small, isolated experiments whenever a concept feels abstract:

|Concept           |Deliberate change                                            |Expected lesson                                                     |
|------------------|-------------------------------------------------------------|--------------------------------------------------------------------|
|Move              |Use a `String` after assigning it to another binding         |The old binding no longer owns the value                            |
|`Copy`            |Repeat with `u32`                                            |Some small value types are copied by assignment                     |
|Immutable borrow  |Mutate a value while a reference is later used               |Shared borrows prevent conflicting mutation                         |
|Mutable borrow    |Make two simultaneous `&mut` references                      |Mutable access must be exclusive                                    |
|Borrow scope      |Move the last reference use before a mutation                |Borrow can end after its last use                                   |
|Dangling reference|Return a reference to a local variable                       |The local owner dies before the reference could be used             |
|Lifetime relation |Return either of two inputs with a signature tied only to one|The signature must describe the returned reference’s possible source|
|Thread capture    |Spawn a thread borrowing a local                             |Spawned work cannot rely on a short-lived stack borrow              |
|`Send`            |Send `Rc<T>` to a thread                                     |Non-atomic reference counting is not thread-safe                    |
|Shared mutation   |Mutate through `Arc<T>` without a lock                       |Shared ownership alone does not permit unsynchronized mutation      |
|`Result`          |Replace `?` with `unwrap()`                                  |Panic and propagation have different failure behavior               |
|TCP framing       |Treat one `read` as one complete request                     |TCP does not preserve write or message boundaries                   |

## A compact borrow\-checker checklist

When the compiler rejects code, ask these in order:

1. **Who owns this value?** Is it about to be moved or dropped?
2. **What reference is live?** Find its final use, not only where it was declared\.
3. **Is this shared or exclusive access?** `&T` permits reading; `&mut T` requires exclusivity\.
4. **Could this operation invalidate a reference?** Think vector growth, replacement, or owner drop\.
5. **Can the API borrow instead of consume?** Use `&T` or a slice for read\-only access\.
6. **Should the result be owned?** Returning a small owned value is often clearer than returning a reference\.
7. **For threads, who owns the captured data and can it cross the thread boundary?** Check `move`, `Send`, `Sync`, and the `'static` requirement separately\.

Avoid “fixing” errors with `clone()` until you know why ownership is needed in two places\. A clone can be correct, but it copies data and may hide an API or lifetime design issue\.

## Progress record

After each session, add this to `NOTES.md`:

```text
Session:
Prediction:
What compiled or failed:
What Rust was protecting:
What change fixed it:
One thing I can now explain:
Next small step:
```

At the end, run `cargo fmt --check`, `cargo test`, and `cargo clippy` only if Clippy is installed with the toolchain\. Clippy is a Rust tool, not a crate dependency\. Keep `Cargo.lock` checked in for this binary/application project even though it has no third\-party dependencies
