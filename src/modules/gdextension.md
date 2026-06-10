# GDExtension - Godot接入

`gdextension` crate 负责 Rust ↔ Godot 的边界适配。

## 职责

- 注册 Godot 类 (RustyCore)
- Rust 类型 ↔ Godot 类型转换
- 暴露 Rust API 给 GDScript
- 处理跨语言调用

## 核心类型

### RustyCore

```rust
use godot::prelude::*;

#[derive(GodotClass)]
#[class(base=Node)]
pub struct RustyCore {
    runtime: Runtime,
    #[base]
    base: Base<Node>,
}
```

## 暴露的 API

### 初始化

```rust
#[godot_api]
impl RustyCore {
    #[func]
    pub fn initialize(&mut self, config: Dictionary) {
        let rust_config = RuntimeConfig {
            assets_root: PathBuf::from(
                config.get("assets_root")
                    .unwrap()
                    .to::<GString>()
                    .to_string()
            ),
            content_package_path: PathBuf::from(
                config.get("content_package_path")
                    .unwrap()
                    .to::<GString>()
                    .to_string()
            ),
        };
        
        self.runtime.initialize(rust_config).unwrap();
    }
}
```

### 启动游戏

```rust
#[godot_api]
impl RustyCore {
    #[func]
    pub fn start_singleplayer(&mut self) {
        self.runtime.start_singleplayer().unwrap();
    }
    
    #[func]
    pub fn start_host(&mut self, port: i32) {
        self.runtime.start_host(port as u16).unwrap();
    }
    
    #[func]
    pub fn start_client(&mut self, server_addr: GString) {
        self.runtime.start_client(server_addr.to_string()).unwrap();
    }
}
```

### 更新和获取数据

```rust
#[godot_api]
impl RustyCore {
    #[func]
    pub fn update(&mut self, delta: f64) {
        self.runtime.update(delta).unwrap();
    }
    
    #[func]
    pub fn get_frontend_snapshot(&self) -> Dictionary {
        let snapshot = self.runtime.get_frontend_snapshot();
        snapshot_to_dictionary(snapshot)
    }
}
```

### 发送命令

```rust
#[godot_api]
impl RustyCore {
    #[func]
    pub fn send_command(&mut self, command: Dictionary) {
        let input = dictionary_to_player_input(command);
        self.runtime.send_input(input).unwrap();
    }
}
```

## 类型转换

### FrontendSnapshot → Dictionary

```rust
fn snapshot_to_dictionary(snapshot: FrontendSnapshot) -> Dictionary {
    let mut dict = Dictionary::new();
    
    // 转换单位列表
    let mut units_array = Array::new();
    for unit in snapshot.units {
        let mut unit_dict = Dictionary::new();
        unit_dict.set("id", unit.id);
        unit_dict.set("position", Vector2::new(unit.position.0, unit.position.1));
        unit_dict.set("health", unit.health.0);
        unit_dict.set("max_health", unit.health.1);
        unit_dict.set("unit_type", GString::from(unit.unit_type));
        units_array.push(unit_dict);
    }
    dict.set("units", units_array);
    
    // 转换资源
    let mut resources_dict = Dictionary::new();
    for (player_id, res) in snapshot.resources {
        resources_dict.set(player_id, res.credits);
    }
    dict.set("resources", resources_dict);
    
    dict
}
```

### Dictionary → PlayerInput

```rust
fn dictionary_to_player_input(dict: Dictionary) -> PlayerInput {
    let cmd_type = dict.get("type").unwrap().to::<GString>().to_string();
    
    match cmd_type.as_str() {
        "move" => {
            let target = dict.get("target").unwrap().to::<Vector2>();
            PlayerInput::Move {
                target: Position {
                    x: target.x,
                    y: target.y,
                }
            }
        }
        "attack" => {
            let entity_id = dict.get("entity_id").unwrap().to::<u64>();
            PlayerInput::Attack { entity_id }
        }
        _ => panic!("Unknown command type"),
    }
}
```

## GDScript 使用示例

```gdscript
extends Node

var rusty_core: RustyCore

func _ready():
    rusty_core = RustyCore.new()
    add_child(rusty_core)
    
    # 初始化
    rusty_core.initialize({
        "assets_root": "res://assets",
        "content_package_path": "res://assets/official"
    })
    
    # 启动单人游戏
    rusty_core.start_singleplayer()

func _process(delta):
    # 更新游戏逻辑
    rusty_core.update(delta)
    
    # 获取数据
    var snapshot = rusty_core.get_frontend_snapshot()
    render_snapshot(snapshot)

func _input(event):
    if event is InputEventMouseButton and event.pressed:
        # 发送移动命令
        rusty_core.send_command({
            "type": "move",
            "target": get_global_mouse_position()
        })
```

## 构建和部署

使用 `builder` 工具自动构建：

```bash
cargo run -p builder
```

产物位置：
```text
launcher/godot/addons/rusty_core/bin/
  windows/libgdextension.dll
  linux/libgdextension.so
  macos/libgdextension.dylib
```

## 注意事项

### 线程安全

Godot 在主线程调用 Rust，确保：
- 不在 Rust 中持有 Godot 对象的可变引用
- 使用 `Gd<T>` 而非 `&T`

### 错误处理

GDScript 没有 Result，错误需要转换：

```rust
#[func]
pub fn start_singleplayer(&mut self) -> bool {
    match self.runtime.start_singleplayer() {
        Ok(_) => true,
        Err(e) => {
            godot_error!("Failed to start: {}", e);
            false
        }
    }
}
```

## 下一步

- 了解构建流程：[Builder](./builder.md)
- 了解开发约定：[贡献约定](../development/contributing.md)
