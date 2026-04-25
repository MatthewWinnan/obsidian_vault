# Learning Rust

Rust is a systems programming language focused on safety, speed, and concurrency. It achieves memory safety without a garbage collector through its ownership model — the central concept that makes Rust unique.

## Philosophy

Learn by doing. Each concept below has:
- A reference to where to read about it
- A small focused project to cement it
- The final section combines everything into one capstone project

## Resources

- **The Book** — [The Rust Programming Language](https://doc.rust-lang.org/book/) (free online, the canonical reference)
- **Rustlings** — Small exercises that follow The Book: `nix run nixpkgs#rustlings`
- **Rust by Example** — [https://doc.rust-lang.org/rust-by-example/](https://doc.rust-lang.org/rust-by-example/) (code-first companion to The Book)
- **Programming Rust** (O'Reilly) — deeper dive, good second book once you have the basics

## Curriculum

| # | Concept | Note |
|---|---------|------|
| 1 | [[01 - Getting Started]] | Toolchain, cargo, hello world |
| 2 | [[02 - Ownership and Borrowing]] | The core of Rust |
| 3 | [[03 - Structs and Enums]] | Custom types |
| 4 | [[04 - Error Handling]] | Result and Option |
| 5 | [[05 - Collections and Iterators]] | Vec, HashMap, iterators |
| 6 | [[06 - Traits]] | Shared behaviour |
| 7 | [[07 - Lifetimes]] | Explicit memory relationships |
| 8 | [[08 - Closures and Functional Patterns]] | Functions as values |
| 9 | [[09 - Concurrency]] | Threads and async |
| 10 | [[10 - Modules and Crates]] | Project structure |
| 11 | [[11 - Capstone Project]] | MQTT sensor aggregator |
