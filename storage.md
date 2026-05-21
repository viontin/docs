# Storage

**Module:** `viontin_framework::storage`  
**Facade:** `Storage`

Filesystem abstraction with multiple disk drivers — swap between local disk, in-memory, or cloud storage without changing application code.

---

## Architecture

```
Storage (facade)
  │
  ├── multiple named disks
  ├── default disk routing
  │
  └── Box<dyn Driver>
        │
        ├── LocalStorage    — filesystem-backed
        └── MemoryStorage   — in-memory (testing)
```

---

## Quick Start

```rust
use viontin::prelude::*;

// Default: local disk at ./storage
let storage = Storage::new();

// Write
storage.put("avatars/user1.png", &image_bytes)?;

// Read
let bytes = storage.get("avatars/user1.png")?;

// Check
if storage.exists("avatars/user1.png") {
    // ...
}
```

---

## Storage Facade

The `Storage` struct manages multiple named disks with a default:

```rust
let mut storage = Storage::new();

// Default disk is "local" at ./storage
storage.get("path/to/file")?;        // uses default disk
storage.put("path/to/file", bytes)?; // uses default disk
storage.exists("path/to/file");      // uses default disk
```

### Multiple Disks

```rust
let mut storage = Storage::new();
storage.add("local", LocalStorage::new("./storage"));
storage.add("backup", LocalStorage::new("/backup"));
storage.add("cache", MemoryStorage::new());

// Access specific disk
let disk = storage.disk("backup").unwrap();
disk.put("snapshot.tar.gz", &data)?;

// Default disk (local) still routes through storage.get() etc.
```

### Methods

```rust
// Default disk
storage.get("path")?;        // Vec<u8>
storage.put("path", bytes)?; // ()
storage.exists("path");      // bool

// Specific disk
storage.disk("s3")
    .ok_or("disk not found")?
    .get("path")?;
```

---

## Driver Trait

```rust
pub trait Driver: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn get(&self, path: &str) -> Result<Vec<u8>, String>;
    fn put(&self, path: &str, content: &[u8]) -> Result<(), String>;
    fn exists(&self, path: &str) -> bool;
    fn delete(&self, path: &str) -> Result<(), String>;
    fn files(&self, directory: &str) -> Result<Vec<String>, String>;
    fn url(&self, path: &str) -> String;
    fn size(&self, path: &str) -> Result<u64, String>;
    fn copy(&self, from: &str, to: &str) -> Result<(), String>;
    fn move_file(&self, from: &str, to: &str) -> Result<(), String>;
}
```

| Method | Returns | Description |
|--------|---------|-------------|
| `name()` | `&str` | Driver identifier |
| `get(path)` | `Result<Vec<u8>>` | Read file contents |
| `put(path, content)` | `Result<()>` | Write file (creates directories) |
| `exists(path)` | `bool` | Check if file exists |
| `delete(path)` | `Result<()>` | Delete file or directory |
| `files(directory)` | `Result<Vec<String>>` | List files in directory |
| `url(path)` | `String` | Public URL for file |
| `size(path)` | `Result<u64>` | File size in bytes |
| `copy(from, to)` | `Result<()>` | Copy file within disk |
| `move_file(from, to)` | `Result<()>` | Move/rename file |

---

## Built-in Drivers

### LocalStorage

Persists files on the local filesystem:

```rust
use viontin_framework::storage::LocalStorage;

// Root at ./storage
let driver = LocalStorage::new("./storage");
let driver = LocalStorage::new("/var/www/storage");

// Default URL: /storage/<path>
driver.url("avatars/user.png");
// "/storage/avatars/user.png"
```

**Characteristics:**
- All paths are relative to the configured root
- `put()` creates parent directories automatically
- `delete()` handles both files and directories
- Best for single-server production deployments

### MemoryStorage

In-memory storage using `HashMap`:

```rust
use viontin_framework::storage::MemoryStorage;

let driver = MemoryStorage::new();
```

**Characteristics:**
- Data lost on process restart
- No disk I/O — fastest option
- Best for testing and ephemeral data

---

## Complete Example

```rust
use viontin::prelude::*;
use viontin_framework::storage::{Storage, LocalStorage, MemoryStorage};

fn main() {
    let mut storage = Storage::new();

    // Configure disks
    storage.add("local", LocalStorage::new("./storage"));
    storage.add("cache", MemoryStorage::new());
    storage.add("backup", LocalStorage::new("/mnt/backup"));

    // Upload a file to default disk (local)
    let avatar_data = std::fs::read("tmp/upload.png").unwrap();
    storage.put("avatars/user_42.png", &avatar_data).unwrap();

    // Read it back
    let bytes = storage.get("avatars/user_42.png").unwrap();
    println!("Avatar size: {} bytes", bytes.len());

    // Use a different disk
    let backup_disk = storage.disk("backup").unwrap();
    backup_disk.put("db/dump.sql", b"SELECT 1;").unwrap();

    // List files
    let files = storage.disk("local").unwrap().files("avatars").unwrap();
    println!("Avatars: {:?}", files);

    // Get public URL
    let url = storage.disk("local").unwrap().url("avatars/user_42.png");
    println!("URL: {}", url);

    // Cleanup
    storage.disk("cache").unwrap().delete("temp/data.json").unwrap();
}
```
