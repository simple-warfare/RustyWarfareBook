# 构建项目

本章介绍如何构建 RustyWarfare 的各个组件。

## Workspace 结构

RustyWarfare 使用 Cargo Workspace 组织，包含以下子包：

```text
├── content/         # 内容包加载
├── protocol/        # 网络协议
├── server/          # 服务端
├── client/          # 客户端
├── runtime_core/    # 运行时核心
├── gdextension/     # Godot 接入
├── builder/         # 构建工具
└── game_domain/     # 游戏领域模型
```

## 构建所有组件

```bash
# 构建整个 workspace
cargo build --workspace

# Release 构建 (优化)
cargo build --workspace --release
```

## 构建特定模块

```bash
# 只构建内容包系统
cargo build -p content

# 只构建服务端
cargo build -p server

# 只构建 GDExtension
cargo build -p gdextension
```

## 构建 GDExtension 动态库

GDExtension 是连接 Rust 核心和 Godot 的桥梁。使用 builder 工具自动构建和部署：

```bash
cargo run -p builder
```

builder 会：
1. 编译 gdextension 为动态库
2. 将产物复制到 `launcher/godot/addons/rusty_core/bin/`
3. 生成正确的 .gdextension 配置文件

## 验证构建

```bash
# 运行所有测试
cargo test --workspace

# 运行特定包的测试
cargo test -p content
cargo test -p server

# 代码风格检查
cargo fmt --check

# Clippy 静态分析
cargo clippy --workspace --all-targets
```

## 构建时间优化

如果构建时间过长，可以尝试：

### 使用 sccache

```bash
# 安装 sccache
cargo install sccache

# 配置 Cargo 使用 sccache
export RUSTC_WRAPPER=sccache
```

### 并行编译

```bash
# 设置并行编译任务数
export CARGO_BUILD_JOBS=8
```

### 增量编译 (开发构建)

```bash
# 增量编译默认开启，如需禁用：
export CARGO_INCREMENTAL=0
```

## 常见构建问题

### 问题：链接错误

如果遇到链接器错误，可能是缺少系统依赖。参考 [环境配置](./setup.md) 确保所有工具已安装。

### 问题：子模块未初始化

```bash
git submodule update --init --recursive
```

### 问题：Bevy 编译慢

首次编译 Bevy 可能需要较长时间 (5-10 分钟)，这是正常现象。后续增量编译会快很多。

## 下一步

继续阅读 [运行与调试](./running.md) 了解如何启动项目。
