# 04 — Error Handling

## Reading
- The Book: [Chapter 9](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
- Rust by Example: [Error handling](https://doc.rust-lang.org/rust-by-example/error.html)

## Key concepts

Rust has no exceptions. Errors are values returned from functions.

**`Option<T>`** — for values that may or may not exist:
```rust
fn find_user(id: u32) -> Option<User> { ... }
```

**`Result<T, E>`** — for operations that can fail with an error:
```rust
fn read_config(path: &str) -> Result<Config, io::Error> { ... }
```

**The `?` operator** propagates errors up the call stack automatically — replaces verbose `match` on every call:
```rust
fn load() -> Result<Config, io::Error> {
    let text = std::fs::read_to_string("config.toml")?; // returns early on error
    Ok(parse(text))
}
```

**`unwrap()` and `expect()`** — panic on error, fine for prototyping, not for production code.

Custom error types let you define your own errors and use `From` to convert between them — the `thiserror` crate makes this ergonomic.

## Project — Config File Loader

Build a tool that loads a TOML config file, validates it, and prints a summary. Handle every failure gracefully with descriptive error messages.

```
$ config-loader config.toml
Loaded config: host=fr3yr port=1883 interval=30s

$ config-loader missing.toml
Error: could not read file: No such file or directory (os error 2)

$ config-loader broken.toml
Error: invalid config: missing required field "host"
```

**What this teaches:**
- Returning `Result` from your own functions
- Using `?` to propagate errors cleanly
- Defining a custom error enum with `thiserror`
- Converting between error types with `From`
- The difference between recoverable errors (bad config) and unrecoverable ones (bug in code)

**Steps:**
1. Add `toml` and `thiserror` to `Cargo.toml`
2. Define a `ConfigError` enum: `Io(io::Error)`, `Parse(toml::de::Error)`, `MissingField(String)`
3. Write a `load_config(path: &str) -> Result<Config, ConfigError>` function using `?`
4. In `main`, match on the result and print a friendly error message

**Stretch goal:** Validate that port is between 1 and 65535 and return a `ConfigError::InvalidValue` if not.
