# 05 — Collections and Iterators

## Reading
- The Book: [Chapter 8](https://doc.rust-lang.org/book/ch08-00-common-collections.html) — Vec, String, HashMap
- The Book: [Chapter 13](https://doc.rust-lang.org/book/ch13-00-functional-features.html) — Iterators and closures

## Key concepts

**`Vec<T>`** — growable array, the most common collection.

**`HashMap<K, V>`** — key/value store, keys must implement `Hash + Eq`.

**Iterators** are lazy — they don't do any work until consumed. This lets you chain operations efficiently without intermediate allocations:
```rust
let total: f64 = readings
    .iter()
    .filter(|r| r.valid)
    .map(|r| r.value)
    .sum();
```

Key iterator methods:
- `.map()` — transform each element
- `.filter()` — keep elements matching a predicate
- `.fold()` — reduce to a single value
- `.collect()` — consume into a collection
- `.enumerate()` — pair each element with its index
- `.zip()` — combine two iterators

**Iterator adapters are zero-cost** — the compiler optimises chains into a single loop.

## Project — Log Analyser

Parse a server log file and produce an analysis report using iterators throughout — no manual loops.

```
$ log-analyser server.log
Total requests: 4,821
Unique IPs:     143
Top 5 endpoints:
  /api/metrics    1,203 requests
  /health           892 requests
  /api/status       441 requests
Status breakdown:
  200: 4,201 (87.1%)
  404:   389 (8.1%)
  500:   231 (4.8%)
```

**What this teaches:**
- Chaining iterator methods instead of writing loops
- Building a `HashMap` from an iterator with `.fold()` or `.entry()`
- Sorting a `Vec` of tuples
- When to use `.iter()` vs `.into_iter()` vs `.iter_mut()`
- Writing clean functional-style data pipelines

**Steps:**
1. Read lines into a `Vec<String>`
2. Parse each line into a struct using `.map()`
3. Use `.filter()` to skip malformed lines
4. Build frequency maps with `.fold()` into `HashMap`
5. Sort and display top entries

**Stretch goal:** Stream the file line by line with `BufReader` instead of loading it all into memory — good practice for large logs.
