# 网络同步

RustyWarfare 使用 Lightyear 实现客户端-服务端状态同步。

## 核心机制

### 1. 复制 (Replication)

标记需要同步的组件：

```rust
use lightyear::prelude::*;

#[derive(Component, Serialize, Deserialize)]
pub struct Position { x: f32, y: f32 }

// 注册复制
app.replicate::<Position>();
```

服务端的 Position 组件会自动同步到客户端。

### 2. 复制组

```rust
// 服务端
let entity = world.spawn((
    Position { x: 100.0, y: 200.0 },
    Health { current: 100, max: 100 },
    Replicated,  // 标记为需要复制的实体
));
```

### 3. 客户端接收

```rust
// 客户端自动创建镜像实体
fn on_replicate(
    mut commands: Commands,
    query: Query<(Entity, &Position, &Health), Added<Replicated>>,
) {
    for (entity, pos, health) in query.iter() {
        println!("New entity {} at ({}, {})", entity, pos.x, pos.y);
    }
}
```

## 同步策略

### 全量同步 vs 增量同步

**全量同步**：每次发送完整状态
```rust
struct Snapshot {
    entities: Vec<EntityState>,  // 所有实体
}
```

**增量同步**：只发送变化
```rust
struct Delta {
    changed: Vec<(EntityId, Component)>,  // 只有变化的组件
    removed: Vec<EntityId>,
}
```

Lightyear 默认使用**增量同步**。

### 可靠性设置

```rust
#[derive(Component, Serialize, Deserialize)]
#[replicate(reliability = Reliable)]  // 保证送达
pub struct Health { ... }

#[derive(Component, Serialize, Deserialize)]
#[replicate(reliability = Unreliable)]  // 允许丢包
pub struct Velocity { ... }
```

- **Reliable**：重要数据（生命值、位置）
- **Unreliable**：可丢失数据（速度、临时特效）

## Tick 系统

### 固定时间步

```rust
pub const SERVER_TICK_RATE: f64 = 20.0;  // 20 ticks/秒
pub const TICK_DURATION: f64 = 1.0 / 20.0;  // 50ms
```

服务端以固定频率更新：

```rust
fn fixed_update(world: &mut World) {
    // 每 50ms 执行一次
    world.run_schedule(FixedUpdate);
}
```

### Tick 编号

```rust
pub struct Tick(pub u32);

// 服务端
let current_tick = server.tick();  // Tick(100)

// 客户端
let server_tick = client.server_tick();  // Tick(98) (延迟 2 tick)
```

## 兴趣管理 (Interest Management)

只同步玩家"感兴趣"的实体：

```rust
fn update_interest(
    mut clients: Query<&mut ClientInterest>,
    players: Query<&Position, With<Player>>,
    entities: Query<(Entity, &Position)>,
) {
    for (mut interest, player_pos) in clients.iter_mut() {
        interest.clear();
        
        // 只同步玩家视野内的实体
        for (entity, pos) in entities.iter() {
            if player_pos.distance(pos) < VISION_RADIUS {
                interest.add(entity);
            }
        }
    }
}
```

减少带宽：
- 不同步战场另一端的单位
- 迷雾中的单位不同步

## 输入同步

### 客户端发送输入

```rust
#[derive(Serialize, Deserialize)]
pub struct PlayerInput {
    tick: Tick,
    command: Command,
}

// 客户端
client.send_input(PlayerInput {
    tick: client.tick(),
    command: Command::Move { target: pos },
});
```

### 服务端处理输入

```rust
fn process_inputs(
    mut events: EventReader<InputEvent>,
    mut units: Query<&mut Position>,
) {
    for event in events.read() {
        if let Command::Move { target } = event.input.command {
            if let Ok(mut pos) = units.get_mut(event.entity) {
                // 执行移动
                *pos = target;
            }
        }
    }
}
```

### 输入缓冲

服务端缓冲一小段时间的输入，处理乱序：

```rust
struct InputBuffer {
    buffer: BTreeMap<Tick, Vec<PlayerInput>>,
    oldest_tick: Tick,
}

impl InputBuffer {
    fn add(&mut self, input: PlayerInput) {
        self.buffer.entry(input.tick)
            .or_insert_with(Vec::new)
            .push(input);
    }
    
    fn drain_up_to(&mut self, tick: Tick) -> Vec<PlayerInput> {
        self.buffer.range(..=tick)
            .flat_map(|(_, inputs)| inputs.clone())
            .collect()
    }
}
```

## 带宽优化

### 1. 组件优先级

```rust
#[replicate(priority = High)]
pub struct Position { ... }

#[replicate(priority = Low)]
pub struct Cosmetic { ... }
```

带宽不足时，优先发送 High 优先级的组件。

### 2. 更新频率

```rust
#[replicate(send_frequency = 10.0)]  // 每秒 10 次
pub struct Position { ... }

#[replicate(send_frequency = 2.0)]   // 每秒 2 次
pub struct Health { ... }
```

### 3. 压缩

```rust
#[derive(Serialize, Deserialize)]
pub struct CompressedPosition {
    x: i16,  // -32768 到 32767
    y: i16,
}

impl From<Position> for CompressedPosition {
    fn from(pos: Position) -> Self {
        Self {
            x: (pos.x * 10.0) as i16,  // 精度 0.1
            y: (pos.y * 10.0) as i16,
        }
    }
}
```

## 延迟补偿

### 服务端延迟补偿

```rust
fn lag_compensation(
    input: &PlayerInput,
    history: &StateHistory,
) -> ServerState {
    // 回退到客户端发送输入时的状态
    let client_tick = input.tick;
    let state_at_input_time = history.get(client_tick);
    
    // 在该状态下验证输入
    validate_input(input, state_at_input_time)
}
```

### 客户端插值

```rust
fn interpolate_entities(
    query: Query<(&NetworkedEntity, &mut Position)>,
    time: Res<InterpolationTime>,
) {
    for (networked, mut pos) in query.iter_mut() {
        let t0 = networked.snapshot_t0;
        let t1 = networked.snapshot_t1;
        
        let alpha = (time.current - t0) / (t1 - t0);
        *pos = Position::lerp(networked.pos_t0, networked.pos_t1, alpha);
    }
}
```

## 实战示例

### 完整的同步流程

```text
1. 服务端 Tick 100
   - 执行游戏逻辑
   - 标记变化的组件
   
2. Lightyear 收集变化
   - Position 组件改变
   - Health 组件未变
   
3. 打包并发送
   - 只发送 Position
   - 添加 Tick 编号
   
4. 客户端接收 (Tick 98)
   - 解包数据
   - 更新镜像实体
   
5. 客户端插值 (Tick 98 → 100)
   - 平滑过渡
   - 显示到屏幕
```

## 诊断工具

### 网络统计

```rust
let stats = client.network_stats();
println!("RTT: {}ms", stats.rtt);
println!("Packet loss: {:.1}%", stats.packet_loss * 100.0);
println!("Bandwidth: {} KB/s", stats.bandwidth_usage / 1024.0);
```

### 可视化

```rust
fn debug_networking(
    mut gizmos: Gizmos,
    clients: Query<&NetworkStats>,
) {
    for stats in clients.iter() {
        // 绘制延迟图表
        gizmos.line(
            Vec2::ZERO,
            Vec2::new(stats.rtt as f32, 0.0),
            Color::RED,
        );
    }
}
```

## 下一步

- 了解内容包系统：[内容包系统](./content-package.md)
- 了解服务端实现：[Server 模块](../modules/server.md)
