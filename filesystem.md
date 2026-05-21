> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Filesystem

**Module:** `viontin_framework::fs`

Safe file and directory operations with consistent error handling, path utilities, and temporary directory support.

---

## File Operations

### Read

```rust
use viontin::prelude::*;

// As string
let content = fs::read("path/to/file.txt")?;

// As bytes
let bytes = fs::read_bytes("path/to/image.png")?;
```

### Write

```rust
// Write (creates parent directories automatically)
fs::write("storage/logs/app.log", "log entry")?;

// Write bytes
fs::write("storage/images/photo.png", &image_bytes)?;
```

### Append

```rust
fs::append("storage/logs/app.log", "another line\n")?;
```

### Delete

```rust
fs::delete("temp/old_file.txt")?;
```

### Check

```rust
if fs::exists("config.json") {
    // ...
}
```

### Metadata

```rust
let file_size = fs::size("data.csv")?;           // u64
let modified = fs::last_modified("data.csv")?;     // SystemTime
let ext = fs::extension("data.csv");               // Some("csv")
let name = fs::stem("data.csv");                   // Some("data")
```

### Copy

```rust
fs::copy("source.txt", "backup/source.txt")?;    // u64 (bytes copied)
```

---

## Directory Operations

```rust
// Ensure directory exists (create if missing)
fs::ensure_dir("storage/logs")?;

// Create (fails if exists)
fs::create_dir("new_dir")?;

// List entries
let entries = fs::list("src/")?;       // Vec<PathBuf>

// List files matching extension
let rs_files = fs::list_files("src", "rs")?;

// Remove empty directory
fs::remove_dir("empty_dir")?;

// Remove directory and all contents
fs::remove_all("temp/")?;
```

### Recursive Operations

```rust
// Copy entire directory tree
fs::copy_all("templates/", "backup/templates/")?;

// Find files recursively by extension
let all_rs = fs::find_files("src", "rs")?;   // Vec<PathBuf>
```

---

## Path Utilities

```rust
// Normalize (resolve . and ..)
let norm = fs::normalize("/a/b/../c/d/./e");
// /a/c/d/e

// Relative path
let rel = fs::relative("/var/www/app/config", "/var/www/app");
// Some("config")

let rel2 = fs::relative("/var/www/app/src", "/var/www/lib");
// Some("../app/src")

// Format location (for error messages)
let loc = fs::format_location("src/main.rs", 42, 10);
// "src/main.rs:42:10"
```

---

## TempDir

Auto-cleaned temporary directory:

```rust
use viontin::prelude::*;

{
    let tmp = TempDir::new("myapp")?;

    let file_path = tmp.path().join("data.txt");
    fs::write(&file_path, "temporary data")?;

    // Use file_path...

    // tmp is dropped here → directory + contents are removed
}
```

| Method | Description |
|--------|-------------|
| `new(prefix)` | Create temp dir with prefix |
| `path()` | Get the temp directory path |

The directory is automatically removed when `TempDir` is dropped.

---

## FileInfo

```rust
use viontin::prelude::*;

let info = fs::info("src/main.rs")?;

println!("Name:      {}", info.name);        // "main"
println!("Extension: {}", info.extension);   // "rs"
println!("Size:      {} bytes", info.size);
println!("Is dir:    {}", info.is_dir);
println!("Modified:  {:?}", info.modified);
```

```rust
pub struct FileInfo {
    pub path: PathBuf,
    pub name: String,
    pub extension: String,
    pub size: u64,
    pub modified: Option<SystemTime>,
    pub is_dir: bool,
}
```

---

## Hash

Compute a content hash of a file:

```rust
let hash = fs::hash("downloads/package.tar.gz")?;
// hex string of DefaultHasher output
```

> **Note:** Uses `std::hash::DefaultHasher` — NOT cryptographically secure. For production, implement proper hashing via a Gem.

---

## Method Reference

| Function | Returns | Description |
|----------|---------|-------------|
| `read(path)` | `Result<String>` | Read file as string |
| `read_bytes(path)` | `Result<Vec<u8>>` | Read file as bytes |
| `write(path, content)` | `Result<()>` | Write file (creates parents) |
| `append(path, content)` | `Result<()>` | Append to file |
| `delete(path)` | `Result<()>` | Delete file |
| `exists(path)` | `bool` | Check existence |
| `size(path)` | `Result<u64>` | File size in bytes |
| `last_modified(path)` | `Result<SystemTime>` | Last modified time |
| `extension(path)` | `Option<String>` | File extension |
| `stem(path)` | `Option<String>` | Filename without extension |
| `ensure_dir(path)` | `Result<()>` | Create dir if missing |
| `create_dir(path)` | `Result<()>` | Create dir (fails if exists) |
| `remove_dir(path)` | `Result<()>` | Remove empty dir |
| `remove_all(path)` | `Result<()>` | Remove dir + contents |
| `list(path)` | `Result<Vec<PathBuf>>` | List directory entries |
| `list_files(path, ext)` | `Result<Vec<PathBuf>>` | List files with extension |
| `copy(src, dst)` | `Result<u64>` | Copy file |
| `copy_all(src, dst)` | `Result<()>` | Copy directory recursively |
| `find_files(dir, ext)` | `Result<Vec<PathBuf>>` | Find files recursively |
| `normalize(path)` | `PathBuf` | Resolve `.` and `..` |
| `relative(target, base)` | `Option<PathBuf>` | Relative path |
| `format_location(path, line, col)` | `String` | Formatted location string |
| `info(path)` | `Result<FileInfo>` | File metadata |
| `hash(path)` | `Result<String>` | Content hash |

---

## Complete Example

```rust
use viontin::prelude::*;

fn backup_logs() -> Result<(), String> {
    let log_dir = "storage/logs";
    let backup_dir = "storage/backups";

    // Ensure backup directory exists
    fs::ensure_dir(backup_dir)?;

    // Find all log files
    let logs = fs::find_files(log_dir, "log")?;

    for log in &logs {
        let filename = log.file_name().unwrap();
        let dest = PathBuf::from(backup_dir).join(filename);

        // Copy with timestamp suffix
        let timestamp = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_secs();

        let backup_path = format!(
            "{}/{}.{}",
            backup_dir,
            fs::stem(&dest).unwrap_or_default(),
            timestamp
        );

        fs::copy(log, &backup_path)?;
        println!("Backed up: {:?}", filename);
    }

    // Write backup manifest
    fs::write(
        format!("{}/manifest.txt", backup_dir),
        format!("Backup completed at {:?}", std::time::SystemTime::now()),
    )?;

    Ok(())
}
```
