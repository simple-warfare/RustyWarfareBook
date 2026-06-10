# Protocol - 网络协议

`protocol` crate 定义 client 和 server 的共享协议。

## 职责

- 定义网络消息
- 定义 Lightyear 复制组件
- 定义玩家输入命令
- 定义同步常量

## 核心组件

### 复制组件

使用 Lightyear 的 `#[component]` 宏：

```rust
use lightyear::prelude::*;

#[derive(Component, Serialize, Deserialize, Clone)]
pub struct Position {
    pub x: f32,
    pub y: f32,
}

#[derive(Component, Serialize, Deserialize, Clone)]
pub struct Health {
    pub current: i32,
    pub max: i32,
}
```

### 玩家输入

```rust
#[derive(Serialize, Deserialize, Clone)]
pub enum PlayerInput {
    Move { target: Position },
    Attack { entity_id: u64 },
    Build { unit_type: String, position: Position },
    Stop,
}
```

### 网络消息

```rust
#[derive(Serialize, Deserialize)]
pub enum ClientMessage {
    Input(PlayerInput),
    RequestSnapshot,
}

#[derive(Serialize, Deserialize)]
pub enum ServerMessage {
    Snapshot(GameSnapshot),
    EntitySpawned { id: u64, template: String },
}
```

## Lightyear 配置

```rust
pub const TICK_RATE: f64 = 20.0;  // 20 ticks/秒
pub const REPLICATION_INTERVAL: Duration = Duration::from_millis(50);
```

## 为什么独立？

`protocol` 不依赖 `client` 或 `server`，因为：
- Server 和 client 都需要相同的定义
- 防止循环依赖
- 便于版本管理和兼容性检查

## 使用示例

```rust
// Server 侧
use protocol::{Position, Health, PlayerInput};

fn spawn_unit(world: &mut World, template: &UnitTemplate) {
    world.spawn((
        Position { x: 0.0, y: 0.0 },
        Health { current: 100, max: 100 },
    ));
}

// Client 侧
fn send_move_command(client: &mut Client, target: Position) {
    client.send(PlayerInput::Move { target });
}
```

## 下一步

- 了解服务端实现：[Server](./server.md)
- 了解客户端实现：[Client](./client.md)
