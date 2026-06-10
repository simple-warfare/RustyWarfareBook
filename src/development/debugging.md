# 调试技巧

## 日志调试

设置日志级别：

```bash
# Windows PowerShell
$env:RUST_LOG="debug"

# Linux / macOS
export RUST_LOG=debug
```

日志级别：
- `error` - 仅错误
- `warn` - 警告及以上
- `info` - 信息及以上
- `debug` - 调试及以上
- `trace` - 所有日志

## 特定模块日志

```bash
export RUST_LOG=server=debug,client=info
```

## 使用调试器

### VS Code

安装 CodeLLDB 扩展，创建 `.vscode/launch.json`。

### RustRover

直接点击运行按钮旁的调试按钮。

## 网络调试

```bash
export RUST_LOG=lightyear=debug
```

查看：
- 网络延迟
- 丢包率
- 预测命中率

## 性能分析

```bash
cargo install flamegraph
cargo flamegraph -p server
```
