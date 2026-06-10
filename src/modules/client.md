# Client - 客户端核心

`client` crate 负责客户端状态管理和预测。

## 职责

- 接收玩家输入
- 本地预测 (立即反馈)
- 接收服务器权威状态
- Reconciliation (校正预测)
- 插值 (平滑显示)
- 生成前端 Snapshot

## 预测机制

### 为什么需要预测？

网络延迟 (50-100ms) 会导致操作感觉"卡顿"。预测让玩家感觉操作是即时的。

```text
时刻 t=0:    玩家点击移动
时刻 t=0:    Client 立即预测 → 单位开始移动 (预测)
时刻 t=0:    发送输入到 Server

时刻 t=50ms: Server 收到输入
时刻 t=50ms: Server 执行权威移动
时刻 t=50ms: Server 发送权威状态

时刻 t=100ms: Client 收到权威状态
时刻 t=100ms: Client 校正 (如有偏差)
```

### 预测实现

```rust
impl Client {
    pub fn predict_input(&mut self, input: PlayerInput) {
        // 1. 保存输入到历史
        self.input_history.push(input.clone());
        
        // 2. 立即在本地模拟
        match input {
            PlayerInput::Move { target } => {
                self.predict_move(target);
            }
            PlayerInput::Attack { entity_id } => {
                self.predict_attack(entity_id);
            }
            _ => {}
        }
        
        // 3. 发送到服务器
        self.send_to_server(input);
    }
}
```

### Reconciliation (校正)

```rust
impl Client {
    pub fn reconcile(&mut self, server_state: ServerSnapshot) {
        for entity in server_state.entities {
            // 获取本地预测的位置
            let predicted_pos = self.get_predicted_position(entity.id);
            
            // 获取服务器权威位置
            let authoritative_pos = entity.position;
            
            // 计算误差
            let error = predicted_pos.distance(authoritative_pos);
            
            if error > RECONCILE_THRESHOLD {
                // 误差过大，平滑校正
                self.smooth_correct(entity.id, authoritative_pos);
            } else {
                // 误差可接受，接受预测
                self.accept_prediction(entity.id);
            }
        }
    }
}
```

## 插值 (Interpolation)

对于非本地玩家控制的实体，使用插值平滑显示：

```rust
fn interpolate_position(
    current: Position,
    target: Position,
    alpha: f32,  // 0.0 到 1.0
) -> Position {
    Position {
        x: current.x + (target.x - current.x) * alpha,
        y: current.y + (target.y - current.y) * alpha,
    }
}
```

## 前端 Snapshot

Client 生成前端可消费的数据：

```rust
pub struct FrontendSnapshot {
    pub units: Vec<FrontendUnit>,
    pub resources: HashMap<u64, PlayerResources>,
    pub game_time: f64,
}

pub struct FrontendUnit {
    pub id: u64,
    pub position: (f32, f32),
    pub health: (i32, i32),  // (current, max)
    pub unit_type: String,
    pub team_id: u8,
}

impl Client {
    pub fn get_frontend_snapshot(&self) -> FrontendSnapshot {
        // 将 ECS 组件转换为前端格式
        FrontendSnapshot {
            units: self.extract_units(),
            resources: self.extract_resources(),
            game_time: self.game_time,
        }
    }
}
```

## 输入缓冲

```rust
pub struct InputBuffer {
    inputs: VecDeque<(Tick, PlayerInput)>,
    last_acknowledged: Tick,
}

impl InputBuffer {
    pub fn add(&mut self, tick: Tick, input: PlayerInput) {
        self.inputs.push_back((tick, input));
    }
    
    pub fn acknowledge(&mut self, tick: Tick) {
        // 服务器确认收到，移除旧输入
        self.inputs.retain(|(t, _)| *t > tick);
        self.last_acknowledged = tick;
    }
}
```

## 使用示例

```rust
use client::Client;
use protocol::PlayerInput;

let mut client = Client::new("127.0.0.1:5000")?;

// 玩家输入
let input = PlayerInput::Move { 
    target: Position { x: 100.0, y: 200.0 } 
};

// 预测并发送
client.predict_input(input);

// 每帧更新
client.update(delta_time)?;

// 获取显示数据
let snapshot = client.get_frontend_snapshot();
```

## 下一步

- 了解运行时编排：[Runtime Core](./runtime-core.md)
- 了解 Godot 接入：[GDExtension](./gdextension.md)
