# 06 — Traits

## Reading
- The Book: [Chapter 10.2](https://doc.rust-lang.org/book/ch10-02-traits.html)
- Rust by Example: [Traits](https://doc.rust-lang.org/rust-by-example/trait.html)

## Key concepts

Traits define shared behaviour — similar to interfaces in other languages. A type opts in by implementing the trait:

```rust
trait Summary {
    fn summarise(&self) -> String;
}

impl Summary for Article {
    fn summarise(&self) -> String {
        format!("{} by {}", self.title, self.author)
    }
}
```

**Trait bounds** let functions accept any type that implements a trait:
```rust
fn print_summary(item: &impl Summary) { ... }
// or equivalently:
fn print_summary<T: Summary>(item: &T) { ... }
```

**Common standard library traits to know:**
- `Display` — how to format for printing (`{}`)
- `Debug` — how to format for debugging (`{:?}`)
- `From` / `Into` — type conversions
- `Iterator` — anything that can be iterated
- `Clone` / `Copy` — duplication behaviour
- `Default` — a sensible zero value

**Trait objects** (`dyn Trait`) allow runtime polymorphism when you need a collection of different types that share a trait.

## Project — Sensor Formatter

Build a system that can format different sensor types (temperature, humidity, pressure) in multiple output formats (human-readable, JSON, CSV) using traits.

```
$ formatter --format json readings.txt
{"sensor":"temp","value":21.3,"unit":"C"}
{"sensor":"humidity","value":61.0,"unit":"%"}

$ formatter --format csv readings.txt
temp,21.3,C
humidity,61.0,%
```

**What this teaches:**
- Defining your own traits
- Implementing the same trait for multiple types
- Using `Box<dyn Trait>` to hold a collection of different sensor types
- Implementing `Display` and `Debug` for your types
- Trait objects vs generics — when to use each

**Steps:**
1. Define a `Sensor` trait with `fn format_json(&self) -> String` and `fn format_csv(&self) -> String`
2. Create `Temperature`, `Humidity`, `Pressure` structs, each implementing `Sensor`
3. Store them as `Vec<Box<dyn Sensor>>` so you can mix types
4. Implement `Display` for each type
5. Match on the `--format` flag to choose output

**Stretch goal:** Add a `Threshold` trait that sensors can optionally implement to report whether a value is out of range.
