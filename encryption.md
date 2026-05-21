# Encryption

**Module:** `viontin_framework::encryption`  
**Implementation:** `SimpleEncrypter`

> **Warning:** The built-in `SimpleEncrypter` uses XOR-based encryption and is **NOT cryptographically secure**. It is intended for development and testing only. For production, implement `Encrypter` with AES via a Gem.

---

## Encrypter Trait

```rust
pub trait Encrypter: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn encrypt(&self, plaintext: &str) -> String;
    fn decrypt(&self, ciphertext: &str) -> Result<String, String>;
    fn keygen() -> String;
}
```

| Method | Returns | Description |
|--------|---------|-------------|
| `encrypt(plaintext)` | `String` | Encrypt to hex-encoded string |
| `decrypt(ciphertext)` | `Result<String>` | Decrypt from hex-encoded string |
| `keygen()` | `String` | Generate a random encryption key |
| `name()` | `&str` | Algorithm identifier |

---

## SimpleEncrypter

Development-only XOR-based encrypter:

```rust
use viontin::prelude::*;

// Generate a key
let key = SimpleEncrypter::keygen();
// "key_1a2b3c4f_d5e6f7a8b9c0d1e2"

// Create encrypter
let encrypter = SimpleEncrypter::new(&key);

// Encrypt
let secret = "Hello, World!";
let encrypted = encrypter.encrypt(secret);
// Hex-encoded string

// Decrypt
let decrypted = encrypter.decrypt(&encrypted).unwrap();
assert_eq!(secret, decrypted);
```

### Algorithm

1. Prepend a random salt byte
2. XOR each byte with the corresponding key byte
3. Add salt to each XORed byte (wrap-around)
4. Hex-encode the result

```
plaintext → XOR with key → add salt → hex encode
```

### Characteristics

- **Deterministic with same key and salt?** No — each encryption produces different output due to random salt
- **Key size:** Any length (XOR wraps around)
- **Output format:** Hex-encoded string
- **Security:** NOT cryptographically secure — do not use in production

---

## Key Management

```rust
// Generate a new key
let key = SimpleEncrypter::keygen();

// Store in environment variable
// env.set("APP_KEY", &key);

// Load at runtime
let app_key = env("APP_KEY", "");
let encrypter = SimpleEncrypter::new(&app_key);
```

---

## Custom Encrypter

For production, implement `Encrypter` with AES:

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct AesEncrypter {
    key: Vec<u8>,
}

impl AesEncrypter {
    fn new(key: &[u8; 32]) -> Self {
        AesEncrypter { key: key.to_vec() }
    }
}

impl Encrypter for AesEncrypter {
    fn name(&self) -> &str { "aes-256-gcm" }

    fn encrypt(&self, plaintext: &str) -> String {
        // Use aes-gcm crate
        todo!("Implement with aes-gcm crate")
    }

    fn decrypt(&self, ciphertext: &str) -> Result<String, String> {
        // Use aes-gcm crate
        todo!("Implement with aes-gcm crate")
    }

    fn keygen() -> String {
        // Generate 32 random bytes, hex-encoded
        todo!("Implement with rand crate")
    }
}
```

---

## Complete Example

```rust
use viontin::prelude::*;

fn main() {
    // Generate and store key
    let key = SimpleEncrypter::keygen();
    println!("Encryption key: {}", key);

    let encrypter = SimpleEncrypter::new(&key);

    // Encrypt sensitive data
    let sensitive = "SSN: 123-45-6789";
    let encrypted = encrypter.encrypt(sensitive);
    println!("Encrypted: {}", encrypted);

    // Decrypt when needed
    match encrypter.decrypt(&encrypted) {
        Ok(decrypted) => println!("Decrypted: {}", decrypted),
        Err(e) => println!("Decryption failed: {}", e),
    }

    // Wrong key fails
    let wrong_encrypter = SimpleEncrypter::new("wrong-key");
    match wrong_encrypter.decrypt(&encrypted) {
        Ok(_) => println!("Unexpected success with wrong key"),
        Err(e) => println!("Expected error: {}", e),
    }
}
```
