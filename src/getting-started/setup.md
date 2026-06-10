# 环境配置

本章介绍如何搭建 RustyWarfare 的开发环境。

## 系统要求

### 必需工具

- **Rust 工具链** (推荐 stable 最新版本)
- **Git** (用于克隆仓库和管理子模块)
- **Godot 4.x** (用于前端开发和测试)

### 可选工具

- **mdBook** (用于构建本文档)
- **Visual Studio Code** 或 **RustRover** (推荐的 IDE)
- **Android SDK** (如需 Android 平台开发)

## 安装 Rust

如果尚未安装 Rust，请访问 [rustup.rs](https://rustup.rs/) 并按照说明安装：

```bash
# Linux / macOS
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Windows
# 下载并运行 rustup-init.exe
```

验证安装：

```bash
rustc --version
cargo --version
```

## 克隆仓库

```bash
git clone <repository-url> rusty_warfare
cd rusty_warfare

# 初始化子模块 (包含 Godot 前端)
git submodule update --init --recursive
```

## 配置 IDE

### Visual Studio Code

推荐安装以下扩展：

- **rust-analyzer** - Rust 语言服务器
- **CodeLLDB** - 调试器
- **Even Better TOML** - TOML 语法支持

### RustRover

RustRover 开箱即用，无需额外配置。

## 验证环境

运行以下命令验证环境配置正确：

```bash
# 检查所有 crate 是否能通过编译
cargo check --workspace

# 运行测试
cargo test --workspace
```

如果所有命令成功执行，说明环境配置完成！

## 下一步

继续阅读 [构建项目](./building.md) 了解如何编译和运行项目。
