# ECS架构

RustyWarfare 使用 Bevy ECS (Entity Component System) 管理游戏状态。

## 什么是 ECS？

ECS 是一种数据驱动的架构模式，将游戏对象拆分为：

- **Entity (实体)**：唯一 ID，代表游戏中的"事物"
- **Component (组件)**：纯数据，描述实体的"属性"
- **System (系统)**：逻辑，处理特定组件的"行为"

## 核心概念

### Entity (实体)

```rust
// 实体只是一个 ID
let tank_entity = world.spawn_empty().id();  // Entity(123)
```

### Component (组件)

```rust
#[derive(Component)]
pub struct Position {
    pub x: f32,
    pub y: f32,
}

#[derive(Component)]
pub struct Health {
    pub current: i32,
    pub max: i32,
}

#[derive(Component)]
pub struct Velocity {
    pub x: f32,
    pub y: f32,
}
```

### 组合实体

```rust
// 创建一个坦克
world.spawn((
    Position { x: 100.0, y: 200.0 },
    Health { current: 100, max: 100 },
    Velocity { x: 0.0, y: 0.0 },
    Tank,  // 标记组件
));
```

### System (系统)

```rust
fn movement_system(
    time: Res<Time>,
    mut query: Query<(&mut Position, &Velocity)>,
) {
    for (mut pos, vel) in query.iter_mut() {
        pos.x += vel.x * time.delta_seconds();
        pos.y += vel.y * time.delta_seconds();
    }
}
```

## 为什么使用 ECS？

### 传统 OOP 的问题

```rust
// OOP: 继承层级复杂
class Unit { ... }
class LandUnit extends Unit { ... }
class Tank extends LandUnit { ... }
class HoverTank extends Tank { ... }  // 会飞的坦克怎么办？
```

### ECS 的灵活性

```rust
// 坦克
world.spawn((Position, Health, LandMovement, Turret));

// 会飞的坦克？加个组件即可
world.spawn((Position, Health, AirMovement, Turret));

// 两栖坦克？
world.spawn((Position, Health, LandMovement, WaterMovement, Turret));
```

### 性能优势

ECS 将相同类型的组件存储在连续内存：

```text
传统 OOP:
Tank1 [Position, Health, ...] → 内存位置 A
Tank2 [Position, Health, ...] → 内存位置 B (可能很远)
Tank3 [Position, Health, ...] → 内存位置 C (可能很远)

ECS:
所有 Position: [Pos1, Pos2, Pos3, ...] → 连续内存
所有 Health:   [HP1, HP2, HP3, ...]   → 连续内存
```

CPU 缓存友好 = 更快！

## RustyWarfare 中的应用

### 单位组件

```rust
// 所有单位共有
Position
Health
Team

// 可移动单位
Velocity
MovementType (Land/Air/Water)

// 战斗单位
Weapon
Target

// 建筑
Building
ProductionQueue
```

### 系统调度

```rust
fn configure_systems(app: &mut App) {
    app
        .add_systems(Update, (
            movement_system,
            combat_system,
            production_system,
            resource_system,
        ).chain());  // 按顺序执行
}
```

## Query 查询

### 基础查询

```rust
// 查询所有有 Position 和 Health 的实体
fn display_system(
    query: Query<(&Position, &Health)>,
) {
    for (pos, health) in query.iter() {
        println!("Entity at ({}, {}) has {} HP", 
                 pos.x, pos.y, health.current);
    }
}
```

### 可变查询

```rust
// 修改组件
fn damage_system(
    mut query: Query<&mut Health>,
) {
    for mut health in query.iter_mut() {
        health.current -= 10;
    }
}
```

### 过滤查询

```rust
// 只查询敌方单位
fn target_enemy_system(
    my_team: Res<MyTeam>,
    query: Query<(Entity, &Position), With<Enemy>>,
) {
    for (entity, pos) in query.iter() {
        // 只处理敌人
    }
}
```

## Commands 延迟操作

```rust
fn spawn_projectile_system(
    mut commands: Commands,
    query: Query<(&Position, &Weapon)>,
) {
    for (pos, weapon) in query.iter() {
        // 延迟生成，不立即执行
        commands.spawn((
            Position { x: pos.x, y: pos.y },
            Projectile,
            Velocity { x: 10.0, y: 0.0 },
        ));
    }
}
// 系统结束后才真正生成
```

## Resources 全局数据

```rust
#[derive(Resource)]
pub struct GameTime {
    pub elapsed: f64,
}

fn time_system(
    time: Res<Time>,
    mut game_time: ResMut<GameTime>,
) {
    game_time.elapsed += time.delta_seconds();
}
```

## 实践示例

### 完整的战斗系统

```rust
fn combat_system(
    mut commands: Commands,
    time: Res<Time>,
    mut targets: Query<(Entity, &mut Health, &Position)>,
    attackers: Query<(&Position, &Weapon, &Target)>,
) {
    for (attacker_pos, weapon, target) in attackers.iter() {
        if let Ok((entity, mut health, target_pos)) = targets.get_mut(target.entity) {
            // 检查距离
            let distance = attacker_pos.distance(target_pos);
            if distance > weapon.range {
                continue;
            }
            
            // 检查冷却
            if weapon.cooldown_remaining > 0.0 {
                continue;
            }
            
            // 造成伤害
            health.current -= weapon.damage;
            
            // 死亡检查
            if health.current <= 0 {
                commands.entity(entity).despawn();
            }
        }
    }
}
```

## 优势总结

1. **灵活组合**：通过组件组合创建复杂实体
2. **高性能**：数据连续存储，缓存友好
3. **并行友好**：不同系统可并行执行
4. **清晰职责**：每个系统只处理特定逻辑

## 下一步

- 了解网络同步如何复制组件：[网络同步](./network-sync.md)
- 了解服务端如何使用 ECS：[Server 模块](../modules/server.md)
