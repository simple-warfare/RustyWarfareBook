# 常见问题

## 环境配置

### Q: 编译失败，提示链接错误

A: 确保安装了所有系统依赖。Windows 需要 Visual Studio Build Tools。

### Q: Bevy 编译时间很长

A: 首次编译 Bevy 需要 5-10 分钟，这是正常现象。后续增量编译会快很多。

## 运行问题

### Q: GDExtension 未加载

A: 
1. 运行 `cargo run -p builder`
2. 检查 `launcher/godot/addons/rusty_core/bin/` 目录
3. 重启 Godot 编辑器

### Q: 连接超时

A:
1. 检查防火墙设置
2. 确认端口开放 (默认 5000)
3. 验证 IP 地址正确

### Q: 内容包加载失败

A: 检查 `manifest.toml` 是否存在，验证 TOML 格式正确。

## 开发问题

### Q: 如何添加新单位？

A: 在 `content/units/` 创建 `.toml` 文件，参考现有单位模板。

### Q: 如何调试网络问题？

A: 设置 `RUST_LOG=lightyear=debug` 查看详细网络日志。

### Q: 如何修改架构？

A: 先阅读架构文档，理解分层设计和依赖规则，再提出修改建议。

## 更多帮助

- 查看 [贡献约定](../development/contributing.md)
- 阅读 [架构文档](../architecture/overview.md)
