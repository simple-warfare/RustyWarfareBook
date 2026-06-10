# Content - 内容包系统

`content` crate 负责游戏数据的加载、验证和管理。

## 职责

- 解析内容包 manifest
- 加载单位模板、地图模板、资源定义
- Schema 验证
- 计算内容包 fingerprint
- 跨引用检查
- 构建 ContentDatabase

## 核心概念

### ContentId

所有内容使用命名空间 ID：

```rust
pub struct ContentId {
    namespace: String,  // "official", "mod_name"
    name: String,       // "tank", "builder"
}

// 示例：
// - official:tank
// - official:builder
// - mega_builders:mega_builder
```

### ContentDatabase

运行时内容数据库：

```rust
pub struct ContentDatabase {
    pub units: HashMap<ContentId, UnitTemplate>,
    pub maps: HashMap<ContentId, MapTemplate>,
    pub projectiles: HashMap<ContentId, ProjectileTemplate>,
    // ...
}
```

### 内容包结构

```text
official_base_game/
  manifest.toml           # 包元数据
  units/
    tank.toml
    builder.toml
    engineer.toml
  maps/
    duel_fields/
      map.toml
  projectiles/
    tank_shell.toml
  resources/
    credits.toml
  factions/
    human.toml
```

## 加载流程

```text
1. 读取 manifest.toml
2. 验证版本和依赖
3. 加载所有 TOML 文件
4. Schema 验证
5. 解析跨引用
6. 构建 ContentDatabase
7. 计算 fingerprint
```

## 使用示例

```rust
use content::{ContentLoader, ContentDatabase};

// 加载内容包
let loader = ContentLoader::new();
let db = loader.load_package("path/to/official_base_game")?;

// 查询单位
let tank = db.get_unit(&ContentId::parse("official:tank")?)?;
println!("Tank HP: {}", tank.health);

// 加载地图
let map = db.get_map(&ContentId::parse("official:dev_test_map")?)?;
println!("Map size: {}x{}", map.width, map.height);
```

## 配置文件格式

### manifest.toml

```toml
id = "official_base_game"
title = "Official Base Game"
version = "0.1.0"
content_version = 1

[entry]
units = "units"
maps = "maps"
```

### 单位模板 (units/tank.toml)

```toml
id = "tank"
display_name = "Tank"

[stats]
health = 100
speed = 2.0
cost = 300

[combat]
damage = 15
range = 150
projectile = "official:tank_shell"
```

## 下一步

- 了解网络协议：[Protocol](./protocol.md)
- 了解服务端如何使用内容：[Server](./server.md)
