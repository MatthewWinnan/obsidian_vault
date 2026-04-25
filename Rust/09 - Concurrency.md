# 09 — Concurrency

## Reading
- The Book: [Chapter 16](https://doc.rust-lang.org/book/ch16-00-concurrency.html) — threads, channels, Mutex
- The Book: [Chapter 17](https://doc.rust-lang.org/book/ch17-00-async-await.html) — async/await
- [Tokio tutorial](https://tokio.rs/tokio/tutorial) — the standard async runtime

## Key concepts

Rust's ownership system makes concurrency bugs a compile-time error rather than a runtime surprise.

**Threads:**
```rust
let handle = thread::spawn(move || {
    // runs in a new thread
});
handle.join().unwrap();
```

**Channels** — message passing between threads (like Go):
```rust
let (tx, rx) = mpsc::channel();
thread::spawn(move || tx.send(42).unwrap());
println!("{}", rx.recv().unwrap());
```

**`Mutex<T>`** — shared mutable state protected by a lock. Wrapped in `Arc` for sharing across threads:
```rust
let data = Arc::new(Mutex::new(vec![]));
```

**`Send` and `Sync` traits** — the compiler uses these to enforce that only thread-safe types cross thread boundaries. You rarely implement them manually but need to understand why the compiler rejects certain patterns.

**Async/await** — cooperative concurrency for I/O-bound work (network, files). Uses `tokio` or `async-std` as the runtime. More efficient than threads for many concurrent I/O operations:
```rust
async fn fetch(url: &str) -> Result<String, reqwest::Error> {
    reqwest::get(url).await?.text().await
}
```

## Project — Multi-Sensor Poller

Simulate polling multiple sensors concurrently and aggregating their readings.

```
Polling 5 sensors concurrently...
[sensor-1] 21.3°C  (took 120ms)
[sensor-3] 61.2%   (took 89ms)
[sensor-2] 1013hPa (took 203ms)
[sensor-5] 19.8°C  (took 95ms)
[sensor-4] 58.1%   (took 178ms)
All readings collected in 203ms (vs 685ms sequential)
```

**What this teaches:**
- Spawning multiple threads and joining them
- Using channels to collect results from worker threads
- `Arc<Mutex<T>>` for shared state
- Why `move` closures are required for threads
- Async vs thread-based concurrency — when to use each

**Steps:**
1. Define a `Sensor` struct with a simulated `poll()` method that sleeps for a random duration
2. Spawn one thread per sensor, each sending its result back over a channel
3. Collect all results in the main thread and display them
4. Measure total wall time to demonstrate the concurrency benefit

**Stretch goal:** Rewrite using `tokio::spawn` and `async fn` instead of threads — compare the code and understand the tradeoffs.
