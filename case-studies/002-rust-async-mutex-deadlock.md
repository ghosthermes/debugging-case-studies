### Case Study 002: The Silent Deadlock

**Date:** April 2, 2024  
**Subject:** Rust/Tokio/Async

#### The Symptom
My CLI tool (`tracehound`) would randomly hang. No error message. No CPU spike. Just total silence. It usually happened after about 50 requests.

#### Initial Theories
1. An external API was hanging the connection.
2. I hit a rate limit and the code was retrying forever.
3. A classic deadlock.

#### The Investigation
I was using `tokio` for concurrency. I suspected a resource was being held open. I used a local LLM to review the code and it pointed out that I was using `std::sync::Mutex` instead of `tokio::sync::Mutex` in an async block.

I didn't believe it at first. I added print statements before and after every lock acquisition. 

```rust
println!("Attempting to lock...");
let data = self.state.lock().unwrap();
println!("Lock acquired.");
// ... some async work here ...
println!("Dropping lock.");
```

The output stopped at "Attempting to lock..." right after a task had yielded to the executor with an `.await` point.

#### The Root Cause
I was holding a synchronous `std::sync::Mutex` across an `.await` point. 

In Rust, if you use a standard mutex and then call `.await`, the thread is suspended while holding the lock. If the executor tries to run another task on that same thread that also needs the lock, you get a deadlock. The thread is waiting for itself.

#### The Solution
I swapped the standard mutex for the `tokio` version, which is designed for async contexts.

```rust
// FROM THIS
use std::sync::Mutex;

// TO THIS
use tokio::sync::Mutex;

// And updating the call site
let data = self.state.lock().await;
```

#### The Lesson
Never hold a `std` mutex across an await point. If you need to share state in an async Rust program, use the primitives provided by your runtime. I now use a custom `clippy` lint to catch this before I compile.