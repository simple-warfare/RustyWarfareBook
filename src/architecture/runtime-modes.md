# 运行模式

RustyWarfare 支持三种运行模式，所有模式使用相同的游戏规则，只是 server 和 client 的部署位置不同。

## 模式概览

| 模式 | Server 位置 | Client 位置 | 通信方式 | 用途 |
|------|------------|------------|---------|------|
| 单人模式 | 本地 | 本地 | 内存通道 | 单人游戏 |
| 主机模式 | 本地 | 本地+远程 | 本地内存+UDP | 局域网/在线主机 |
| 远程模式 | 远程 | 本地 | UDP | 加入他人房间 |
| 专用服务器 | 独立进程 | 远程 | UDP | 持久化服务器 |

## 单人模式 (Singleplayer)

### 架构

```text
┌──────────────────────────────────┐
│     Godot + gdextension          │
│  ┌────────────────────────────┐  │
│  │     runtime_core           │  │
│  │  ┌──────────┐  ┌─────────┐ │  │
│  │  │  Client  │←→│ Server  │ │  │
│  │  └──────────┘  └─────────┘ │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
        单一进程
```

### 特点

- **低延迟**：无网络开销
- **离线可玩**：不需要网络连接
- **完整预测**：仍然使用预测机制，保持架构一致性
- **调试友好**：所有代码在同一进程

### 数据流

```text
1. Godot 采集输入
2. gdextension → runtime_core
3. client.process_input()
4. client → server (内存通道，CrossbeamIo)
5. server 执行权威逻辑
6. server → client (内存复制)
7. client.reconcile()
8. runtime_core.get_snapshot()
9. gdextension → Godot
```

### 启动代码

```rust
impl Runtime {
    pub fn start_singleplayer(&mut self) -> Result<()> {
        // 加载内容包
        let content_db = self.load_content()?;
        
        // 创建本地 server
        let server = Server::new_local(content_db)?;
        
        // 创建本地 client
        let client = Client::new_local()?;
        
        // 建立内存通道
        let (tx, rx) = crossbeam_channel::unbounded();
        server.connect_channel(rx);
        client.connect_channel(tx);
        
        self.server = Some(server);
        self.client = Some(client);
        self.mode = RuntimeMode::Singleplayer;
        
        Ok(())
    }
}
```

## 主机模式 (Host)

### 架构

```text
本机玩家:
┌────────────────────────────────────┐
│     Godot + gdextension            │
│  ┌──────────────────────────────┐  │
│  │     runtime_core             │  │
│  │  ┌──────────┐  ┌───────────┐ │  │
│  │  │  Client  │←→│  Server   │ │  │
│  │  └──────────┘  └─────┬─────┘ │  │
│  └────────────────────────┼──────┘  │
└────────────────────────────┼────────┘
                             │ UDP
                             ↓
                      ┌──────────────┐
                      │ 远程 Client  │
                      └──────────────┘
```

### 特点

- **混合通信**：本机用内存通道，远程用 UDP
- **主机优势**：本机玩家零延迟
- **灵活性**：可随时开始/停止接受连接

### 数据流

**本机玩家**：
```text
同单人模式 (内存通道)
```

**远程玩家**：
```text
1. 远程 Client 发送输入 → UDP → 本机 Server
2. 本机 Server 执行权威逻辑
3. 本机 Server → UDP → 远程 Client (复制状态)
4. 远程 Client reconcile 并显示
```

### 启动代码

```rust
impl Runtime {
    pub fn start_host(&mut self, port: u16) -> Result<()> {
        let content_db = self.load_content()?;
        
        // 创建网络 server (监听端口)
        let server = Server::new_host(content_db, port)?;
        
        // 创建本地 client
        let client = Client::new_local()?;
        
        // 本地连接用内存通道
        let (tx, rx) = crossbeam_channel::unbounded();
        server.add_local_client(rx);
        client.connect_channel(tx);
        
        // Server 同时监听 UDP 端口
        
        self.server = Some(server);
        self.client = Some(client);
        self.mode = RuntimeMode::Host { port };
        
        Ok(())
    }
}
```

## 远程模式 (Client)

### 架构

```text
本地:
┌────────────────────────────┐
│   Godot + gdextension      │
│  ┌──────────────────────┐  │
│  │   runtime_core       │  │
│  │  ┌──────────┐        │  │
│  │  │  Client  │        │  │
│  │  └─────┬────┘        │  │
│  └────────┼─────────────┘  │
└────────────┼───────────────┘
             │ UDP
             ↓
      ┌─────────────┐
      │远程 Server  │
      └─────────────┘
```

### 特点

- **轻量级**：不运行 server
- **预测 + 插值**：掩盖网络延迟
- **自动校正**：服务器权威，客户端接受校正

### 数据流

```text
1. Godot 采集输入
2. Client 立即本地预测
3. Client → UDP → Server (发送输入)
4. [网络延迟...]
5. Server 执行权威逻辑
6. Server → UDP → Client (复制状态)
7. Client reconcile (对比预测和权威)
8. Client interpolate (平滑过渡)
9. 显示最终状态
```

### 预测与校正

```text
时刻 t=0: 用户点击移动
时刻 t=0: Client 预测 → 单位立即开始移动
时刻 t=0: 发送输入到 Server

时刻 t=50ms: Server 收到输入
时刻 t=50ms: Server 执行权威移动
时刻 t=50ms: Server 返回权威状态

时刻 t=100ms: Client 收到权威状态
时刻 t=100ms: Client 对比预测和权威
时刻 t=100ms: 如有偏差，平滑校正
```

### 启动代码

```rust
impl Runtime {
    pub fn start_client(&mut self, server_addr: &str) -> Result<()> {
        // 仅创建 client
        let client = Client::new_remote(server_addr)?;
        
        self.client = Some(client);
        self.server = None;  // 无本地 server
        self.mode = RuntimeMode::Client {
            server_addr: server_addr.to_string(),
        };
        
        Ok(())
    }
}
```

## 专用服务器 (Dedicated Server)

### 架构

```text
专用服务器 (独立进程):
┌────────────────────┐
│  runtime_core      │
│  ┌──────────────┐  │
│  │   Server     │  │
│  └──────┬───────┘  │
└─────────┼──────────┘
          │ UDP
    ┌─────┴─────┐
    ↓           ↓
┌────────┐  ┌────────┐
│Client 1│  │Client 2│
└────────┘  └────────┘
```

### 特点

- **持久化**：不依赖玩家在线
- **无 GUI**：无 Godot，纯 Rust
- **高性能**：无渲染开销
- **独立部署**：可部署到云服务器

### 组成模块

```text
dedicated_server
    └── runtime_core
        └── server
        └── protocol
        └── content
```

**不包含**：
- ❌ `client`
- ❌ `gdextension`
- ❌ Godot

### 启动代码

```rust
// 独立的 dedicated_server 二进制
fn main() {
    let mut runtime = Runtime::new();
    
    runtime.start_dedicated_server(
        port: 5000,
        max_players: 8,
    )?;
    
    loop {
        runtime.update(TICK_DURATION)?;
        thread::sleep(TICK_DURATION);
    }
}
```

## 模式对比

### 包含的模块

| 模块 | 单人 | 主机 | 远程 | 专用 |
|------|------|------|------|------|
| gdextension | ✅ | ✅ | ✅ | ❌ |
| runtime_core | ✅ | ✅ | ✅ | ✅ |
| client | ✅ | ✅ | ✅ | ❌ |
| server | ✅ | ✅ | ❌ | ✅ |
| protocol | ✅ | ✅ | ✅ | ✅ |
| content | ✅ | ✅ | ❌ | ✅ |

### 延迟特性

| 模式 | 本地延迟 | 网络延迟 | 预测 | 插值 |
|------|---------|---------|------|------|
| 单人 | <1ms | 无 | 有(架构一致) | 无 |
| 主机 | <1ms | 20-100ms | 有 | 有 |
| 远程 | - | 20-100ms | 有 | 有 |
| 专用 | - | 20-100ms | - | - |

## 切换模式

在运行时不支持模式切换。必须：
1. 调用 `runtime.shutdown()`
2. 调用新模式的 `start_*()` 方法

```rust
// 从单人模式切换到主机模式
runtime.shutdown()?;
runtime.start_host(5000)?;
```

## 下一步

了解详细的数据流动：[数据流](./data-flow.md)
