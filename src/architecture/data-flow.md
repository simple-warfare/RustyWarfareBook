# 数据流

本章详细说明数据在系统各层之间的流动。

## 单人模式数据流

```text
┌─────────────────────────────────────────────┐
│             Godot (GDScript)                │
│  1. 用户点击 → get_global_mouse_position()  │
│  2. 调用 rusty_core.send_command()          │
└──────────────────┬──────────────────────────┘
                   │ Dictionary
                   ↓
┌─────────────────────────────────────────────┐
│            gdextension (Rust)               │
│  3. Dictionary → PlayerInput                │
│  4. runtime.send_input()                    │
└──────────────────┬──────────────────────────┘
                   │ PlayerInput
                   ↓
┌─────────────────────────────────────────────┐
│          runtime_core (Rust)                │
│  5. 分发到 client                            │
└──────────────────┬──────────────────────────┘
                   │ PlayerInput
                   ↓
┌─────────────────────────────────────────────┐
│             client (Rust)                   │
│  6. 本地预测 → 立即更新显示状态               │
│  7. 发送到 server (内存通道)                 │
└──────────────────┬──────────────────────────┘
                   │ PlayerInput
                   ↓
┌─────────────────────────────────────────────┐
│             server (Rust)                   │
│  8. 处理输入命令                             │
│  9. 执行权威逻辑 (Bevy ECS)                  │
│  10. 更新 ECS World                         │
│  11. 复制组件 → client                       │
└──────────────────┬──────────────────────────┘
                   │ Replicated Components
                   ↓
┌─────────────────────────────────────────────┐
│             client (Rust)                   │
│  12. 接收权威状态                            │
│  13. Reconciliation (校正预测)              │
│  14. 生成 FrontendSnapshot                  │
└──────────────────┬──────────────────────────┘
                   │ FrontendSnapshot
                   ↓
┌─────────────────────────────────────────────┐
│          runtime_core (Rust)                │
│  15. 转发 snapshot                          │
└──────────────────┬──────────────────────────┘
                   │ FrontendSnapshot
                   ↓
┌─────────────────────────────────────────────┐
│            gdextension (Rust)               │
│  16. FrontendSnapshot → Dictionary          │
└──────────────────┬──────────────────────────┘
                   │ Dictionary
                   ↓
┌─────────────────────────────────────────────┐
│             Godot (GDScript)                │
│  17. 读取 snapshot["units"]                 │
│  18. 更新 Node2D 位置                        │
│  19. 渲染到屏幕                              │
└─────────────────────────────────────────────┘
```

## 远程模式数据流

### 客户端 → 服务器

```text
Godot → gdextension → runtime_core → client
                                      ↓
                              ┌──────────────┐
                              │ 本地预测     │ (立即反馈)
                              └──────────────┘
                                      ↓
                              ┌──────────────┐
                              │ UDP Socket   │
                              └──────┬───────┘
                                     │ 网络延迟 (50-100ms)
                                     ↓
                              ┌──────────────┐
                              │远程 Server   │
                              └──────────────┘
```

### 服务器 → 客户端

```text
                              ┌──────────────┐
                              │远程 Server   │
                              │执行权威逻辑  │
                              └──────┬───────┘
                                     │ 网络延迟 (50-100ms)
                                     ↓
                              ┌──────────────┐
                              │ UDP Socket   │
                              └──────┬───────┘
                                     ↓
client ← runtime_core ← gdextension ← Godot
   ↓
┌──────────────┐
│ Reconcile    │ (对比预测和权威)
└──────┬───────┘
       ↓
┌──────────────┐
│ Interpolate  │ (平滑过渡)
└──────┬───────┘
       ↓
FrontendSnapshot → 显示
```

## 启动时数据流

```text
1. Godot 启动
   └─> 加载 gdextension.dll

2. gdextension 初始化
   └─> 注册 RustyCore 类

3. GDScript 调用 RustyCore.initialize(config)
   └─> Dictionary → RuntimeConfig
       └─> runtime_core 初始化

4. runtime_core.initialize()
   └─> 加载内容包
       └─> content::ContentLoader.load_package()
           └─> 读取 manifest.toml
           └─> 解析 units/*.toml
           └─> 验证 schema
           └─> 构建 ContentDatabase

5. GDScript 调用 RustyCore.start_singleplayer()
   └─> runtime_core.start_singleplayer()
       ├─> 创建 Server (使用 ContentDatabase)
       │   └─> 初始化 Bevy App
       │   └─> 加载地图
       │   └─> 生成初始单位
       └─> 创建 Client
           └─> 建立与 server 的内存通道

6. 进入游戏循环
```

## 游戏循环数据流

### 每帧流程 (60 FPS)

```text
Godot _process(delta):
  ├─> rusty_core.update(delta)
  │   └─> runtime_core.update()
  │       ├─> server.update()
  │       │   └─> bevy_app.update()
  │       │       ├─> movement_system
  │       │       ├─> combat_system
  │       │       ├─> resource_system
  │       │       └─> replication_system
  │       └─> client.update()
  │           ├─> 接收复制组件
  │           ├─> reconcile
  │           └─> interpolate
  │
  ├─> var snapshot = rusty_core.get_frontend_snapshot()
  │   └─> client.get_frontend_snapshot()
  │       └─> 遍历 ECS → 构建 FrontendSnapshot
  │           └─> Dictionary
  │
  └─> render_snapshot(snapshot)
      └─> 更新 Godot 节点
```

## 关键数据结构转换

### Rust → Godot

```text
FrontendSnapshot (Rust)
  ├─> units: Vec<FrontendUnit>
  │   └─> [
  │         FrontendUnit { id, position, health, ... },
  │         FrontendUnit { ... },
  │       ]
  └─> resources: HashMap<u64, PlayerResources>

         ↓ (gdextension 转换)

Dictionary (Godot)
  ├─> "units": Array[Dictionary]
  │   └─> [
  │         { "id": 1, "position": Vector2(100, 200), ... },
  │         { "id": 2, ... },
  │       ]
  └─> "resources": Dictionary
      └─> { 0: 5000, 1: 4500 }
```

### Godot → Rust

```text
Dictionary (Godot)
  ├─> "type": "move"
  └─> "target": Vector2(300, 400)

         ↓ (gdextension 转换)

PlayerInput (Rust)
  └─> PlayerInput::Move {
        target: Position { x: 300.0, y: 400.0 }
      }
```

## 状态同步机制

### Lightyear 复制流程

```text
Server:
  Entity(123)
    ├─> Position { x: 100.0, y: 200.0 }  [Replicated]
    ├─> Health { current: 80, max: 100 } [Replicated]
    └─> Velocity { ... }                  [Replicated]

         ↓ (Lightyear 自动序列化和发送)

Client:
  Entity(123)  (镜像实体)
    ├─> Position { x: 100.0, y: 200.0 }  [从 server 接收]
    ├─> Health { current: 80, max: 100 } [从 server 接收]
    └─> Velocity { ... }                  [从 server 接收]
```

## 下一步

了解核心概念详解：
- [权威服务端](../concepts/authoritative-server.md)
- [客户端预测](../concepts/prediction.md)
- [网络同步](../concepts/network-sync.md)
