> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Scheduling

**Module:** `viontin_framework::schedule`

Cron-based task scheduler for running recurring jobs.

---

## Quick Start

```rust
use viontin::prelude::*;

let mut scheduler = Scheduler::new();
scheduler.add(BackupJob);
scheduler.run_due().unwrap();
```

---

## ScheduledJob Trait

```rust
pub trait ScheduledJob: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn handle(&self) -> Result<(), String>;
    fn expression(&self) -> &str;
}
```

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct BackupJob;

impl ScheduledJob for BackupJob {
    fn name(&self) -> &str { "backup" }

    fn expression(&self) -> &str {
        "0 2 * * *"  // every day at 2:00 AM
    }

    fn handle(&self) -> Result<(), String> {
        println!("Running backup...");
        Ok(())
    }
}
```

---

## Scheduler

```rust
pub struct Scheduler {
    jobs: Vec<Box<dyn ScheduledJob>>,
}
```

### Methods

```rust
let mut scheduler = Scheduler::new();
scheduler.add(BackupJob);

// Run all jobs whose schedule matches now
let executed = scheduler.run_due()?;  // Vec<String> — job names

scheduler.jobs();  // &[Box<dyn ScheduledJob>]
```

---

## Cron Expressions

Standard 5-field cron:

```
┌───────── minute (0-59)
│ ┌───────── hour (0-23)
│ │ ┌───────── day of month (1-31)
│ │ │ ┌───────── month (1-12)
│ │ │ │ ┌───────── weekday (0-6, Sunday=0)
│ │ │ │ │
* * * * *
```

### Features

| Pattern | Example | Description |
|---------|---------|-------------|
| `*` | `* * * * *` | Every minute |
| `N` | `5 * * * *` | At minute 5 |
| `N-M` | `9-17 * * * *` | Range (9 AM to 5 PM) |
| `*/N` | `*/15 * * * *` | Every N minutes |
| `A,B,C` | `0,30 * * * *` | Multiple values |

### Examples

```rust
"*/15 * * * *"  // every 15 minutes
"0 * * * *"     // every hour
"0 9 * * *"     // every day at 9 AM
"0 0 * * 0"     // every Sunday at midnight
"30 4 * * 1-5"  // weekdays at 4:30 AM
```

---

## Complete Example

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct SendDailyReports;

impl ScheduledJob for SendDailyReports {
    fn name(&self) -> &str { "daily_reports" }
    fn expression(&self) -> &str { "0 8 * * *" }

    fn handle(&self) -> Result<(), String> {
        println!("Generating and sending daily reports...");
        Ok(())
    }
}

#[derive(Debug)]
struct CleanupTempFiles;

impl ScheduledJob for CleanupTempFiles {
    fn name(&self) -> &str { "cleanup" }
    fn expression(&self) -> &str { "0 */6 * * *" }

    fn handle(&self) -> Result<(), String> {
        println!("Cleaning temporary files...");
        Ok(())
    }
}

fn main() {
    let mut scheduler = Scheduler::new();
    scheduler.add(SendDailyReports);
    scheduler.add(CleanupTempFiles);

    // In a real app, run this on a loop or cron trigger
    match scheduler.run_due() {
        Ok(executed) => {
            for job in executed {
                println!("Executed: {}", job);
            }
        }
        Err(e) => eprintln!("Scheduler error: {}", e),
    }
}
```
