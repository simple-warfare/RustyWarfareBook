# RustyWarfare 开发者文档

这是 RustyWarfare 项目的完整开发者文档，使用 mdBook 构建。

## 📚 文档内容

- **快速开始** - 环境配置、构建、运行调试
- **架构设计** - 总体架构、分层设计、运行模式、数据流
- **核心模块** - Content、Protocol、Server、Client、Runtime Core、GDExtension、Builder
- **核心概念** - 权威服务端、ECS架构、网络同步、客户端预测、内容包系统
- **开发指南** - 贡献约定、代码规范、测试、调试
- **参考资料** - API文档、配置格式、常见问题

## 🚀 本地预览

```bash
# 安装 mdBook
cargo install mdbook

# 启动本地服务器
mdbook serve
```

访问 http://localhost:3000

## 📖 在线访问

推送到 main 分支后，文档会自动部署到：
https://your-org.github.io/rusty_warfare/

## 🛠️ 构建

```bash
mdbook build
```

生成的文档在 `book/` 目录。

## 📝 维护

修改文档：
1. 编辑 `src/` 目录下的 Markdown 文件
2. 本地预览验证
3. 提交并推送到 main 分支
4. GitHub Actions 自动部署

## 📂 目录结构

```
RustyWarfareBook/
├── book.toml              # mdBook 配置
├── src/
│   ├── SUMMARY.md         # 目录
│   ├── introduction.md    # 首页
│   ├── getting-started/   # 快速开始
│   ├── architecture/      # 架构设计
│   ├── modules/           # 核心模块
│   ├── concepts/          # 核心概念
│   ├── development/       # 开发指南
│   └── reference/         # 参考资料
└── .github/workflows/     # 自动部署配置
```

## 🎯 特点

- ✅ 详细的架构说明和代码示例
- ✅ 从入门到深入的完整学习路径
- ✅ 内置搜索功能
- ✅ 响应式设计，支持移动端
- ✅ 自动部署到 GitHub Pages

---

**文档版本**: 基于项目 commit 状态自动生成
