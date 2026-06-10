# RustyWarfare 开发者文档

欢迎来到 RustyWarfare 开发者文档！

## 项目简介

**RustyWarfare** 是一款面向未来题材的实时策略游戏的 Rust 核心引擎。项目实现了：

- **权威服务端架构**：所有游戏逻辑在服务端执行，杜绝作弊
- **网络同步**：基于 Lightyear 的客户端预测、插值和复制
- **ECS 架构**：使用 Bevy ECS 管理游戏状态
- **数据驱动**：通过 TOML 配置文件定义游戏内容
- **跨平台**：支持 Windows、Linux、Android 等平台
- **Godot 前端**：通过 GDExtension 接入 Godot 引擎

## 技术栈

| 类别 | 技术 |
|------|------|
| 核心语言 | Rust 2024 Edition |
| 游戏框架 | Bevy 0.18.1 |
| 网络同步 | Lightyear 0.26.4 |
| 前端引擎 | Godot 4.x (通过 GDExtension) |
| 序列化 | Serde, TOML |
| 构建系统 | Cargo Workspace |

## 运行模式

项目支持三种运行模式，所有模式使用统一的游戏规则：

1. **单人模式**：本地 server + 本地 client
2. **主机模式**：本地 server + 本地 client + 远程 clients
3. **远程模式**：远程 server + 本地 client

## Workspace 结构

```text
rusty_warfare/
├── content/         # 内容包加载与验证
├── protocol/        # 网络协议定义
├── server/          # 权威服务端逻辑
├── client/          # 客户端核心
├── runtime_core/    # 运行时编排
├── gdextension/     # Godot 接入层
├── builder/         # 构建工具
├── game_domain/     # 游戏领域模型
└── launcher/godot/  # Godot 前端项目
```

## 文档导航

- **新手？** 从 [环境配置](./getting-started/setup.md) 开始
- **了解架构？** 查看 [总体架构](./architecture/overview.md)
- **深入模块？** 浏览 [核心模块](./modules/content.md) 章节
- **参与开发？** 阅读 [贡献约定](./development/contributing.md)

## 获取帮助

- 查看 [常见问题](./reference/faq.md)
- 阅读项目根目录的 `README.md` 和 `CONTRIBUTING.md`
- 参考 `docs/` 目录下的架构文档
