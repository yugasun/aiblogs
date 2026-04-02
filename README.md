# AI Agent 技术博客

> 由 AI 生成的技术博客，深入探讨现代 AI Agent 的架构设计与最佳实践。

[![Blog](https://img.shields.io/badge/visit-aiblogs.yugasun.com-purple)](https://aiblogs.yugasun.com)
[![GitHub Pages](https://img.shields.io/badge/部署到-GitHub%20Pages-blue)](https://aiblogs.yugasun.com)
[![mdBook](https://img.shields.io/badge/静态网站生成器-mdBook-orange)](https://rust-lang.github.io/mdBook/)
[![License: APACHE-2.0](https://img.shields.io/badge/License-APACHE%202.0-green)](LICENSE)

## 关于本项目

`aiblogs` 是一个完全由 **AI 生成的静态技术博客系统**，具有以下特点：

- **AI 驱动**: 每篇文章由 AI 生成，并明确标注所使用的 AI 模型
- **自动化部署**: 推送至 `main` 分支自动构建并部署到 GitHub Pages
- **静态生成**: 使用 mdBook 构建，极速加载体验
- **开源免费**: 基于 Apache-2.0 许可证开源

## 文章列表

| 文章                                                      | AI 模型      | 更新时间   |
| --------------------------------------------------------- | ------------ | ---------- |
| [介绍](./src/introduction.md)                             | minimax-m2.5 | 2026-04-02 |
| [Claw Code 架构深度解析](./src/claw-code-architecture.md) | minimax-m2.5 | 2026-04-02 |

## 快速开始

### 本地预览

```bash
# 安装 mdBook
cargo install mdbook

# 启动本地服务器
mdbook serve

# 访问 http://localhost:3000
```

### 构建发布

```bash
# 构建静态网站
mdbook build

# 输出目录: book/
```

## 添加新文章

1. 在 `src/` 目录创建 `.md` 文件
2. 在 `src/SUMMARY.md` 添加链接
3. 文件开头添加 AI 模型标注：

```markdown
> *AI 模型: minimax-m2.5 | 生成时间: YYYY-MM-DD*
```

4. 更新 README.md 文章列表

## 自动部署流程

```
push main → GitHub Actions → mdbook build → GitHub Pages
```

工作流文件: `.github/workflows/deploy.yml`

部署地址: https://aiblogs.yugasun.com

## 技术栈

| 类别         | 技术                                          |
| ------------ | --------------------------------------------- |
| 静态网站生成 | [mdBook](https://rust-lang.github.io/mdBook/) |
| 托管平台     | GitHub Pages                                  |
| 持续集成     | GitHub Actions                                |
| 内容创作     | AI (minimax-m2.5)                             |

## License

APACHE-2.0 License - 详见 [LICENSE](LICENSE) 文件