# 02 — Ownership and Borrowing

## Reading
- The Book: [Chapter 4](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)
- Rust by Example: [Ownership and moves](https://doc.rust-lang.org/rust-by-example/scope/move.html)

## Key concepts
Ownership is Rust's answer to memory safety without garbage collection. Three rules govern it:

1. Each value has exactly one owner
2. When the owner goes out of scope, the value is dropped (memory freed)
3. There can only be one mutable reference OR many immutable references at a time — never both

**Move vs Copy** — assigning a heap value (like `String`) moves ownership. Assigning a stack value (like `i32`) copies it.

**Borrowing** — pass a reference (`&T`) instead of ownership so the caller keeps its value. Mutable borrows (`&mut T`) allow modification but only one can exist at a time.

The borrow checker enforces all of this at compile time — no runtime cost.

## Project — Text Statistics Tool

Build a tool that reads a text file and reports statistics about it.

```
$ textstats article.txt
Words:      1,042
Lines:      87
Characters: 5,891
Unique words: 430
Most common: "the" (52 times)
```

**What this teaches:**
- Reading files with `std::fs::read_to_string`
- Why you can't use a `String` after moving it into a function
- Passing `&str` (borrowed string slices) vs owned `String`
- Multiple immutable borrows (counting words while also iterating lines)
- The difference between `&str` and `String`

**Steps:**
1. Read the file into a `String` (owned)
2. Pass `&str` references to separate functions for each stat
3. Use a `HashMap<&str, usize>` to count word frequencies — note you're borrowing slices of the original string, not copying them
4. Find the most common word

**Stretch goal:** Accept multiple files and aggregate totals across all of them.
