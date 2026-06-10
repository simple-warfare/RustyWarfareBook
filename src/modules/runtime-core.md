# Runtime Core - 运行时核心

`runtime_core` crate 负责运行模式编排和生命周期管理。

## 职责

- 管理运行模式 (单人/主机/远程)
- 启动和关闭 server/client
- 协调 server 和 client 的更新
- 前端 Snapshot 投影
- 错误处理和状态管理

## 核心类型

### Runtime

```rust
pub struct Runtime {
    mode: RuntimeMode,
    server: Option<Server>,
    client: Option<Client>,
    content_db: Option<ContentDatabase>,
}
```

### RuntimeMode

```rust
pub enum RuntimeMode {
    None,
    Singleplayer,
    Host { port: u16 },
    Client { server_addr: String },
}
```

### RuntimeConfig

```rust
pub struct RuntimeConfig {
    pub assets_root: PathBuf,
    pub content_package_path: PathBuf,
}
```

## 核心 API

### 初始化

```rust
impl Runtime {
    pub fn new() -> Self {
        Runtime {
            mode: RuntimeMode::None,
            server: None,
            client: None,
            content_db: None,
        }
    }
    
    pub fn initialize(&mut self, config: RuntimeConfig) -> Result<()> {
        // 加载内容包
        let loader = ContentLoader::new();
        self.content_db = Some(loader.load_package(&config.content_package_path)?);
        Ok(())
    }
}
```

### 启动模式

```rust
impl Runtime {
    pub fn start_singleplayer(&mut self) -> Result<()> {
        let content_db = self.content_db.as_ref()
            .ok_or(RuntimeError::NotInitialized)?;
        
        // 创建本地 server
        let server = Server::new_local(content_db.clone())?;
        
        // 创建本地 client
        let client = Client::new_local()?;
        
        // 建立内存通道
        self.connect_local(&mut server, &mut client)?;
        
        self.server = Some(server);
        self.client = Some(client);
        self.mode = RuntimeMode::Singleplayer;
        
        Ok(())
    }
    
    pub fn start_host(&mut self, port: u16) -> Result<()> {
        // 类似单人模式，但 server 监听网络端口
        // ...
    }
    
    pub fn start_client(&mut self, server_addr: String) -> Result<()> {
        // 仅创建 client，连接远程 server
        // ...
    }
}
```

### 更新循环

```rust
impl Runtime {
    pub fn update(&mut self, delta_seconds: f64) -> Result<()> {
        match self.mode {
            RuntimeMode::Singleplayer => {
                // 更新 server
                if let Some(server) = &mut self.server {
                    server.update(delta_seconds)?;
                }
                
                // 更新 client
                if let Some(client) = &mut self.client {
                    client.update(delta_seconds)?;
                }
            }
            RuntimeMode::Host { .. } => {
                // 同上
            }
            RuntimeMode::Client { .. } => {
                // 仅更新 client
                if let Some(client) = &mut self.client {
                    client.update(delta_seconds)?;
                }
            }
            RuntimeMode::None => {
                return Err(RuntimeError::NotStarted);
            }
        }
        
        Ok(())
    }
}
```

### 前端 Snapshot

```rust
impl Runtime {
    pub fn get_frontend_snapshot(&self) -> Option<FrontendSnapshot> {
        self.client.as_ref().map(|c| c.get_frontend_snapshot())
    }
}
```

### 关闭

```rust
impl Runtime {
    pub fn shutdown(&mut self) -> Result<()> {
        if let Some(mut server) = self.server.take() {
            server.shutdown()?;
        }
        
        if let Some(mut client) = self.client.take() {
            client.shutdown()?;
        }
        
        self.mode = RuntimeMode::None;
        
        Ok(())
    }
}
```

## 错误处理

```rust
#[derive(Debug, thiserror::Error)]
pub enum RuntimeError {
    #[error("Runtime not initialized")]
    NotInitialized,
    
    #[error("Runtime not started")]
    NotStarted,
    
    #[error("Server error: {0}")]
    ServerError(#[from] ServerError),
    
    #[error("Client error: {0}")]
    ClientError(#[from] ClientError),
    
    #[error("Content error: {0}")]
    ContentError(#[from] ContentError),
}
```

## 为什么需要 runtime_core？

如果没有 `runtime_core`，`gdextension` 会直接管理：
- Server 和 client 的生命周期
- 网络配置
- 内容加载
- 模式切换

这会让 `gdextension` 变得臃肿，且难以测试。

`runtime_core` 提供**纯 Rust API**，使得：
- 可以编写不依赖 Godot 的测试
- 未来可支持其他前端 (如纯 Bevy 渲染)
- GDExtension 层只做类型转换

## 使用示例

```rust
use runtime_core::{Runtime, RuntimeConfig};

let mut runtime = Runtime::new();

// 初始化
runtime.initialize(RuntimeConfig {
    assets_root: PathBuf::from("assets"),
    content_package_path: PathBuf::from("assets/official"),
})?;

// 启动单人模式
runtime.start_singleplayer()?;

// 游戏循环
loop {
    runtime.update(0.016)?;  // 60 FPS
    
    if let Some(snapshot) = runtime.get_frontend_snapshot() {
        // 渲染 snapshot
    }
}

// 关闭
runtime.shutdown()?;
```

## 下一步

- 了解 Godot 接入：[GDExtension](./gdextension.md)
- 了解构建流程：[Builder](./builder.md)
