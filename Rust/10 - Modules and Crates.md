# 10 — Modules and Crates

## Reading
- The Book: [Chapter 7](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)
- [crates.io](https://crates.io) — the package registry

## Key concepts

**Crate** — the smallest compilation unit. A binary crate has a `main.rs`, a library crate has a `lib.rs`.

**Module** — a namespace within a crate, defined with `mod`:
```rust
mod sensors {
    pub struct Reading { ... }
    pub fn parse(raw: &str) -> Reading { ... }
}
```

**`pub`** controls visibility. Everything is private by default — only `pub` items are accessible outside the module.

**`use`** brings items into scope:
```rust
use sensors::Reading;
use std::collections::HashMap;
```

**Workspaces** — a `Cargo.toml` at the root that groups multiple related crates. Useful when your project grows into distinct library + binary components.

**Key crates worth knowing early:**
| Crate | Purpose |
|-------|---------|
| `serde` / `serde_json` | Serialisation/deserialisation |
| `tokio` | Async runtime |
| `reqwest` | HTTP client |
| `clap` | CLI argument parsing |
| `thiserror` | Custom error types |
| `tracing` | Structured logging |
| `rumqttc` | MQTT client |

## Project — Refactor a Previous Project into a Library

Take your log analyser (lesson 05) or sensor formatter (lesson 06) and restructure it into a proper library crate + binary crate.

```
my-analyser/
├── Cargo.toml
└── src/
    ├── main.rs       ← binary, just CLI parsing + calls into lib
    ├── lib.rs        ← public API
    ├── parser.rs     ← mod parser
    ├── report.rs     ← mod report
    └── error.rs      ← mod error
```

**What this teaches:**
- Splitting code across multiple files and modules
- Deciding what should be `pub` and what should stay private
- The difference between `lib.rs` and `main.rs`
- Writing a clean public API that hides implementation details
- How `use` and `mod` interact

**Steps:**
1. Create a new workspace or restructure the existing project
2. Move parsing logic into `parser.rs`, reporting into `report.rs`
3. Expose only what's needed via `pub` in `lib.rs`
4. Keep `main.rs` thin — it only parses CLI args and calls library functions
5. Write a basic integration test in `tests/` that uses your library as an external consumer would

**Stretch goal:** Publish to a private registry or just verify `cargo doc` generates clean documentation for your public API.
