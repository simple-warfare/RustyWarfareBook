# Builder - 构建工具

`builder` crate 负责跨平台编译和部署 GDExtension。

## 职责

- 编译 gdextension 为动态库
- 复制产物到 Godot 项目
- 生成 .gdextension 配置文件
- 处理跨平台路径

## 使用方法

```bash
cargo run -p builder
```

## 构建流程

```text
1. 检测当前平台
2. 编译 gdextension (release)
3. 复制动态库到目标目录
4. 生成 .gdextension 文件
```

## 实现示例

```rust
fn main() -> Result<()> {
    let target_dir = detect_platform_target()?;
    
    // 编译 gdextension
    println!("Building gdextension...");
    let status = Command::new("cargo")
        .args(&["build", "-p", "gdextension", "--release"])
        .status()?;
    
    if !status.success() {
        return Err("Build failed".into());
    }
    
    // 复制产物
    let lib_name = get_lib_name();
    let src = format!("target/release/{}", lib_name);
    let dst = format!("launcher/godot/addons/rusty_core/bin/{}/{}", 
                      target_dir, lib_name);
    
    fs::copy(&src, &dst)?;
    println!("Copied {} -> {}", src, dst);
    
    // 生成配置
    generate_gdextension_file()?;
    
    Ok(())
}

fn get_lib_name() -> &'static str {
    if cfg!(target_os = "windows") {
        "gdextension.dll"
    } else if cfg!(target_os = "macos") {
        "libgdextension.dylib"
    } else {
        "libgdextension.so"
    }
}
```

## 产物结构

```text
launcher/godot/addons/rusty_core/
  rusty_core.gdextension    # 配置文件
  bin/
    windows/
      libgdextension.dll
    linux/
      libgdextension.so
    macos/
      libgdextension.dylib
    android/
      arm64-v8a/
        libgdextension.so
```

## .gdextension 配置

```ini
[configuration]
entry_symbol = "gdext_rust_init"
compatibility_minimum = 4.2

[libraries]
windows.debug.x86_64 = "res://addons/rusty_core/bin/windows/libgdextension.dll"
windows.release.x86_64 = "res://addons/rusty_core/bin/windows/libgdextension.dll"
linux.debug.x86_64 = "res://addons/rusty_core/bin/linux/libgdextension.so"
linux.release.x86_64 = "res://addons/rusty_core/bin/linux/libgdextension.so"
macos.debug = "res://addons/rusty_core/bin/macos/libgdextension.dylib"
macos.release = "res://addons/rusty_core/bin/macos/libgdextension.dylib"
```

## 跨平台构建

### Windows → Android

```bash
# 安装目标
rustup target add aarch64-linux-android

# 配置 NDK 路径
export ANDROID_NDK_HOME=/path/to/ndk

# 构建
cargo build -p gdextension --target aarch64-linux-android --release
```

### 交叉编译

```bash
# Linux → Windows
cargo build -p gdextension --target x86_64-pc-windows-gnu --release

# macOS → iOS
cargo build -p gdextension --target aarch64-apple-ios --release
```

## 下一步

- 了解开发流程：[贡献约定](../development/contributing.md)
- 了解测试：[测试指南](../development/testing.md)
