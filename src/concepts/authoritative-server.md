# 权威服务端

权威服务端 (Authoritative Server) 是 RustyWarfare 的核心架构原则。

## 核心原则

**所有游戏规则在服务端执行，客户端只负责输入和显示。**

## 为什么需要权威服务端？

### 防止作弊

如果客户端能决定游戏结果：
- 玩家可以修改内存，让单位无敌
- 玩家可以修改资源数量
- 玩家可以修改移动速度

权威服务端架构下：
- 客户端只发送"我想移动到这里"
- 服务端检查是否合法
- 服务端执行移动并告诉客户端结果
- 客户端无法直接修改游戏状态

### 保证一致性

多人游戏中，所有玩家必须看到相同的游戏状态。

权威服务端保证：
- 只有一个真实状态（服务端）
- 所有客户端同步到这个状态
- 不会出现"我这里他死了，他那里还活着"

## 架构对比

### ❌ 点对点架构 (P2P)

```text
Client A ←→ Client B ←→ Client C

问题：
- 每个客户端维护自己的状态
- 网络延迟导致状态不一致
- 容易作弊
```

### ✅ 权威服务端架构

```text
     Server (权威)
      ↙  ↓  ↘
Client A  B  C

优点：
- Server 维护唯一真实状态
- Client 只是"视图"
- 无法作弊
```

## 实现细节

### 服务端职责

```rust
// 服务端决定战斗结果
fn combat_system(
    mut targets: Query<&mut Health>,
    attackers: Query<(&Damage, &Target)>,
) {
    for (damage, target) in attackers.iter() {
        if let Ok(mut health) = targets.get_mut(target.entity) {
            health.current -= damage.value;  // 服务端权威扣血
        }
    }
}
```

### 客户端职责

```rust
// 客户端只能"请求"
fn send_attack_command(client: &mut Client, target_id: u64) {
    client.send(PlayerInput::Attack { 
        entity_id: target_id 
    });
    // 客户端无法直接扣血，只能请求
}
```

## 客户端预测

权威服务端 ≠ 卡顿。通过客户端预测掩盖延迟：

```text
t=0ms:    玩家点击移动
t=0ms:    客户端立即预测 (单位开始移动)
t=0ms:    发送请求到服务端

t=50ms:   服务端收到请求
t=50ms:   服务端验证并执行
t=50ms:   服务端发送结果

t=100ms:  客户端收到权威结果
t=100ms:  对比预测和权威
t=100ms:  如有偏差，平滑校正
```

玩家感觉：**即时响应** (因为 t=0ms 就有预测)

实际情况：**服务端权威** (t=100ms 会校正错误预测)

## 单人模式也是权威服务端

单人模式不是特例，架构完全相同：

```rust
// 单人模式
Godot → Client (预测) → Server (权威) → Client (校正) → Godot

// 多人模式
Godot → Client (预测) → Server (权威) → Client (校正) → Godot
                           ↓
                     其他玩家的 Client
```

区别仅在于：
- 单人：Server 在本地进程
- 多人：Server 在远程

## 权威检查示例

### 移动检查

```rust
fn validate_move(server: &Server, unit_id: u64, target: Position) -> bool {
    let unit = server.get_unit(unit_id)?;
    
    // 检查是否属于该玩家
    if unit.owner != player_id {
        return false;
    }
    
    // 检查距离是否合理
    let distance = unit.position.distance(target);
    if distance > unit.speed * server.tick_duration() {
        return false;
    }
    
    // 检查地形是否可通行
    if !server.map.is_walkable(target) {
        return false;
    }
    
    true
}
```

### 攻击检查

```rust
fn validate_attack(server: &Server, attacker_id: u64, target_id: u64) -> bool {
    let attacker = server.get_unit(attacker_id)?;
    let target = server.get_unit(target_id)?;
    
    // 检查是否是敌人
    if attacker.team == target.team {
        return false;
    }
    
    // 检查距离
    let distance = attacker.position.distance(target.position);
    if distance > attacker.weapon.range {
        return false;
    }
    
    true
}
```

## 优势总结

1. **防作弊**：客户端无法修改游戏状态
2. **一致性**：所有玩家看到相同状态
3. **简化逻辑**：只需实现一套规则（服务端）
4. **可扩展**：容易添加观战、回放等功能

## 下一步

- 了解客户端如何掩盖延迟：[客户端预测](./prediction.md)
- 了解如何同步状态：[网络同步](./network-sync.md)
