[English](README.md) | 中文

# DSH 插件避坑手册

非官方、社区维护的 **DeepSeek Harness（DSH）插件开发避坑手册**。

**[English handbook: docs/DSH-PLUGIN-PITFALLS.en.md](docs/DSH-PLUGIN-PITFALLS.en.md)** · **[中文手册：docs/DSH-PLUGIN-PITFALLS.zh-CN.md](docs/DSH-PLUGIN-PITFALLS.zh-CN.md)**

## 这是什么

官方 DSH 插件文档的实战笔记补充。它收集了端到端构建 DSH bundle 时真正耗时间的失败案例：

- 打包与安装（tarball 与目录 link、依赖解析、重启语义）
- Host/Client 两半、slot、locale、主题 token
- 配置 UI：`plugins.row.config` / `plugins.bundle.config` / `settings.section`、`volatile()` 字段、行 key 前缀
- 没有浏览器控制时用 `cordis_inspect_query` 做验证
- 可复现的发布资产、校验和、带审批闸门的 GitHub Actions
- 与 skill 及其他 bundle 共存

每条都按 **现象 → 根因 → 做法 → 验证** 编写，因此可以当检查清单用，而不是散文。

范围：DSH Desktop `44.7.0`、profile `desktop`、2026-10。DSH 迭代很快；请把这当作一个快照，两者冲突时以官方文档为准。

## 为什么单独建仓库

本手册有意独立于任何单个插件。无论其他 DSH 插件作者（以及其他 agent）构建什么，都应该能复用它。

## 贡献

只有当你能说清现象、你验证过的根因，以及能证明修复的检查时，才新增条目。请把官方行为与本地权宜做法清楚分开。

## 许可证

MIT。本文档不是 DeepSeek 或 DSH 的官方文档。
