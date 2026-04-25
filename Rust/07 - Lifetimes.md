# 07 — Lifetimes

## Reading
- The Book: [Chapter 10.3](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html)
- Programming Rust (O'Reilly): Chapter 5 — the deepest treatment of lifetimes available

## Key concepts

Lifetimes are how the borrow checker tracks how long references are valid. Most of the time Rust infers them (lifetime elision), but sometimes you need to be explicit.

A lifetime annotation `'a` is not a duration — it's a constraint that says "these references must all be valid for the same scope":

```rust
// The returned reference lives as long as the shorter of x or y
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

**Structs that hold references** need lifetime annotations:
```rust
struct Parser<'a> {
    input: &'a str,  // the struct cannot outlive the string it borrows
}
```

**The golden rule:** a reference cannot outlive the data it points to. Lifetimes are just the compiler making you state this explicitly when it can't figure it out itself.

Most real code doesn't need many explicit lifetimes — you encounter them most often when returning references from functions or storing references in structs.

## Project — Zero-Copy Log Parser

Build a log parser that processes a large log file without copying any strings — all parsed fields borrow slices from the original buffer.

```rust
struct LogEntry<'a> {
    timestamp: &'a str,
    level:     &'a str,
    message:   &'a str,
}
```

**What this teaches:**
- Why you need lifetime annotations on structs that hold references
- The performance benefit of borrowing slices vs cloning strings
- How the borrow checker prevents you from returning a reference to local data
- Lifetime elision rules — when you don't need to write them explicitly

**Steps:**
1. Read the entire log file into a `String` (owned)
2. Write a `parse_line<'a>(line: &'a str) -> Option<LogEntry<'a>>` function that slices into the line without copying
3. Collect into `Vec<LogEntry>` — note the vec cannot outlive the original `String`
4. Filter and print entries by log level

**Stretch goal:** Add a `fn messages_for_level<'a>(entries: &'a [LogEntry], level: &str) -> Vec<&'a str>` function and reason through why the lifetime annotation is needed.
