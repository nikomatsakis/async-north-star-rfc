- Feature Name: `normal_async`
- Start Date: 2026-01-14
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC defines "Normal Async" as a Rust project priority: making async Rust feel like a natural extension of regular Rust rather than a separate, harder dialect. The north star is **"just add async"** - developers targeting typical server and application environments should be able to write async Rust without requiring arcane knowledge, adapter crates, or a fundamentally different mental model than synchronous Rust.

The async ecosystem has been blocked for years waiting on language features. Key libraries like Tower remain on 0.x because they can't express the APIs they need. This RFC charts a path forward: a focused set of language improvements that unblock the ecosystem while laying groundwork for more advanced use cases.

This North Star is focused on async servers that run on typical server hardware. It is explicitly *not* aimed at async Rust in embedded or kernel environments. That doesn't mean we can ignore those use cases - they're important and we need to ensure that whatever work we do can scale to them. But server and application environments are the most common async use case, and some of these design challenges are large enough that we need to focus on concrete subproblems rather than trying to solve everything at once.

# Motivation
[motivation]: #motivation

## Target audience

This RFC focuses on developers building **network services for typical cloud environments**. This spans a wide range:

- Cloud infrastructure providers building control planes and data planes
- Backend services deployed to container orchestrators like Kubernetes
- Serverless functions on platforms like AWS Lambda or Azure Functions
- API servers, message processors, and data pipelines

This RFC does **not** directly target embedded development, mobile applications, or client-side code. However, we take care that the designs we propose do not preclude those use cases - they have different constraints that deserve their own focused attention.

## The goal

Building an async network service in Rust should be **just as easy** as building a typical synchronous Rust program. Developers who know Rust should be able to add `async` to their code and have their existing mental model transfer. When async requires something extra, the compiler should guide them with clear, actionable errors.

Today, that's not the experience. Async Rust is widely perceived as qualitatively harder than sync Rust - not just more complex, but a different kind of challenge entirely:

> "I feel like there's a ramp in learning and then there's a jump and then there's async over here. And so the goal is to get enough excitement about Rust to where you can jump the chasm of sadness and land on the async Rust side." (from the Rust Vision Doc interviews)

## The gaps

The [interviews done for the Async Vision Doc some years back](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo.html) identified major gaps between sync and async Rust that prevent us from meeting this bar. Although we have made progress since then, the majority of challenges that were identified persist to this day:

1. **Async functions in traits don't support dynamic dispatch.** You can write `async fn` in a trait, but you can't use `&dyn Trait` with it. This forces developers to use proc macros or restructure their code.

2. **Recursive async functions require arcane signatures.** In sync Rust, recursion just works. In async Rust, it requires `Box<dyn Future<Output = T> + Send + 'a>` and explicit lifetime annotations.

3. **Send bounds on async functions are hard to express.** When a trait has async methods or a function takes an async closure, there's no good way to say "I need the returned future to be Send."

4. **Spurious Send errors from compiler bugs.** The compiler sometimes incorrectly rejects valid async code when computing whether a future is Send, particularly with higher-ranked lifetimes. See [#149407].

5. **Static lifetime requirements force Arc everywhere.** Many async patterns require spawning tasks - whether to run blocking code off the async executor, to add timeouts, or to introduce parallelism. But spawning requires `'static` bounds, which pushes developers away from ergonomic `&`-based APIs toward `Arc<Mutex<...>>` patterns that feel un-Rusty.

[#149407]: https://github.com/rust-lang/rust/issues/149407

6. **No async drop.** Resources that need async cleanup (database connections, network sessions) cannot perform that cleanup in destructors.

To understand how these gaps affect real development, let's follow a developer encountering async Rust for the first time.

## Ferris joins Crabby Corp

Ferris has been writing Rust for about a year - comfortable with ownership, lifetimes, and traits. They've just joined Crabby Corp, which has an async Rust codebase for their backend services. Ferris's first task: add a caching layer to the database client.

### Gap 1: Async functions in traits don't support dynamic dispatch

*See also: [Alan needs async in traits]*

Ferris starts by defining a trait for the cache:

```rust
trait Cache {
    async fn get(&self, key: &str) -> Option<Vec<u8>>;
    async fn set(&self, key: &str, value: &[u8]);
}
```

The compiler accepts this - async fn in traits is stable now. Ferris implements it for their Redis-backed cache and tries to use it:

```rust
async fn fetch_user(cache: &dyn Cache, db: &Database, id: UserId) -> User {
    if let Some(bytes) = cache.get(&id.to_string()).await {
        return deserialize(&bytes);
    }
    // ...
}
```

```
error[E0038]: the trait `Cache` cannot be made into an object
  --> src/lib.rs:12:23
   |
12 | async fn fetch_user(cache: &dyn Cache, ...) -> User {
   |                       ^^^^^^^^^^ `Cache` cannot be made into an object
   |
note: for a trait to be "object safe" it needs to allow building a vtable
      for trait methods, `get` and `set` return `impl Future` which is not
      object-safe
```

Ferris is confused. In sync Rust, `&dyn Trait` just works. They search online and find they need to add `#[trait_variant::make(CacheSend: Send)]` or use the `async_trait` crate which boxes the futures. They pick `async_trait` because the examples are clearer:

```rust
#[async_trait]
trait Cache {
    async fn get(&self, key: &str) -> Option<Vec<u8>>;
    async fn set(&self, key: &str, value: &[u8]);
}
```

This works. But Ferris wonders: why do they need a proc macro for something that feels like it should just work?

### Gap 2: Recursive async functions require arcane signatures

*See also: [Alan tries processing some files]*

The caching layer needs to handle nested data structures - fetching a user might require fetching their organization, which might require fetching parent organizations. Ferris writes a recursive function:

```rust
async fn fetch_nested(&self, id: &str) -> NestedData {
    let data = self.fetch_one(id).await;
    if let Some(parent_id) = data.parent {
        let parent = self.fetch_nested(&parent_id).await;
        data.with_parent(parent)
    } else {
        data
    }
}
```

```
error[E0733]: recursion in an `async fn` requires boxing
 --> src/lib.rs:71:5
  |
71 | async fn fetch_nested(&self, id: &str) -> NestedData {
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ recursive `async fn`
  |
  = note: a recursive `async fn` must be rewritten to return a boxed `dyn Future`
```

Ferris has written recursive functions before - they just work. But async functions have an implicit return type (the generated future), and that type's size depends on the function body. For recursive functions, that creates infinite-size types.

After more searching, Ferris finds the incantation:

```rust
fn fetch_nested<'a>(&'a self, id: &'a str) -> Box<dyn Future<Output = NestedData> + Send + 'a> {
    Box::pin(async move {
        let data = self.fetch_one(id).await;
        if let Some(parent_id) = data.parent {
            let parent = self.fetch_nested(&parent_id).await;
            data.with_parent(parent)
        } else {
            data
        }
    })
}
```

It works, but look at that signature. Explicit lifetimes, `Box`, `dyn Future`, `Pin` (via `Box::pin`), `Send` bounds - all for a recursive function. In sync Rust, recursion is trivial. Here, it requires understanding implementation details of how async functions are compiled.

### Gaps 3 and 4: Send bounds and spurious errors

*See also: [Alan thinks he needs async locks]*

With caching working, Ferris needs to write a function that accepts any `Cache` implementation and spawns a background task to process it:

```rust
trait Cache {
    async fn process(&self);
}

async fn background_process(cache: &impl Cache) {
    tokio::spawn(async {
        cache.process().await
    });
}
```

```
error: future cannot be sent between threads safely
   --> src/lib.rs:8:5
    |
8   |     tokio::spawn(async {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    |
note: future is not `Send` because this value is used across an await
   --> src/lib.rs:9:9
    |
9   |         cache.process().await
    |         ^^^^^ the future returned by `process` is not guaranteed to be `Send`
```

Ferris understands the problem now: `tokio::spawn` requires a `Send` future, but the trait doesn't guarantee that `process()` returns a `Send` future. They try to add that bound:

```rust
async fn background_process<C: Cache + Send + Sync>(cache: &C)
where
    // How do I say "C::process() returns a Send future"?
```

There's no syntax for this. Ferris searches and finds various workarounds, none satisfying. The cleanest one involves passing a closure that boxes the future:

```rust
async fn background_process<C: Cache>(
    cache: &C,
    make_process: impl Fn(&C) -> BoxFuture<'_, ()>,
) {
    tokio::spawn(make_process(cache));
}
```

Callers have to write:

```rust
background_process(&my_cache, |c| Box::pin(c.process())).await;
```

This works because the caller knows the concrete type and can verify it's `Send`. But it's bizarre - why should the caller have to pass a closure just to call a method that's right there on the trait?

To make matters worse, sometimes Ferris writes code that *should* work but the compiler rejects anyway. The compiler's Send-bound analysis has known bugs, particularly with higher-ranked lifetimes, that cause spurious errors. Ferris can't tell if they made a mistake or hit a compiler bug.

### Gap 5: Static lifetime requirements force Arc everywhere

*See also: [Barbara bridges sync and async]*

Next task: integrate with a legacy service that only has a synchronous client library. Ferris knows not to block the async runtime, so they reach for `spawn_blocking`:

```rust
async fn fetch_legacy_data(&self, key: &str) -> LegacyData {
    let result = tokio::task::spawn_blocking(|| {
        self.legacy_client.fetch(key)
    }).await.unwrap();
    result
}
```

```
error[E0373]: closure may outlive the current function, but it borrows `self`
  --> src/lib.rs:67:47
   |
67 |     let result = tokio::task::spawn_blocking(|| {
   |                                               ^^ may outlive borrowed value `self`
68 |         self.legacy_client.fetch(key)
   |         ---- `self` is borrowed here
```

Right, `spawn_blocking` moves the closure to another thread, so it needs `'static`. Ferris tries to clone what they need:

```rust
async fn fetch_legacy_data(&self, key: &str) -> LegacyData {
    let client = self.legacy_client.clone();
    let key = key.to_owned();
    tokio::task::spawn_blocking(move || {
        client.fetch(&key)
    }).await.unwrap()
}
```

But `LegacyClient` doesn't implement `Clone`. It holds a connection pool that can't be cloned. Ferris ends up wrapping it in `Arc`:

```rust
struct MyService {
    legacy_client: Arc<LegacyClient>,  // was just LegacyClient
    // ...
}
```

Now they can clone the `Arc`. But this pattern spreads - soon half the fields are wrapped in `Arc` just to satisfy `spawn_blocking`'s lifetime requirements. The code compiles, but it feels wrong. In sync Rust, Ferris would just call the function. Here, they're restructuring their entire data model to work around async's constraints.

### Gap 6: No async drop

*See also: [Alan finds database drops hard]*

Ferris's caching layer holds database connections. In sync Rust, they'd implement `Drop` to close connections cleanly:

```rust
impl Drop for DatabasePool {
    fn drop(&mut self) {
        self.close_all_connections(); // sync cleanup
    }
}
```

But Crabby Corp's database client requires async operations to close connections properly - it needs to send a graceful shutdown message and wait for acknowledgment. There's no way to do this in `Drop`:

```rust
impl Drop for DatabasePool {
    fn drop(&mut self) {
        // Can't do this - drop isn't async!
        self.close_all_connections().await;
    }
}
```

Ferris looks for workarounds. They find suggestions to spawn a cleanup task, but that has its own problems - the task might not complete before the process exits, and it can't borrow from the dropped value. Other suggestions involve manual cleanup methods:

```rust
impl DatabasePool {
    async fn shutdown(self) {
        self.close_all_connections().await;
    }
}
```

But this requires callers to remember to call `shutdown()`. If they forget, connections leak. In sync Rust, `Drop` guarantees cleanup happens. In async Rust, Ferris has to choose between automatic cleanup that might not work correctly and manual cleanup that might be forgotten.

### The ecosystem is waiting

As Ferris digs deeper into the async ecosystem, they notice a pattern: many libraries are usable but feel unfinished. Documentation mentions "1.0 coming soon" or links to tracking issues for major redesigns. Tower, the middleware library everyone recommends, is still on 0.x after years of development.

Ferris asks a senior colleague why. The answer: the ecosystem is waiting on language features.

Tower's `Service` trait, for example, predates `async fn` in traits. It uses an associated `Future` type and complex bounds to work around limitations that no longer exist. The maintainers *want* to ship a cleaner design using `async fn`, but they're blocked on a key problem: they need to support both `Send` and non-`Send` futures cleanly.

Work-stealing executors like Tokio require `Send` futures. But single-threaded and thread-per-core designs don't - and forcing `Send` on them adds unnecessary constraints. Tower needs a way to define *one* trait that library authors implement, while letting users choose whether they need the `Send` variant. Today, that requires duplicating traits or using complex workarounds. The language doesn't have a clean solution.

So Tower waits. And because Tower waits, the middleware ecosystem built on it stays in flux. Libraries depend on unstable APIs, and patterns don't converge. Ferris realizes the gaps aren't just affecting their own code - they're holding back the entire async ecosystem.

### What went wrong?

None of these problems were about async concepts like futures or polling. Ferris understands concurrency. The problems were about *incidental complexity*:

- Proc macros required for basic trait patterns
- Arcane signatures required for patterns that are trivial in sync Rust
- `Send` bounds that don't correspond to anything in the sync mental model
- Compiler bugs that reject valid code
- Lifetime requirements that force architectural changes
- Missing language features for cleanup patterns that work in sync Rust

Each issue has a workaround. But the workarounds require knowledge that doesn't transfer from sync Rust, and the compiler errors don't guide you to them. Developers who would otherwise build a network service in Java, JavaScript, or Go hit these walls and wonder if Rust is worth the trouble.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This section describes the features we propose to close the gaps identified above.

## Dynamic dispatch for async traits (AFIDT)

This RFC establishes that Rust will support dynamic dispatch for `async fn` in traits (and potentially other suitable RPITIT methods) with these properties:

- Boxing may be required at the call site
- `&dyn Trait` works as a type with no special annotations required on the trait definition or use-site

The expected mechanism is the `.box` operator described in the [box-box-box] design, though the specific syntax will be determined in a follow-up RFC. The key insight is that async trait methods can return boxed futures when used through dynamic dispatch, while static dispatch continues to use unboxed futures.

At the call site, developers use `.box` to opt into boxing:

```rust
async fn fetch_user(cache: &dyn Cache, id: UserId) -> User {
    if let Some(bytes) = cache.get(&id.to_string()).box.await {
        //                                         ^^^^ boxes the future
        return deserialize(&bytes);
    }
    // ...
}
```

The `.box` makes the allocation explicit and visible. The compiler provides a clear error message when it's needed.

**Where are we now:** The design direction is described in the [box-box-box] blog post. A follow-up RFC will propose the specific syntax. We expect to draft that RFC and begin implementation in 2026, with stabilization possible by late 2026 or early 2027.

## Boxed async functions

The `box async fn` syntax enables recursive async functions without arcane signatures:

```rust
box async fn fetch_nested(&self, id: &str) -> NestedData {
    let data = self.fetch_one(id).await;
    if let Some(parent_id) = data.parent {
        let parent = self.fetch_nested(&parent_id).await;
        data.with_parent(parent)
    } else {
        data
    }
}
```

The `box` keyword tells the compiler to box the returned future, allowing recursion to work naturally.

**Where are we now:** This is part of the same design as AFIDT ([box-box-box]). No RFC yet. Expected to be developed alongside AFIDT in 2026.

## Return Type Notation (RTN)

RTN allows expressing bounds on the futures returned by async trait methods:

```rust
fn spawn_health_checker<H>()
where
    H: HealthCheck,
    H::check(..): Send,  // "the future returned by check() must be Send"
{
    tokio::spawn(async move {
        let checker = H::new();
        checker.check().await
    });
}
```

This solves the "send bound problem" - you can write traits that don't hardcode `Send`, while callers can require `Send` when they need it.

**Where are we now:** RTN has an [accepted RFC][rtn-rfc] and a working implementation. Stabilization did not proceed due to dependencies on the [next-generation trait solver][next-gen-solver]. Once the trait solver work lands (expected 2026), RTN can move toward stabilization.

[rtn-rfc]: https://rust-lang.github.io/rfcs/3654-return-type-notation.html

## RTN for closures

A similar notation for async closures (likely `F(..): Send`):

```rust
fn spawn_task<F>(f: F)
where
    F: AsyncFn(),
    F(..): Send,  // "the future returned by this closure must be Send"
{
    tokio::spawn(f());
}
```

**Where are we now:** This was left as future work in the RTN RFC. No RFC for the specific syntax yet. Expected to follow RTN stabilization.

## Fix spurious Send errors

[Issue #149407][149407] causes the compiler to incorrectly reject valid async code. This fix depends on the [next-generation trait solver][next-gen-solver] and specifically on improvements to how implications are handled.

Once fixed, code that *should* compile will compile - no user-visible syntax changes, just fewer false errors.

**Where are we now:** The fix requires the next-generation trait solver. That work is ongoing with a 2026 target. Once it lands, this bug can be addressed.

## Guaranteed destructors (linear types)

This RFC establishes that Rust will support some mechanism for guaranteeing that certain values cannot be leaked via `mem::forget` - the compiler will ensure their destructors run.

This enables safe scoped parallelism - APIs like `spawn_blocking` can borrow data without requiring `'static`, because the compiler guarantees the spawned task's future will complete (and thus release the borrow) before the borrowed data goes out of scope.

```rust
async fn fetch_legacy_data(&self, key: &str) -> LegacyData {
    // Just works - no Arc needed
    tokio::task::spawn_blocking(|| {
        self.legacy_client.fetch(key)
    }).await
}
```

**Where are we now:** The [move, destruct, forget][linear-types-post] blog post outlines one possible design direction, but RFCs and further exploration are needed to work out the details. This requires significant language work. Exploration and design work expected in 2026, with implementation and stabilization in 2027 or beyond.

## Async destructors

Building on guaranteed destructors, this allows types to perform async cleanup when dropped in an async context:

```rust
impl AsyncDrop for DatabasePool {
    async fn drop(&mut self) {
        self.close_all_connections().await;
    }
}
```

Resources like database connections can now clean up properly without manual shutdown methods.

**Where are we now:** Async destructors depend on guaranteed destructors (you need to ensure the async destructor actually runs). Design exploration is ongoing. Stabilization expected 2027 or beyond.

## Ferris revisited

With these features in place, let's revisit Ferris's experience at Crabby Corp.

### Dynamic dispatch: add `.box`

Ferris defines the cache trait and tries to use `&dyn Cache`:

```rust
trait Cache {
    async fn get(&self, key: &str) -> Option<Vec<u8>>;
    async fn set(&self, key: &str, value: &[u8]);
}

async fn fetch_user(cache: &dyn Cache, db: &Database, id: UserId) -> User {
    if let Some(bytes) = cache.get(&id.to_string()).await {
        // ...
    }
}
```

```
error[E0038]: async methods require boxing for dynamic dispatch
  --> src/lib.rs:12:34
   |
12 |     if let Some(bytes) = cache.get(&id.to_string()).await {
   |                                ^^^^^^^^^^^^^^^^^^^^^^^^^ add `.box` here
   |
help: box the future to enable dynamic dispatch
   |
12 |     if let Some(bytes) = cache.get(&id.to_string()).box.await {
   |                                                     ++++
```

Ferris adds `.box`, and it works. The error told them exactly what to do.

### Recursion: use `box async fn`

Ferris writes a recursive async function:

```rust
async fn fetch_nested(&self, id: &str) -> NestedData {
    // ...recursive call...
}
```

```
error[E0733]: recursion in an `async fn` requires boxing
 --> src/lib.rs:71:5
   |
71 | async fn fetch_nested(&self, id: &str) -> NestedData {
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
help: use `box async fn` to box the returned future
   |
71 | box async fn fetch_nested(&self, id: &str) -> NestedData {
   | +++
```

Ferris adds `box`, and recursion works. No arcane signatures needed.

### Send bounds: RTN makes them expressible

Ferris needs to spawn a background task that calls a trait method:

```rust
trait Cache {
    async fn process(&self);
}

async fn background_process(cache: &impl Cache<process(..): Send>)
{
    tokio::spawn(async move {
        cache.process().await
    });
}
```

RTN (`Cache<process(..): Send>`) lets Ferris express exactly what they need: "I require an implementation where `process()` returns a `Send` future." No closures, no `BoxFuture`, just a bound.

And with the trait solver bug fixed, code that *should* compile *does* compile - no more mysterious rejections of valid code.

### Implementable trait aliases: the `Service` pattern

Many libraries want to offer two variants of a trait: one that requires `Send` futures (for use with work-stealing executors like Tokio) and one that doesn't (for single-threaded or thread-per-core designs). With implementable trait aliases, libraries can define:

```rust
trait LocalService {
    type Response;
    async fn request(&self) -> Self::Response;
}

trait Service = LocalService<request(..): Send> + Send;
```

Users can then implement either trait depending on their needs:

```rust
impl Service for MyType {
    type Response = String;
    async fn request(&self) -> String { ... }
}
```

And library authors can write code that's generic over both:

```rust
fn process<S: LocalService>(service: S) { ... }  // works with any service
fn spawn_process<S: Service>(service: S) { ... } // requires Send
```

This pattern is particularly important for Tower's `Service` trait, which currently uses complex workarounds to achieve similar flexibility.

**Note:** The exact mechanism for implementable trait aliases is not yet RFC'd. What this RFC establishes is the *goal*: users should be able to define two related traits where implementors can choose either one, and generic code can work with both. The specific syntax and semantics will be determined in follow-up RFCs. Full support is expected in 2027 at the earliest.

### `spawn_blocking`: just borrow

With guaranteed destructors, Ferris can borrow into `spawn_blocking`:

```rust
async fn fetch_legacy_data(&self, key: &str) -> LegacyData {
    tokio::task::spawn_blocking(|| {
        self.legacy_client.fetch(key)
    }).await
}
```

No `Arc` needed. The compiler knows the spawned task will complete before the borrow expires.

### Async drop: automatic cleanup

Ferris implements `AsyncDrop` for the database pool, and connections clean up automatically. No manual shutdown methods, no forgotten cleanup.

### Ecosystem benefits

With these language features in place, the ecosystem can evolve. Ferris finds that libraries like Tower have stabilized, providing middleware abstractions that work across runtimes. When Ferris needs to add logging, rate limiting, or retry logic to their service, they reach for well-documented middleware that composes cleanly:

```rust
use tower::{Service, ServiceBuilder};

let service = ServiceBuilder::new()
    .rate_limit(100, Duration::from_secs(1))
    .timeout(Duration::from_secs(30))
    .service(my_handler);
```

The `Service` trait uses async functions with Send bounds expressed via RTN, and works with dynamic dispatch thanks to AFIDT. Ferris doesn't need to know these details - they just use the library.

---

The pattern throughout: Ferris's sync Rust knowledge transfers. When async needs something extra (`.box`, `box async fn`), the compiler guides them with clear errors. The "chasm of sadness" becomes a gentle step, and the ecosystem provides the abstractions developers expect from a mature language.

[box-box-box]: https://smallcultfollowing.com/babysteps/blog/2025/03/24/box-box-box/
[linear-types-post]: https://smallcultfollowing.com/babysteps/blog/2025/10/21/move-destruct-leak/
[149407]: https://github.com/rust-lang/rust/issues/149407
[next-gen-solver]: https://github.com/rust-lang/rust-project-goals/issues/113

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Timeline

| Initiative | 2025 (status quo) | 2026 | 2027 | 2028 |
| --- | --- | --- | --- | --- |
| **RTN** | RFC accepted, impl on nightly | Stabilize\* | Stabilize\* | — |
| **Fix #149407** | Blocked on trait solver | Fix lands\* | Fix lands\* | — |
| **AFIDT** | Not supported | RFC + nightly | Stabilize | — |
| **`box async fn`** | Not supported | RFC + nightly | Stabilize | — |
| **RTN for closures** | Not supported | RFC + nightly | Stabilize | — |
| **Implementable trait aliases** | RFC accepted | See [2026 project goal][ita-goal] | | |
| **Guaranteed destructors** | Not supported | Design exploration | RFC + nightly | Stabilize |
| **Async destructors** | Not supported | Design exploration | RFC + nightly | Stabilize |

\* Depends on [next-generation trait solver][next-gen-solver] stabilization

[ita-goal]: https://rust-lang.github.io/rust-project-goals/2026/supertrait-auto-impl.html

## Detailed status

This section catalogs the specific gaps between sync and async Rust that this initiative aims to close.

## Async functions in traits

**Current state:** Async functions in traits require the `#[async_trait]` procedural macro, which boxes the returned future and erases its type.

**Target state:** Native support for `async fn` in traits, stabilized and working without adapter crates.

**Status:** `async fn` in traits is stable as of Rust 1.75, but ergonomic gaps remain around Send bounds and return-position impl trait in traits (RPITIT).

## Async closures

**Current state:** There is no native async closure syntax. Workarounds involve returning async blocks from regular closures, which has limitations around capturing and inference.

**Target state:** Native `async || { ... }` closure syntax that works analogously to regular closures.

**Status:** TODO - current status of async closures work

## Async drop

**Current state:** There is no way to run async code during drop. Resources that require async cleanup (network connections, async locks) cannot be cleaned up in destructors.

**Target state:** A mechanism for types to perform async cleanup when dropped in an async context.

**Status:** TODO - current status of async drop work

## Async iteration

**Current state:** No native support for async iteration. The `for` loop cannot iterate over async streams. Libraries provide `Stream` traits but these don't integrate with language syntax.

**Target state:** Async iterators that work with `for await` loops, analogous to how sync iterators work with `for` loops.

**Status:** TODO - current status of async iteration work

## Send bound computation bugs

**Current state:** The compiler sometimes incorrectly rejects valid async code when computing whether a future is `Send`. These bugs particularly affect code using higher-ranked lifetimes. See issues labeled `fixed-by-higher-ranked-assumptions`.

**Target state:** The compiler correctly computes Send/Sync bounds for async futures without spurious errors.

**Status:** Known bugs exist; fixes require improvements to higher-ranked assumption handling.

## Diagnostic quality

**Current state:** Error messages for async code often expose implementation details (generated future types, opaque wrapper types) rather than concepts the developer is working with.

**Target state:** Error messages for async code should be as clear and actionable as those for sync code.

**Status:** TODO - ongoing diagnostic improvements

## Intentional limitations

The features proposed here are intentionally scoped to deliver value quickly rather than solve every possible use case. This section documents known limitations and how we expect them to be addressed in the future.

### AFIDT does not make `dyn Trait: Trait`

The AFIDT design enables using `&dyn Trait` as a type when the trait has async methods, but it does *not* make `dyn Trait` implement `Trait` in the general sense. This means you cannot write:

```rust
async fn fetch_user<C: ?Sized + Cache>(cache: &C, ...) -> User {
    cache.get(key).await  // works with any Cache, including &dyn Cache
}
```

and have it work seamlessly with both concrete types and `&dyn Cache`. Instead, code that uses dynamic dispatch must be written specifically for `&dyn Cache` with `.box` at call sites.

This limitation exists because making `dyn Trait: Trait` hold in general requires solving harder problems around how the boxed future is allocated and returned. The current design sidesteps this by making boxing explicit at the call site.

### `.box` only supports global allocator

The `.box` expression allocates using the global allocator. Support for custom allocators, in-place initialization, or stack-allocated boxed futures is not part of this proposal.

We expect these capabilities to emerge from the [in-place initialization project goal][in-place-init], which is exploring more general mechanisms for constructing values directly in their final location. Once that work matures, we anticipate extending the boxing story to support additional allocation strategies.

[in-place-init]: https://github.com/rust-lang/rust-project-goals/issues/395

# Drawbacks
[drawbacks]: #drawbacks

## `.box` may encourage suboptimal allocation patterns

The `.box` operator makes boxing explicit and easy, which is the goal. But this ease may lead developers to use `.box` in situations where a different approach would be more efficient.

For example, consider a loop that calls an async trait method:

```rust
for item in items {
    cache.process(&item).box.await;  // allocates on every iteration
}
```

This allocates a new `Box` on every iteration. A more efficient approach might batch operations or use a single pre-allocated buffer. But because `.box` is simple and the compiler suggests it, developers may not think about alternatives.

This is a tradeoff we accept in the interest of shipping incrementally. The `.box` operator with global allocation is not the final solution - work on in-place initialization and custom allocators continues separately. But the ecosystem has already been blocked for years waiting on language features. Continuing to wait for the fully general solution would extend that delay further. By shipping `.box` now, we unblock libraries like Tower from stabilizing their APIs, even if some users will eventually want more control over allocation. Those users have a path forward as the allocator story matures; in the meantime, `.box` serves the majority of use cases well enough to let the ecosystem move forward.

## Guaranteed destructors add language complexity

Linear types (guaranteed destructors) represent a significant addition to Rust's type system. They introduce new concepts that developers must understand: which types are "must-destruct," how to handle them in error paths, and what happens when they interact with existing code.

This complexity is justified by the problems it solves (safe scoped parallelism, async drop), but it does raise the bar for fully understanding Rust's ownership model.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why "Normal Async" as a specific flavor?

Async Rust serves diverse use cases with different constraints:

- **Embedded/no-std:** Memory allocation may not be available; executor must be minimal
- **High-performance systems:** Every allocation matters; custom executors for specific workloads
- **Normal applications:** Memory allocation is fine; developer ergonomics matter most

This RFC focuses on the third category. By explicitly scoping to "normal" async - where allocation is acceptable and the priority is ergonomic parity with sync Rust - we can make clear decisions without trying to satisfy all constraints simultaneously.

This doesn't mean other use cases are less important. It means they may need different tradeoffs and potentially different features.

## What is explicitly out of scope?

This RFC focuses on **language features**. The following are explicitly out of scope:

- **Default/blessed runtime:** Which executor to use is an ecosystem choice, not a language feature
- **Standard library async IO:** Traits like `AsyncRead`/`AsyncWrite` are important but are library concerns
- **Ecosystem consolidation:** Runtime interoperability is valuable but not a language-level problem

The goal is to build out language features that *enable* good ecosystem solutions, not to prescribe those solutions.

## FAQ

### What other challenges exist for these users, and why are they not covered here?

**Standard library abstractions.** Developers building network services would benefit from standardized traits like `AsyncRead`, `AsyncWrite`, and middleware abstractions like [Tower]'s `Service` trait. These are not covered because:

1. They are blocked on language features - both the features in this RFC and others like [implementable trait aliases][ita]. For example, Tower's `Service` trait needs trait aliases to ergonomically express Send bounds without breaking existing code.
2. The right designs require more experimentation in the ecosystem. Once these language features land, we effectively unblock Tower to ship 1.0, and the ecosystem can converge on patterns before any potential standardization.

This RFC aims to unblock that experimentation rather than prescribe standard library additions.

**Portability across runtimes.** Today, choosing a library often means choosing a runtime. Libraries built on Tokio may not work with async-std, and vice versa. Standardized traits would help, but those traits need the language features we're proposing here. See the [wg-async vision document][wg-async-portable] for more on this challenge.

**Streaming and async iteration.** Async iterators (streams) are not covered by this RFC and would be follow-up work. However, with the features proposed here, it is already possible to write "async iterators" in the same form as one writes iterators today in Rust—i.e., a struct with an `async fn next(&mut self) -> Option<Item>` method. There's no built-in `AsyncIterator` trait support yet, but users can define their own trait that behaves similarly to `Iterator`. The language features in this RFC (particularly AFIDT) make such user-defined traits more practical.

**Library recommendations.** "Which async runtime should I use?" is a common question. Recommendations are out of scope because it's unclear how the Rust project should approach library recommendations, and this is not specific to async.

### What do we gain by focusing on this cohort?

Focusing on "normal async" (where allocation is acceptable) lets us ship improvements sooner:

**Simpler boxing story.** The `.box` expression and `box async fn` proposed here are not the most general mechanisms for boxed dispatch. Work on in-place initialization and custom allocators continues separately, and we expect to eventually support more than just `.box`. But waiting for the fully general solution would delay improvements for the majority of users who don't need that generality.

**Concrete tradeoffs.** By scoping to environments where allocation is acceptable, we can make clear design decisions. Trying to satisfy embedded constraints simultaneously would slow progress or lead to unsatisfying compromises for everyone.

[Tower]: https://docs.rs/tower
[ita]: https://github.com/rust-lang/rfcs/pull/3437
[wg-async-portable]: https://rust-lang.github.io/wg-async/vision/roadmap/portable.html

# Prior art
[prior-art]: #prior-art

TODO: Survey async in other languages (C#, JavaScript, Python, Kotlin, Swift)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO

# Future possibilities
[future-possibilities]: #future-possibilities

TODO

<!-- Status quo story links -->
[Alan needs async in traits]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/alan_needs_async_in_traits.html
[Alan thinks he needs async locks]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/alan_thinks_he_needs_async_locks.html
[Barbara bridges sync and async]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_bridges_sync_and_async.html
[Alan tries processing some files]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/alan_tries_processing_some_files.html
[Alan finds database drops hard]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/alan_finds_database_drops_hard.html
