# Rust QA Reference

## Running Tests

```bash
# All tests in workspace
cargo test --workspace

# Specific crate
cargo test -p my-crate

# Specific test by name (substring match)
cargo test my_function_name

# Include ignored tests (integration / slow)
cargo test --workspace -- --include-ignored

# Show stdout even on pass
cargo test -- --nocapture

# Run tests single-threaded (useful for DB tests)
cargo test -- --test-threads=1
```

## Linting & Formatting

```bash
# Check formatting (CI-safe, no writes)
cargo fmt --check

# Auto-fix formatting
cargo fmt

# Lint with all warnings as errors
cargo clippy --workspace -- -D warnings

# Lint specific crate
cargo clippy -p my-crate -- -D warnings

# Lint including tests
cargo clippy --workspace --tests -- -D warnings
```

## Writing Unit Tests — Rust Conventions

Unit tests live in the same file as the code under test:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_happy_path() {
        let result = my_function(valid_input);
        assert_eq!(result, expected);
    }

    #[test]
    fn test_error_case() {
        let result = my_function(invalid_input);
        assert!(result.is_err());
    }

    #[test]
    #[should_panic(expected = "some message")]
    fn test_panics() {
        my_function_that_panics();
    }
}
```

### Async tests (Tokio)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_async_function() {
        let result = my_async_fn().await;
        assert!(result.is_ok());
    }
}
```

### Using rstest for parameterized tests

```rust
use rstest::rstest;

#[rstest]
#[case(1, 2, 3)]
#[case(0, 0, 0)]
#[case(-1, 1, 0)]
fn test_add(#[case] a: i32, #[case] b: i32, #[case] expected: i32) {
    assert_eq!(add(a, b), expected);
}
```

## Integration Tests

Integration tests live in `tests/` at the crate root (outside `src/`):

```
my-crate/
├── src/
│   └── lib.rs
└── tests/
    └── integration_test.rs
```

```rust
// tests/integration_test.rs
use my_crate::SomePublicType;

#[test]
fn test_full_flow() {
    // tests the public API
}
```

Mark slow/expensive integration tests with `#[ignore]` and run them explicitly:
```bash
cargo test -- --ignored
```

## Common Failure Patterns

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `cannot borrow as mutable` in test | Forgot `mut` on test variable | Add `mut` |
| Async test hangs | Missing `#[tokio::test]` | Add the attribute |
| DB test fails intermittently | Tests sharing state | Use `--test-threads=1` or isolate with transactions |
| Clippy: `needless_return` | Explicit `return` at end of fn | Remove `return` keyword |
| Clippy: `unwrap_used` | `.unwrap()` in non-test code | Use `?` or proper error handling |
| Doc test fails | Example in rustdoc is outdated | Update the `///` example |

## Coverage (optional)

```bash
# Install cargo-tarpaulin (Linux only)
cargo install cargo-tarpaulin

cargo tarpaulin --workspace --out Html
```

Or use `cargo-llvm-cov`:
```bash
cargo install cargo-llvm-cov
cargo llvm-cov --workspace --html
```
