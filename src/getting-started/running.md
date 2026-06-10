# 运行与调试

本章介绍如何运行和调试 RustyWarfare。

## 运行 Godot 客户端

RustyWarfare 的主要运行方式是通过 Godot 前端：

1. **构建 GDExtension**：

```bash
cargo run -p builder
```

2. **打开 Godot 项目**：

使用 Godot 4.x 打开 `launcher/godot/` 目录

3. **运行场景**：

在 Godot 编辑器中按 F5 或点击播放按钮

## 运行模式

RustyWarfare 支持三种运行模式：

### 单人模式

在 Godot 中选择"单人游戏"，本地同时启动 server 和 client。

### 主机模式

1. 选择"创建房间"
2. 系统启动本地 server 并自动连接
3. 其他玩家可通过 IP 加入

### 客户端模式

1. 选择"加入房间"
2. 输入服务器地址
3. 连接到远程 server

## 调试技巧

### 启用日志

项目使用 `tracing` 进行日志记录。在启动前设置环境变量：

```bash
# Windows (PowerShell)
$env:RUST_LOG="debug"

# Linux / macOS
export RUST_LOG=debug
```

日志级别：
- `error` - 仅错误
- `warn` - 警告及以上
- `info` - 信息及以上 (推荐)
- `debug` - 调试及以上
- `trace` - 所有日志 (非常详细)

### 调试特定模块

```bash
# 只显示 server 模块的调试日志
export RUST_LOG=server=debug

# 多个模块
export RUST_LOG=server=debug,client=info,runtime_core=trace
```

### 使用调试器

#### VS Code

创建 `.vscode/launch.json`：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug gdextension",
            "cargo": {
                "args": ["build", "-p", "gdextension"]
            },
            "args": [],
            "cwd": "${workspaceFolder}"
        }
    ]
}
```

#### RustRover

右键点击 `gdextension/src/lib.rs` -> Debug 'gdextension'

### Godot 控制台输出

GDExtension 的 Rust 日志会输出到 Godot 控制台。在 Godot 编辑器底部查看输出面板。

### 网络调试

启用 Lightyear 的诊断功能：

```bash
export RUST_LOG=lightyear=debug
```

这会显示：
- 网络延迟
- 丢包率
- 预测命中率
- 回滚次数

## 性能分析

### 使用 `cargo flamegraph`

```bash
# 安装
cargo install flamegraph

# 生成火焰图
cargo flamegraph -p server
```

### 使用 `perf` (Linux)

```bash
perf record --call-graph dwarf cargo run -p server --release
perf report
```

## 常见运行问题

### GDExtension 未加载

**症状**：Godot 报告找不到 RustyCore 类

**解决**：
1. 确认 `cargo run -p builder` 执行成功
2. 检查 `launcher/godot/addons/rusty_core/bin/` 下是否有动态库
3. 重启 Godot 编辑器

### 连接超时

**症状**：客户端无法连接到服务器

**解决**：
1. 检查防火墙设置
2. 确认服务器端口已开放 (默认 5000)
3. 验证 IP 地址正确

### 内容包加载失败

**症状**：启动时报告内容包错误

**解决**：
1. 检查 `launcher/godot/assets/content_packages/official/manifest.toml` 存在
2. 验证内容包格式正确
3. 查看详细错误日志

## 下一步

了解项目架构，请阅读 [总体架构](../architecture/overview.md)。
