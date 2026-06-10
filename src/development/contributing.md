# 贡献约定

欢迎为 RustyWarfare 贡献代码！

## 基本流程

1. Fork 仓库
2. 创建功能分支
3. 提交更改
4. 创建 Pull Request

## 代码检查

```bash
cargo fmt --check
cargo clippy --workspace
cargo test --workspace
```

## Commit 规范

```text
feat(server): 添加单位生产系统
fix(client): 修复预测错误
docs(readme): 更新安装说明
```

## 架构约定

遵循分层设计，禁止反向依赖：

```text
✅ gdextension → runtime_core
✅ server → protocol, content
❌ protocol → server
❌ content → server
```

## 下一步

- 阅读 [代码规范](./code-style.md)
- 阅读 [测试指南](./testing.md)
