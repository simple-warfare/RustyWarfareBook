# 代码规范

## Rust 代码风格

使用 `rustfmt` 和 `clippy`：

```bash
cargo fmt
cargo clippy --workspace
```

## 命名规范

- **类型**：`PascalCase`
- **函数/变量**：`snake_case`
- **常量**：`SCREAMING_SNAKE_CASE`
- **生命周期**：`'a`, `'b`

## 错误处理

使用 `Result` 和 `thiserror`：

```rust
#[derive(Debug, thiserror::Error)]
pub enum MyError {
    #[error("Invalid input: {0}")]
    InvalidInput(String),
}
```

## 文档注释

```rust
/// 简要描述
///
/// # Arguments
/// * `x` - 参数说明
///
/// # Returns
/// 返回值说明
pub fn my_function(x: i32) -> Result<()> {
    // ...
}
```
