# 总体架构

RustyWarfare 采用**权威服务端架构**，结合 **Bevy ECS** 和 **Lightyear 网络同步**，实现了单人、主机、远程三种模式下的统一游戏逻辑。

## 架构原则

1. **服务端权威**：所有游戏规则在服务端执行，客户端只负责输入和显示
2. **单一规则集**：单人和多人使用相同的游戏逻辑，无重复代码
3. **数据驱动**：游戏内容通过 TOML 配置定义，支持 Mod 和内容包
4. **前端分离**：Godot 仅负责渲染和输入，不包含游戏规则
5. **严格分层**：模块间依赖单向，防止循环依赖

## 技术架构图

```text
┌─────────────────────────────────────────────┐
│          Godot 前端 (GDScript)              │
│         渲染、UI、输入采集                   │
└──────────────────┬──────────────────────────┘
                   │ GDExtension FFI
┌──────────────────▼──────────────────────────┐
│           gdextension (Rust)                │
│        Godot ↔ Rust 类型转换和接口           │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│         runtime_core (Rust)                 │
│    运行模式编排、生命周期管理、前端投影       │
└─────────┬──────────────────────┬────────────┘
          │                      │
┌─────────▼─────────┐   ┌────────▼───────────┐
│   client (Rust)   │   │   server (Rust)    │
│  预测、插值、校正  │   │  权威逻辑、ECS世界  │
└─────────┬─────────┘   └────────┬───────────┘
          │                      │
          └──────────┬───────────┘
                     │
        ┌────────────▼─────────────┐
        │    protocol (Rust)       │
        │  网络消息、共享组件定义    │
        └────────────┬─────────────┘
                     │
        ┌────────────▼─────────────┐
        │    content (Rust)        │
        │  内容包加载、验证、数据库  │
        └──────────────────────────┘
```

## 依赖关系

```text
gdextension
    └── runtime_core
        ├── client ──┐
        │            ├── protocol
        └── server ──┤
                     └── content
```

**关键约束**：
- `protocol` 和 `content` 是底层库，不依赖任何上层模块
- `server` 和 `client` 平行，互不依赖
- `runtime_core` 编排 server 和 client，但不实现游戏规则
- `gdextension` 仅做类型转换，不包含业务逻辑

## 核心流程

### 启动流程

```text
1. Godot 启动
2. 加载 gdextension 动态库
3. GDScript 调用 RustyCore.initialize()
4. runtime_core 读取配置
5. content 加载并验证内容包
6. 根据模式启动 server/client
7. 进入游戏循环
```

### 游戏循环 (单人模式)

```text
每帧：
1. Godot 采集输入 → GDScript
2. GDScript → gdextension.send_command()
3. gdextension → runtime_core.update()
4. runtime_core → client.process_input()
5. client → server (本地通道)
6. server 执行权威逻辑 (Bevy ECS)
7. server → client (复制组件)
8. client 进行预测和校正
9. client → runtime_core.get_snapshot()
10. runtime_core → gdextension (Dictionary)
11. gdextension → GDScript
12. GDScript 更新 Godot 节点
```

### 游戏循环 (远程模式)

```text
每帧：
1-4. [同单人模式]
5. client → 远程 server (UDP)
6. 本地 client 立即预测
7. 远程 server 执行权威逻辑
8. server → client (网络复制)
9. client reconcile (对比预测和权威结果)
10-12. [同单人模式]
```

## 为什么这样设计？

### 为什么保留 runtime_core？

如果没有 `runtime_core`，`gdextension` 会直接管理：
- Godot 类型转换
- Server 生命周期
- Client 生命周期
- 网络配置
- 错误处理

这会让 `gdextension` 变得臃肿且难以测试。`runtime_core` 提供纯 Rust API，使得：
- 可以编写纯 Rust 测试
- 未来可支持非 Godot 前端
- GDExtension 层保持简洁

### 为什么 client 和 server 分离？

- **明确职责**：server 只关心规则，client 只关心显示
- **防止泄漏**：防止客户端逻辑影响服务端
- **独立测试**：可以单独测试 server 逻辑
- **专用服务器**：未来可构建无 client 的专用服务器

### 为什么 protocol 独立？

`protocol` 定义 server 和 client 的"语言"：
- 网络消息格式
- 复制组件结构
- 共享常量

这些定义必须双方一致，因此独立为底层库。

## 下一步

- 了解详细分层设计：[分层设计](./layers.md)
- 了解运行模式差异：[运行模式](./runtime-modes.md)
- 了解数据流动：[数据流](./data-flow.md)
