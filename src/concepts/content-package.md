# 内容包系统

内容包系统是 RustyWarfare 的数据驱动核心，支持官方内容、地图和 Mod。

## 设计理念

**游戏规则和游戏数据分离**

- **规则**：Rust 代码实现（移动、战斗、生产）
- **数据**：TOML 配置定义（单位属性、地图布局）

好处：
- 修改数据无需重新编译
- 支持 Mod
- 多人游戏内容一致性验证

## 内容包结构

```text
official_base_game/
  manifest.toml          # 包元数据
  units/                 # 单位定义
    tank.toml
    builder.toml
  maps/                  # 地图
    duel_fields/
      map.toml
  projectiles/           # 弹丸
    tank_shell.toml
  resources/             # 资源类型
    credits.toml
```

## 使用示例

```rust
use content::ContentLoader;

// 加载内容包
let loader = ContentLoader::new();
let db = loader.load_package("assets/official")?;

// 查询单位
let tank = db.get_unit(&ContentId::parse("official:tank")?)?;

// 计算 fingerprint
let fingerprint = db.fingerprint();
```

## 多人一致性

通过 fingerprint 保证所有玩家使用相同的内容包：

```rust
// 客户端连接时验证
if client_fingerprint != server_fingerprint {
    return Err("Content mismatch");
}
```

## 下一步

- 了解配置格式：[配置文件格式](../reference/config-format.md)
- 了解内容加载：[Content 模块](../modules/content.md)
