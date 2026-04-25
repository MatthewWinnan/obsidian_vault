# 08 — Closures and Functional Patterns

## Reading
- The Book: [Chapter 13](https://doc.rust-lang.org/book/ch13-00-functional-features.html)
- Rust by Example: [Closures](https://doc.rust-lang.org/rust-by-example/fn/closures.html)

## Key concepts

**Closures** are anonymous functions that can capture variables from their surrounding scope:
```rust
let threshold = 30.0;
let over_threshold = readings.iter().filter(|r| r.value > threshold);
```

The three closure traits determine how they capture their environment:
- `Fn` — borrows immutably, can be called multiple times
- `FnMut` — borrows mutably, can be called multiple times
- `FnOnce` — takes ownership, can only be called once

**Higher-order functions** take or return closures:
```rust
fn apply_transform(readings: Vec<f64>, f: impl Fn(f64) -> f64) -> Vec<f64> {
    readings.into_iter().map(f).collect()
}
```

**`move` closures** force the closure to take ownership of captured variables — essential when passing closures to threads:
```rust
let name = String::from("sensor");
thread::spawn(move || println!("{}", name)); // name is moved into the closure
```

## Project — Sensor Pipeline Builder

Build a configurable data processing pipeline where each processing step is a closure, and pipelines can be composed and reused.

```rust
let pipeline = Pipeline::new()
    .add_step(|v| v * 1.8 + 32.0)   // celsius to fahrenheit
    .add_step(|v| (v * 10.0).round() / 10.0)  // round to 1dp
    .add_step(|v| if v > 100.0 { 100.0 } else { v }); // clamp

let result = pipeline.run(37.0); // 98.6
```

**What this teaches:**
- Storing closures in structs with `Box<dyn Fn>`
- Building composable APIs with closures
- The difference between `Fn`, `FnMut`, `FnOnce` in practice
- Method chaining (builder pattern)
- When closures capture by reference vs by value

**Steps:**
1. Define a `Pipeline` struct holding `Vec<Box<dyn Fn(f64) -> f64>>`
2. Implement `add_step(self, f: impl Fn(f64) -> f64) -> Self` (consumes and returns self for chaining)
3. Implement `run(&self, input: f64) -> f64` that folds through all steps
4. Build several named pipelines (temperature conversion, humidity normalisation etc.) and run readings through them

**Stretch goal:** Make the pipeline generic over the value type `T` instead of hardcoded `f64`.
