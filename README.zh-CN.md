# TensorCLI

[English](README.md) | [简体中文](README.zh-CN.md)

**Tensor** 系列软件的命令行工具与 Agent 调用层，覆盖 **TensorReading**（文献管理与阅读）和 **TensorWriting**（学术写作）。

通过 TensorCLI，你自己、脚本以及各类 AI Agent（Claude Code、Copilot、Codex、MCP 客户端等）无需打开图形界面，就能检索、阅读、导入和引用桌面端所使用的同一份本地文献库。

> **当前状态：规划阶段。** 仓库目前只包含需求文档，见 [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md)，尚未实现任何功能。

## 目标

- 统一入口 `tensor`，按软件划分子命令：`tensor reading ...`、`tensor writing ...`。
- 稳定、可脚本化的输出（`--json`），便于 Shell 脚本和 Agent 可靠解析。
- 一等公民级别的 Agent 集成：提供 MCP Server 和可直接使用的 Agent Skill。
- 默认安全：读操作绝不修改文献库；写操作只走各软件官方的本地 API。

## 计划中的用法

以下为设计草案，可能调整。

```bash
# TensorReading
tensor reading status                          # 软件是否在运行、文献库位置
tensor reading search "deep brain stimulation" --limit 10 --json
tensor reading get <item_key> --fulltext       # 元数据、PDF 路径、AI 笔记
tensor reading collections
tensor reading import --doi 10.48550/arXiv.1706.03762 --collection "To read"
tensor reading cite <item_key> --style apa

# TensorWriting
tensor writing status
tensor writing list
# 其余命令待确认 TensorWriting 的对外接口后再定义

# Agent 集成
tensor mcp serve                               # 基于 stdio 的 MCP Server
tensor skill install --target claude           # 安装 Agent Skill
```

## 工作原理

TensorCLI 通过本机上各 Tensor 软件的本地接口工作，不会向任何远程服务发送数据。

| 接口 | 用途 | 可用条件 |
|---|---|---|
| 本地 HTTP API（TensorReading：`127.0.0.1:23120`） | 元数据检索、引文、所有写操作（导入） | 软件需处于运行状态 |
| 文献库存储目录（只读 SQLite 与文件） | PDF 全文、AI 笔记、更复杂的查询 | 始终可用，软件关闭时也可用 |

## 路线图

| 里程碑 | 范围 |
|---|---|
| M0 | 建库、README、需求文档（本次提交） |
| M1 | CLI 骨架、配置、`tensor reading` 只读命令 |
| M2 | `tensor reading import`、查重、PDF 附件 |
| M3 | MCP Server 与 Agent Skill |
| M4 | `tensor writing` 命令 |
| M5 | 打包与发布（先 Windows，再 macOS、Linux） |

详细需求与验收标准见 [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md)。

## 参与贡献

欢迎提交 Issue 和讨论。新增命令前请先阅读需求文档，以保持命名和输出约定一致。

## 许可证

[MIT](LICENSE)
