# 01 — Getting Started

## Reading
- The Book: [Chapter 1](https://doc.rust-lang.org/book/ch01-00-getting-started.html)
- The Book: [Chapter 3](https://doc.rust-lang.org/book/ch03-00-common-programming-concepts.html) — variables, types, functions, control flow

## Key concepts
- `rustup` manages the toolchain (already available via `nix`)
- `cargo` is the build system and package manager — handles compiling, testing, dependencies
- Variables are immutable by default, use `mut` to make them mutable
- Rust is statically typed but infers types in most cases
- `println!` is a macro (note the `!`), not a function

## Project — CLI Unit Converter

Build a command-line tool that converts between units.

```
$ converter 100 km miles
100 km = 62.14 miles

$ converter 37 celsius fahrenheit
37°C = 98.6°F
```

**What this teaches:**
- Setting up a cargo project (`cargo new`)
- Reading command line arguments (`std::env::args`)
- Parsing strings to numbers
- Basic functions and control flow
- Printing formatted output

**Steps:**
1. `cargo new converter`
2. Read args from `std::env::args().collect::<Vec<String>>()`
3. Match on the unit pair and apply the conversion formula
4. Print the result

**Stretch goal:** Add a `--list` flag that prints all supported conversions.
