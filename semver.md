# Semantic Versioning

**Module:** `viontin_framework::semver`

Version parsing, constraint matching, and compatibility checking — for gem dependencies and package management.

---

## Version

```rust
pub struct Version {
    pub major: u64,
    pub minor: u64,
    pub patch: u64,
    pub pre: String,
    pub build: String,
}
```

```rust
use viontin::prelude::*;

let v = Version::parse("1.2.3-alpha+build1").unwrap();
v.major;   // 1
v.minor;   // 2
v.patch;   // 3
v.pre;     // "alpha"
v.build;   // "build1"

// Display
println!("{}", v);  // "1.2.3-alpha+build1"
```

---

## VersionReq

Constraint matching for version requirements:

```rust
use viontin::prelude::*;

let req = VersionReq::parse("^1.0.0").unwrap();
let version = Version::parse("1.5.2").unwrap();

assert!(req.matches(&version));
```

### Constraint Syntax

| Pattern | Example | Matches |
|---------|---------|---------|
| `^` (caret) | `^1.0.0` | `>=1.0.0` and `<2.0.0` |
| `~` (tilde) | `~1.2.0` | `>=1.2.0` and `<1.3.0` |
| `>=` | `>=2.0` | `>=2.0.0` |
| `>` | `>1.0` | `>1.0.0` |
| `<` | `<2.0` | `<2.0.0` |
| `=` | `=1.0.0` | Exactly `1.0.0` |
| `*` | `1.*` | `1.x.x` |
| OR | `^1.0 \|\| ^2.0` | `>=1.0.0` and `<2.0.0` OR `>=2.0.0` and `<3.0.0` |

```rust
VersionReq::parse(">=1.0, <2.0").unwrap().matches(&v);  // true
VersionReq::parse("^0.1.0").unwrap().matches(&v);        // matches 0.1.x only
VersionReq::parse("~1.2").unwrap().matches(&v);          // matches >=1.2, <1.3
```

---

## Meta & Compatibility

```rust
pub struct Meta {
    pub version: Version,
    pub compatibility: Vec<Compatibility>,
}

pub enum Compatibility {
    Compatible,
    Breaking,
}
```

Used by the gem system to validate gem compatibility.

---

## Complete Example

```rust
use viontin::prelude::*;

fn check_gem_compatibility(gem_version: &str, constraint: &str) -> bool {
    let version = Version::parse(gem_version).unwrap();
    let req = VersionReq::parse(constraint).unwrap();
    req.matches(&version)
}

assert!(check_gem_compatibility("1.2.3", "^1.0.0"));
assert!(check_gem_compatibility("2.0.0", "^1.0.0") == false);
assert!(check_gem_compatibility("1.5.0", "~1.2.0") == false);
assert!(check_gem_compatibility("1.2.5", "~1.2.0"));
```
