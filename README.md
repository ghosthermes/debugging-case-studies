# debugging-case-studies
Published writeups on debugs and troubleshooting

This repository is a public log of technical investigations. It contains the raw notes, terminal output, and thought processes from production failures I've encountered and fixed. 

Most portfolios show finished products. This one shows the messy middle. It documents the hours spent staring at a debugger or arguing with an LLM when the system behavior contradicts the documentation. 

Each study follows a consistent format:
1. The Symptom: What broke and who noticed.
2. Initial Theories: My first (usually wrong) guesses.
3. The Investigation: The tools and commands used to isolate the variable.
4. The Root Cause: The specific line of code or configuration error responsible.
5. The Lesson: How to prevent the same failure next time.

### Repository Structure

```text
/case-studies
  ├── 001-alpine-dns-resolution.md
  ├── 002-rust-async-mutex-deadlock.md
  ├── 003-gh-actions-permission-drift.md
  └── 004-python-pydantic-v2-migration.md
```

---

### Case Study 001: The Alpine DNS Ghost

**Date:** March 14, 2024  
**Subject:** Python/Docker/Networking

#### The Symptom
A small scraping service built on `python:3.11-alpine` worked perfectly on my local machine. As soon as I pushed it to the staging cluster, every outgoing request to the internal API returned a `Name or service not known` error. 

#### Initial Theories
1. The API was down. (Verified: It was up.)
2. The staging VPC had a missing routing table entry. (Verified: Other services could reach it.)
3. I misspelled the environment variable. (Verified: I didn't.)

#### The Investigation
I launched a shell inside the running container to manually test the connection.

```bash
# Inside the container
ping internal-api.staging.svc.cluster.local
# Result: ping: bad address 'internal-api.staging.svc.cluster.local'
```

I checked `/etc/resolv.conf`. The search domains were correct. Then I remembered that Alpine Linux uses `musl libc` instead of `glibc`. 

I used `tcpdump` on the host machine to watch the DNS traffic. I noticed something strange: the container was sending DNS queries for the API address, getting a valid response from the cluster DNS, and then... ignoring it. It kept retrying until it timed out.

#### The Root Cause
The issue was the `ndots:5` configuration in Kubernetes combined with how `musl` handles DNS. In Alpine, if a DNS response is too large for a UDP packet, it doesn't always fail over to TCP correctly in certain older kernel environments. More importantly, `musl` does not support the same asynchronous DNS resolution logic as `glibc`. 

The internal API address was long enough that, combined with the search path suffixes, it triggered a specific edge case in how `musl` parses the resolver response.

#### The Solution
I switched the base image from `python:3.11-alpine` to `python:3.11-slim`. The `slim` image uses Debian and `glibc`. 

```dockerfile
# BEFORE
FROM python:3.11-alpine

# AFTER
FROM python:3.11-slim
```

The DNS resolution worked immediately. 

#### The Lesson
Alpine is great for small image sizes, but the `musl` vs `glibc` difference is a massive hidden variable. If a network issue makes no sense on Alpine, try a Debian-based image first. It saves hours of packet sniffing.

---

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