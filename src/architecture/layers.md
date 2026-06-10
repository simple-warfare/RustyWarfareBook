# 分层设计

RustyWarfare 采用严格的分层架构，每层有明确的职责边界。

## 分层概览

```text
┌─────────────────────────────────────┐
│  Layer 8: Godot 前端 (GDScript)     │  表现层
├─────────────────────────────────────┤
│  Layer 7: gdextension (Rust)        │  接入层
├─────────────────────────────────────┤
│  Layer 6: runtime_core (Rust)       │  编排层
├─────────────────────────────────────┤
│  Layer 5: client (Rust)             │  客户端层
├─────────────────────────────────────┤
│  Layer 4: server (Rust)             │  服务端层
├─────────────────────────────────────┤
│  Layer 3: protocol (Rust)           │  协议层
├─────────────────────────────────────┤
│  Layer 2: content (Rust)            │  内容层
├─────────────────────────────────────┤
│  Layer 1: game_domain (Rust)        │  领域模型层
└─────────────────────────────────────┘
```

## Layer 1: game_domain - 领域模型层

**职责**：定义游戏领域的基础类型和概念

**包含**：
- 坐标系统 (Position, Vec2)
- 单位ID、玩家ID
- 资源类型
- 游戏状态枚举

**特点**：
- 无依赖，纯数据结构
- 可被所有上层使用
- 不包含业务逻辑

## Layer 2: content - 内容层

**职责**：内容包的加载、验证和管理

**包含**：
- 内容包 manifest 解析
- 单位模板、地图模板
- Schema 验证
- Fingerprint 计算
- 跨引用检查
- ContentDatabase

**特点**：
- 只依赖 `game_domain`
- 不依赖 Bevy、Godot
- 纯文件系统操作

**核心类型**：
```rust
pub struct ContentDatabase {
    units: HashMap<ContentId, UnitTemplate>,
    maps: HashMap<ContentId, MapTemplate>,
    // ...
}
```

## Layer 3: protocol - 协议层

**职责**：定义 client 和 server 的共享协议

**包含**：
- 网络消息定义
- Lightyear 复制组件
- 玩家输入命令
- 同步常量 (tick rate 等)

**特点**：
- 被 `client` 和 `server` 共同依赖
- 不能依赖 `client` 或 `server`
- 定义数据契约，不实现逻辑

**核心类型**：
```rust
#[derive(Serialize, Deserialize, Component)]
pub struct Position {
    pub x: f32,
    pub y: f32,
}

#[derive(Serialize, Deserialize)]
pub enum PlayerInput {
    Move { target: Position },
    Attack { target: EntityId },
}
```

## Layer 4: server - 服务端层

**职责**：权威游戏逻辑

**包含**：
- Bevy ECS World
- 游戏规则系统
- 地图加载
- 单位生成
- 战斗结算
- 资源生产
- 胜负判定

**依赖**：
- `protocol` (共享组件)
- `content` (加载游戏数据)

**核心系统**：
```rust
fn combat_system(
    mut query: Query<(&mut Health, &Damage)>,
) {
    // 权威战斗逻辑
}
```

## Layer 5: client - 客户端层

**职责**：客户端状态管理和预测

**包含**：
- 输入缓冲
- 客户端预测
- 服务器状态接收
- Reconciliation (校正)
- 插值
- 前端 Snapshot 生成

**依赖**：
- `protocol` (共享组件)

**不包含**：
- 权威规则
- Godot 类型

**核心流程**：
```rust
impl Client {
    pub fn predict_input(&mut self, input: PlayerInput) {
        // 本地预测
    }
    
    pub fn reconcile(&mut self, server_state: ServerSnapshot) {
        // 对比预测和服务器状态，校正
    }
    
    pub fn get_frontend_snapshot(&self) -> FrontendSnapshot {
        // 生成前端可消费的数据
    }
}
```

## Layer 6: runtime_core - 编排层

**职责**：运行时生命周期管理和模式编排

**包含**：
- Runtime 结构
- 运行模式管理 (单人/主机/远程)
- Server/Client 启动和关闭
- 前端 Snapshot 投影
- 错误处理

**依赖**：
- `client`
- `server`
- `protocol`
- `content`

**核心 API**：
```rust
pub struct Runtime {
    mode: RuntimeMode,
    server: Option<Server>,
    client: Option<Client>,
}

impl Runtime {
    pub fn start_singleplayer(&mut self) -> Result<()>;
    pub fn update(&mut self, delta: f64) -> Result<()>;
    pub fn get_snapshot(&self) -> FrontendSnapshot;
}
```

## Layer 7: gdextension - 接入层

**职责**：Godot ↔ Rust 边界适配

**包含**：
- GodotClass 定义
- Rust ↔ Godot 类型转换
- RustyCore 接口暴露
- Dictionary/Array 转换

**依赖**：
- `runtime_core` (唯一依赖)
- `godot` crate (Godot 绑定)

**核心实现**：
```rust
#[derive(GodotClass)]
#[class(base=Node)]
pub struct RustyCore {
    runtime: Runtime,
}

#[godot_api]
impl RustyCore {
    #[func]
    pub fn initialize(&mut self, config: Dictionary) {
        // 转换 Godot 类型 → Rust 类型
    }
    
    #[func]
    pub fn get_snapshot(&self) -> Dictionary {
        // 转换 Rust 类型 → Godot 类型
    }
}
```

## Layer 8: Godot 前端

**职责**：渲染和用户交互

**包含**：
- UI 场景
- 输入处理
- 节点更新
- 资源加载 (贴图、音频)

**特点**：
- 不包含游戏规则
- 通过 `RustyCore` 接口访问 Rust
- 只读取 Snapshot，不直接访问 ECS

## 依赖规则

### 允许的依赖方向

```text
上层 → 下层  ✅
同层平行     ✅ (client ↔ server 除外)
下层 → 上层  ❌ 禁止
```

### 具体禁止项

- ❌ `protocol` 不得依赖 `client` 或 `server`
- ❌ `content` 不得依赖 `server`
- ❌ `server` 不得依赖 `client`
- ❌ `client` 不得依赖 `server`
- ❌ `runtime_core` 不得包含游戏规则
- ❌ `gdextension` 不得包含游戏规则

## 为什么严格分层？

1. **可测试性**：每层可独立测试
2. **复用性**：底层可被多个上层使用
3. **可维护性**：修改某层不影响其他层
4. **防止循环依赖**：Rust 禁止循环依赖
5. **清晰职责**：每层职责明确，不会混淆

## 实践示例

### 添加新单位

1. **Layer 2 (content)**：定义单位模板 TOML
2. **Layer 4 (server)**：实现单位生成和行为系统
3. **Layer 3 (protocol)**：添加复制组件 (如需同步)
4. **Layer 5 (client)**：处理预测逻辑 (如需)
5. **Layer 8 (Godot)**：添加贴图和渲染节点

### 添加新网络消息

1. **Layer 3 (protocol)**：定义消息结构
2. **Layer 4 (server)**：处理接收逻辑
3. **Layer 5 (client)**：发送逻辑

### 修改 UI

1. **Layer 8 (Godot)**：修改 GDScript 和场景
2. 无需修改 Rust 代码

## 下一步

了解不同运行模式下的差异：[运行模式](./runtime-modes.md)
