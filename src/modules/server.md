# Server - 权威服务端

`server` crate 实现游戏的权威逻辑。

## 职责

- 维护权威游戏状态 (Bevy ECS World)
- 加载地图和生成单位
- 处理玩家输入命令
- 执行游戏规则 (移动、战斗、资源生产)
- 判断胜负条件
- 向客户端复制状态

## 架构

```text
Server
  ├── Bevy App
  │   ├── ECS World (权威状态)
  │   └── Systems (游戏规则)
  ├── Command Handler (处理玩家输入)
  └── Replication (同步到客户端)
```

## 核心系统

### 移动系统

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

### 战斗系统

```rust
fn combat_system(
    mut commands: Commands,
    mut targets: Query<(&mut Health, &Position)>,
    attackers: Query<(&Weapon, &Position, &Target)>,
) {
    for (weapon, attacker_pos, target) in attackers.iter() {
        if let Ok((mut health, target_pos)) = targets.get_mut(target.entity) {
            let distance = attacker_pos.distance(target_pos);
            if distance <= weapon.range {
                health.current -= weapon.damage;
                if health.current <= 0 {
                    commands.entity(target.entity).despawn();
                }
            }
        }
    }
}
```

### 资源生产系统

```rust
fn resource_production_system(
    time: Res<Time>,
    mut players: Query<(&mut Resources, &Buildings)>,
    extractors: Query<&Extractor>,
) {
    for (mut resources, buildings) in players.iter_mut() {
        for building in buildings.iter() {
            if let Ok(extractor) = extractors.get(*building) {
                resources.credits += extractor.rate * time.delta_seconds();
            }
        }
    }
}
```

## 命令处理

```rust
impl Server {
    pub fn handle_input(&mut self, player_id: u64, input: PlayerInput) {
        match input {
            PlayerInput::Move { target } => {
                self.execute_move_command(player_id, target);
            }
            PlayerInput::Attack { entity_id } => {
                self.execute_attack_command(player_id, entity_id);
            }
            PlayerInput::Build { unit_type, position } => {
                self.execute_build_command(player_id, unit_type, position);
            }
            PlayerInput::Stop => {
                self.execute_stop_command(player_id);
            }
        }
    }
}
```

## 状态复制

使用 Lightyear 自动复制组件：

```rust
// 标记需要复制的组件
app.add_plugins(ReplicationPlugins)
    .replicate::<Position>()
    .replicate::<Health>()
    .replicate::<Velocity>();
```

## 地图加载

```rust
impl Server {
    pub fn load_map(&mut self, map_id: &ContentId) -> Result<()> {
        let map = self.content_db.get_map(map_id)?;
        
        // 生成地形
        for layer in &map.terrain_layers {
            self.spawn_terrain(layer)?;
        }
        
        // 生成初始单位
        for unit in &map.initial_units {
            self.spawn_unit(unit)?;
        }
        
        // 设置队伍和资源
        for team in &map.teams {
            self.setup_team(team)?;
        }
        
        Ok(())
    }
}
```

## 下一步

- 了解客户端实现：[Client](./client.md)
- 了解运行时编排：[Runtime Core](./runtime-core.md)
