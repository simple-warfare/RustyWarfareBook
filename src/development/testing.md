# 测试指南

## 运行测试

```bash
# 所有测试
cargo test --workspace

# 特定包
cargo test -p content
cargo test -p server

# 特定测试
cargo test test_content_loading
```

## 单元测试

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_example() {
        let result = my_function(42);
        assert!(result.is_ok());
    }
}
```

## 集成测试

```rust
// tests/integration_test.rs
use my_crate::*;

#[test]
fn test_full_flow() {
    let runtime = Runtime::new();
    runtime.initialize(config).unwrap();
    // ...
}
```

## 测试覆盖率

```bash
cargo install cargo-tarpaulin
cargo tarpaulin --workspace
```
