# 03 — Structs and Enums

## Reading
- The Book: [Chapter 5](https://doc.rust-lang.org/book/ch05-00-structs.html) — Structs
- The Book: [Chapter 6](https://doc.rust-lang.org/book/ch06-00-enums.html) — Enums and pattern matching

## Key concepts

**Structs** group related data together, like a class without inheritance:
```rust
struct Sensor {
    name: String,
    value: f64,
    unit: String,
}
```

**impl blocks** add methods to structs:
```rust
impl Sensor {
    fn display(&self) -> String {
        format!("{}: {} {}", self.name, self.value, self.unit)
    }
}
```

**Enums** in Rust are far more powerful than in most languages — each variant can hold different data:
```rust
enum Reading {
    Temperature(f64),
    Humidity(f64),
    Status(String),
    Error { code: u32, message: String },
}
```

**Pattern matching** with `match` is exhaustive — the compiler forces you to handle every variant.

## Project — Weather Station Log Parser

Parse a CSV log of sensor readings and produce a summary report.

```
$ weather-parser readings.csv
Temperature: avg 21.3°C  min 14.1°C  max 28.9°C
Humidity:    avg 61.2%   min 44.0%   max 89.3%
Errors:      3 (invalid readings skipped)
```

**What this teaches:**
- Defining structs to represent a parsed row
- Using enums to represent "this reading is temperature OR humidity OR an error"
- Pattern matching to branch on enum variants
- `impl` blocks for formatting and calculation logic
- Separating data (struct) from behaviour (impl)

**Steps:**
1. Define a `Reading` enum with `Temperature(f64)`, `Humidity(f64)`, `Invalid(String)` variants
2. Define a `Summary` struct with min/max/avg fields
3. Parse each CSV line into a `Reading` using `match`
4. Accumulate into a `Summary` and print the report

**Stretch goal:** Add a `SensorType` enum and make the parser detect the column type automatically from the CSV header.
