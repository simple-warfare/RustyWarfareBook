# 客户端预测

客户端预测是掩盖网络延迟、提供即时反馈的关键技术。

## 问题：网络延迟

典型网络延迟：50-100ms

```text
没有预测的体验：
t=0ms:    玩家点击移动
t=0ms:    发送到服务器
t=50ms:   服务器收到并处理
t=100ms:  客户端收到结果，单位才开始移动

玩家感觉：卡顿！点击后要等 100ms 才有反应
```

## 解决方案：立即预测

```text
有预测的体验：
t=0ms:    玩家点击移动
t=0ms:    客户端立即预测 → 单位立即移动
t=0ms:    同时发送到服务器
t=50ms:   服务器处理
t=100ms:  客户端收到权威结果 → 校正（如有偏差）

玩家感觉：即时响应！
```

## 预测流程

### 1. 输入阶段

```rust
fn on_click(client: &mut Client, target: Position) {
    let input = PlayerInput::Move { target };
    
    // 1. 保存到历史
    client.input_history.push(input.clone());
    
    // 2. 立即本地模拟（预测）
    client.predict_move(target);
    
    // 3. 发送到服务器
    client.send_to_server(input);
}
```

### 2. 预测阶段

```rust
fn predict_move(&mut self, target: Position) {
    // 获取选中单位
    for entity in self.selected_units.iter() {
        if let Some(unit) = self.predicted_state.get_mut(entity) {
            // 立即更新预测状态
            unit.target = Some(target);
            unit.is_moving = true;
        }
    }
}
```

### 3. 校正阶段

```rust
fn reconcile(&mut self, server_snapshot: ServerSnapshot) {
    for entity in server_snapshot.entities {
        let predicted = self.predicted_state.get(entity.id);
        let authoritative = entity.position;
        
        let error = predicted.distance(authoritative);
        
        if error < 0.1 {
            // 预测准确！
            self.accept_prediction(entity.id);
        } else if error < 5.0 {
            // 小偏差，平滑校正
            self.smooth_correct(entity.id, authoritative);
        } else {
            // 大偏差，直接跳转
            self.hard_correct(entity.id, authoritative);
        }
    }
}
```

## 预测什么？

### ✅ 应该预测

- **移动**：路径确定，容易预测
- **转向**：即时反馈重要
- **动画状态**：行走、攻击动画

### ❌ 不应该预测

- **战斗结果**：依赖双方状态，难以准确预测
- **资源生产**：服务端权威
- **单位死亡**：必须等待服务端确认

## 预测准确性

### 准确预测示例

```text
玩家：移动到 (100, 200)
预测：单位沿直线移动到 (100, 200)
服务端：确认，单位确实到达 (100, 200)
结果：预测完全准确，无需校正 ✅
```

### 需要校正的情况

```text
玩家：移动到 (100, 200)
预测：单位沿直线移动
服务端：路径被敌人阻挡，单位停在 (80, 180)
结果：预测偏差 20 像素，平滑校正 ⚠️
```

### 严重错误的情况

```text
玩家：攻击单位 A
预测：单位 A 立即死亡
服务端：单位 A 存活（护盾吸收伤害）
结果：预测完全错误，硬校正 ❌
```

## 校正策略

### 平滑校正 (Smooth Correction)

小误差用插值平滑过渡：

```rust
fn smooth_correct(&mut self, entity_id: u64, target: Position) {
    let current = self.get_position(entity_id);
    let correction_speed = 5.0;  // 每秒校正 5 单位
    
    self.correction_targets.insert(entity_id, CorrectionTarget {
        from: current,
        to: target,
        speed: correction_speed,
    });
}

fn update_corrections(&mut self, delta: f32) {
    for (entity_id, correction) in self.correction_targets.iter_mut() {
        let current = self.get_position(entity_id);
        let direction = (correction.to - current).normalize();
        let step = direction * correction.speed * delta;
        
        let new_pos = current + step;
        self.set_position(entity_id, new_pos);
        
        if current.distance(correction.to) < 0.5 {
            // 校正完成
            self.correction_targets.remove(entity_id);
        }
    }
}
```

### 硬校正 (Hard Correction)

严重错误直接跳转：

```rust
fn hard_correct(&mut self, entity_id: u64, authoritative: Position) {
    self.set_position(entity_id, authoritative);
    // 可能播放"传送"特效
}
```

## 输入历史

保存输入历史用于回放：

```rust
pub struct InputHistory {
    inputs: VecDeque<(Tick, PlayerInput)>,
    last_acked: Tick,
}

impl InputHistory {
    pub fn add(&mut self, tick: Tick, input: PlayerInput) {
        self.inputs.push_back((tick, input));
    }
    
    pub fn acknowledge(&mut self, server_tick: Tick) {
        // 服务端确认到某个 tick，清理旧输入
        self.inputs.retain(|(t, _)| *t > server_tick);
        self.last_acked = server_tick;
    }
    
    pub fn replay_from(&self, tick: Tick) -> Vec<PlayerInput> {
        // 回放从某个 tick 之后的所有输入
        self.inputs.iter()
            .filter(|(t, _)| *t > tick)
            .map(|(_, input)| input.clone())
            .collect()
    }
}
```

## 预测 + 回滚

当服务端状态与预测差异较大时，需要回滚：

```text
1. 客户端预测了 tick 100-110 的输入
2. 服务端返回 tick 105 的权威状态
3. 客户端发现 tick 105 的预测错误
4. 回滚到 tick 105
5. 重新应用 tick 106-110 的输入（基于正确的 105 状态）
```

```rust
fn rollback_and_replay(&mut self, server_tick: Tick, server_state: ServerSnapshot) {
    // 1. 恢复到服务器 tick
    self.state = server_state;
    
    // 2. 获取之后的所有输入
    let inputs_to_replay = self.input_history.replay_from(server_tick);
    
    // 3. 重新模拟
    for input in inputs_to_replay {
        self.simulate_input(input);
    }
}
```

## 优化技巧

### 1. 预测置信度

```rust
pub enum PredictionConfidence {
    High,    // 移动等确定性操作
    Medium,  // 受其他单位影响的操作
    Low,     // 战斗结果等不确定操作
}
```

低置信度的预测可以：
- 延迟显示
- 显示"预测中"标记
- 更快校正

### 2. 部分预测

```rust
// 只预测位置，不预测战斗结果
fn predict_move(&mut self, input: PlayerInput::Move) {
    self.update_position(input.target);
    // 不预测沿途的战斗
}
```

### 3. 客户端插值

对于非本地玩家，使用插值而非预测：

```rust
fn interpolate_remote_players(&mut self, delta: f32) {
    for entity in self.remote_entities.iter() {
        let last = entity.last_server_pos;
        let next = entity.next_server_pos;
        let alpha = entity.interpolation_progress;
        
        let pos = last.lerp(next, alpha);
        self.set_position(entity.id, pos);
        
        entity.interpolation_progress += delta / INTERPOLATION_DELAY;
    }
}
```

## 总结

| 方面 | 无预测 | 有预测 |
|------|--------|--------|
| 延迟感 | 100ms 延迟 | 即时响应 |
| 实现复杂度 | 简单 | 复杂 |
| 需要校正 | 不需要 | 需要 |
| 用户体验 | 卡顿 | 流畅 |

## 下一步

- 了解网络同步机制：[网络同步](./network-sync.md)
- 了解客户端实现：[Client 模块](../modules/client.md)
