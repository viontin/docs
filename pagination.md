> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Pagination

**Module:** `viontin_framework::page`

Paginated results for API and list endpoints.

---

## Page

```rust
pub struct Page<T: Clone> {
    pub items: Vec<T>,
    pub total: u64,
    pub per_page: u64,
    pub current_page: u64,
    pub last_page: u64,
    pub has_more: bool,
}
```

```rust
use viontin::prelude::*;

let data = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
let page = Page::new(data.clone(), data.len() as u64, 1, 5);

page.items;          // [1, 2, 3, 4, 5]
page.total;          // 10
page.current_page;   // 1
page.last_page;      // 2
page.per_page;       // 5
page.has_more;       // true
```

### Methods

```rust
page.items();        // &[T]
page.into_items();   // Vec<T>
page.count();        // usize (actual items on this page)
page.is_empty();     // bool
page.has_pages();    // true if last_page > 1
page.first_page();   // true if current_page == 1
page.last_page();    // true if on last page
page.links();        // PaginationLinks
```

---

## PaginationLinks

```rust
pub struct PaginationLinks {
    pub first: Option<u64>,
    pub last: Option<u64>,
    pub prev: Option<u64>,
    pub next: Option<u64>,
    pub current: u64,
}
```

```rust
let links = page.links();
links.first;    // Some(1)
links.last;     // Some(2)
links.prev;     // None (first page)
links.next;     // Some(2)
links.current;  // 1
```

---

## paginate()

Full pagination with total:

```rust
let items = vec!["a", "b", "c", "d", "e", "f", "g"];
let page = paginate(&items, items.len() as u64, 2, 3);

page.items;         // ["d", "e", "f"]
page.total;         // 7
page.current_page;  // 2
page.last_page;     // 3
page.has_more;      // true
```

---

## simple_paginate()

Simple pagination without total count — for infinite scroll / "load more":

```rust
let page = simple_paginate(&items, 1, 5);
page.items;     // up to 5 items
page.total;     // 0 (unknown)
page.has_more;  // true if more items exist (fetched N+1)
```

Fetches `per_page + 1` items to determine if more exist, then discards the extra.

---

## Complete Example

```rust
use viontin::prelude::*;

fn list_users(page_num: u64) -> Page<String> {
    let all_users = vec![
        "Alice".into(), "Bob".into(), "Charlie".into(),
        "Diana".into(), "Eve".into(), "Frank".into(),
    ];

    paginate(&all_users, all_users.len() as u64, page_num, 2)
}

fn handler(req: Request) -> Response {
    let page: u64 = req.query("page")
        .and_then(|s| s.parse().ok())
        .unwrap_or(1);

    let result = list_users(page);

    Response::json(&result).unwrap_or_else(|_| Response::html("Error"))
}
```
