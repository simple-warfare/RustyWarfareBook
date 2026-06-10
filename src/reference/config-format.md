# 配置文件格式

RustyWarfare 使用 TOML 格式定义游戏内容。

## 单位模板

```toml
id = "tank"
display_name = "Tank"

[stats]
health = 100
speed = 2.0
cost = 300

[combat]
damage = 15
range = 150.0
projectile = "official:tank_shell"

[visual]
sprite = "units/tank.png"
```

## 地图模板

```toml
id = "duel_fields"
title = "Duel Fields"

[dimensions]
width = 64
height = 64

[[teams]]
id = 0
name = "Player 1"

[[spawns]]
team = 0
x = 120.0
y = 240.0
units = ["official:builder"]
```

## 下一步

- 查看 [Content 模块](../modules/content.md)
- 查看 [内容包系统](../concepts/content-package.md)
