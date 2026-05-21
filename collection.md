> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Collection

**Module:** `viontin_framework::collection`

Fluent collection wrapper — inspired by Laravel Collections. Provides a rich API for working with arrays of data: map, filter, reduce, sort, chunk, unique, merge, and more.

---

## Quick Start

```rust
use viontin::Collection;

let result = Collection::new(vec![1, 2, 3, 4, 5])
    .filter(|n| **n > 2)
    .map(|n| n * 10)
    .sum();

println!("{}", result); // 120
```

---

## Creating a Collection

```rust
// From Vec
let c = Collection::new(vec![1, 2, 3]);

// From iterator
let c = Collection::from(vec![1, 2, 3]);

// Into Vec
let items: Vec<i32> = c.into_inner();
```

---

## Methods

### Accessing Items

```rust
c.items();       // &[T]
c.count();       // usize
c.is_empty();    // bool
c.first();       // Option<&T>
c.last();        // Option<&T>
c.get(2);        // Option<&T>
```

### Transformation

```rust
c.map(|n| n * 2);              // Collection<U>
c.filter(|n| **n > 10);        // Collection<&T>
c.reduce(0, |sum, n| sum + n); // U
c.sort_by(|a, b| b.cmp(a));    // Collection<T> (descending)
c.reverse();                   // Collection<T>
c.unique();                    // Collection<T>
c.chunk(3);                    // Collection<Vec<&T>>
```

### Iteration

```rust
c.each(|n| println!("{}", n));
c.into_iter();                    // IntoIterator
(&c).into_iter();                 // iterator over &T
```

### Merging

```rust
c.merge(other_collection);       // combine items
```

### Slicing

```rust
c.take(5);                       // first 5 items
c.skip(10);                      // skip first 10
```

### Numeric

```rust
// Available for Collection<i64> and Collection<f64>
c.sum();                         // i64 or f64
c.avg();                         // f64
c.min();                         // Option<i64> or Option<f64>
c.max();                         // Option<i64> or Option<f64>
```

### Serialization (with serde)

```rust
c.to_json();                     // Result<String, String>
```

---

## Complete Example

```rust
use viontin::Collection;

let data = Collection::new(vec![
    ("Alice", 31, true),
    ("Bob", 25, false),
    ("Charlie", 35, true),
]);

let active_names: Vec<String> = data
    .filter(|(_, _, active)| **active)
    .map(|(name, _, _)| name.to_string())
    .into_inner();

println!("{:?}", active_names);
// ["Alice", "Charlie"]
```
