# TensorCLI 需求文档

版本：0.1（草案）
状态：规划中，功能待开发

## 1. 背景与目标

Tensor 系列桌面软件（TensorReading、TensorWriting）目前只能通过图形界面使用。TensorCLI 提供命令行与 Agent 调用入口，使用户、脚本和 AI Agent 能在不打开 GUI 的情况下使用其核心能力。

目标：

1. 提供统一命令 `tensor`，覆盖 TensorReading 与 TensorWriting。
2. 提供稳定的机器可读输出，便于脚本与 Agent 解析。
3. 提供 MCP Server 和 Agent Skill，让主流 Agent 开箱即用。
4. 保证对用户数据的安全：读不改库，写走官方 API。

非目标：

- 不替代桌面应用的图形界面。
- 不提供云端服务或远程访问。
- 不直接写入应用的数据库或存储目录。

## 2. 用户与场景

| 用户 | 场景 |
|---|---|
| 研究者（命令行） | 在终端快速检索文献、查看元数据、导入论文 |
| 脚本作者 | 批量导入 DOI 列表、导出某个集合的引文 |
| AI Agent | 为写作或综述任务检索文献库、读取 PDF 与 AI 笔记、导入新文献、插入引文 |

## 3. 现有接口（已知事实）

### 3.1 TensorReading

基于现有 TensorReading 本地接口（见用户本机的 tensorreading skill）：

- **本地 HTTP API**：`http://127.0.0.1:23120`，仅在应用运行时可用。
  - `GET /` 返回应用名称与版本，用于探活。
  - `GET /api/cite/search?q=&limit=` 全文检索，返回 CSL-JSON。
  - `GET /api/cite/csl?keys=` 按 item_key 获取 CSL-JSON。
  - `GET /api/collections`、`POST /api/collections` 集合列表与创建。
  - `GET /api/items/check?doi=&title=&isbn=` 查重。
  - `POST /api/items` 创建条目；`POST /api/items/{id}/attachments/pdf` 附加 PDF（base64，最大 64 MB）。
- **存储目录**：默认 `%APPDATA%\com.tensorreading.desktop\storage`，可被 `storage_path.txt` 重定向。包含 `tensorreading.db`（SQLite）和每篇论文一个目录，其中可有 PDF、`ai_note.json`、`outline.json`、`blog_post.md`、`figs/`。
- 数据库必须以只读方式打开（`mode=ro`），应用可能同时在写入。

### 3.2 TensorWriting

对外接口尚未确认。需要先明确：

- 是否有本地 HTTP API 或其他 IPC 方式，端口与鉴权。
- 文档和项目的存储格式与位置。
- 与 TensorReading 的引文联动方式（预期使用 CSL-JSON）。

在以上确认前，第 4.2 节的需求只作为意图，不作为实现规格。

## 4. 功能需求

优先级：P0 必须、P1 重要、P2 可选。

### 4.1 `tensor reading`

| ID | 优先级 | 需求 |
|---|---|---|
| R-01 | P0 | `status`：检测应用是否运行（HTTP 探活）、显示版本、解析出存储目录路径 |
| R-02 | P0 | `search <query>`：全文检索，支持 `--limit`、`--collection`、`--tag`、`--year` |
| R-03 | P0 | `get <item_key>`：返回元数据、作者、标签、所属集合、PDF 绝对路径 |
| R-04 | P0 | `collections`：列出集合树 |
| R-05 | P0 | 应用未运行时，R-02 到 R-04 自动降级为只读 SQLite 查询，并在输出中标明数据来源 |
| R-06 | P1 | `get --note`：输出 `ai_note.json`、`outline.json`、`blog_post.md` 的内容 |
| R-07 | P1 | `get --fulltext`：提取并输出 PDF 文本，支持页码范围 |
| R-08 | P1 | `import`：通过 `--doi`、`--arxiv`、`--url` 或本地 PDF 导入；导入前自动查重；支持 `--collection`、`--tag` |
| R-09 | P1 | `import` 支持从文件批量导入（每行一个 DOI/arXiv ID），汇总成功、重复、失败 |
| R-10 | P1 | `cite <item_key...>`：输出 CSL-JSON、BibTeX 或按样式（如 apa）渲染的引文 |
| R-11 | P2 | `collections create <name>`：创建集合 |
| R-12 | P2 | `export`：按集合或检索结果导出 BibTeX、CSL-JSON、RIS |
| R-13 | P2 | `open <item_key>`：在 TensorReading 中定位并打开该条目 |

### 4.2 `tensor writing`（待确认接口后细化）

| ID | 优先级 | 需求 |
|---|---|---|
| W-01 | P0 | `status`：检测 TensorWriting 是否可用、显示版本与数据位置 |
| W-02 | P0 | `list`：列出写作项目或文档 |
| W-03 | P1 | `get <doc>`：读取文档内容（Markdown 或结构化 JSON） |
| W-04 | P1 | 创建文档、向文档追加或替换内容 |
| W-05 | P1 | 在文档中插入来自 TensorReading 的引文，并更新参考文献列表 |
| W-06 | P2 | 导出文档（如 PDF、DOCX、Markdown） |

### 4.3 Agent 集成

| ID | 优先级 | 需求 |
|---|---|---|
| A-01 | P0 | 所有命令支持 `--json`，输出结构稳定、带版本字段；错误也以 JSON 输出 |
| A-02 | P0 | `tensor mcp serve`：基于 stdio 的 MCP Server，把 4.1、4.2 的核心命令暴露为 tools |
| A-03 | P1 | 提供 Agent Skill（SKILL.md）及安装命令 `tensor skill install --target <claude\|copilot\|...>` |
| A-04 | P1 | MCP 的写类 tools（导入、修改文档）在 tool 描述中明确标注为写操作，并支持 `--read-only` 启动模式 |
| A-05 | P2 | `tensor schema`：输出各命令输入输出的 JSON Schema |

### 4.4 通用

| ID | 优先级 | 需求 |
|---|---|---|
| G-01 | P0 | 退出码约定：0 成功；1 一般错误；2 参数错误；3 应用未运行且无降级路径；4 未找到；5 重复 |
| G-02 | P0 | 配置：命令行参数 > 环境变量（如 `TENSOR_READING_URL`、`TENSOR_READING_STORAGE`）> 配置文件 > 默认值 |
| G-03 | P0 | 不上传任何本地数据；除用户显式指定 DOI/arXiv/URL 导入时获取元数据外，不发起远程请求 |
| G-04 | P1 | 人类可读输出（表格）与 JSON 输出二选一；`--quiet`、`--verbose` |
| G-05 | P1 | Shell 补全（PowerShell、bash、zsh） |
| G-06 | P2 | 多语言提示（中文、英文），跟随系统语言 |

## 5. 非功能需求

- **安全**：数据库只读打开；写操作仅经官方 HTTP API；HTTP 仅连接 `127.0.0.1`；不执行用户输入拼接的 SQL（使用参数化查询）。
- **兼容性**：Windows 优先，后续支持 macOS 和 Linux。存储路径解析需处理各平台差异。
- **性能**：本地检索在 10 万条目规模下响应小于 1 秒；`status` 小于 200 ms。
- **稳定性**：应用运行时并发读取不产生锁错误（只读、短连接、必要时重试）。
- **可测试性**：HTTP 和 SQLite 层可被模拟；提供包含示例数据库的测试夹具。

## 6. 技术方案（建议，待确认）

| 项 | 建议 | 理由 |
|---|---|---|
| 语言 | Python 3.11+ | 官方 MCP SDK 成熟；SQLite、PDF 文本提取生态完善 |
| CLI 框架 | Typer 或 Click | 子命令组织清晰，自带补全 |
| HTTP 客户端 | httpx | 同步与异步均可 |
| 分发 | `pipx` / `uv tool install`；后续可选单文件可执行包 | 安装简单 |
| 测试 | pytest | 通用 |

备选：TypeScript（Node 22），与 Tauri 前端技术栈一致。最终选择在 M1 前确定。

## 7. 里程碑与验收

| 里程碑 | 验收标准 |
|---|---|
| M0 | 仓库公开，README 与需求文档完成 |
| M1 | 完成 R-01 到 R-05、A-01、G-01、G-02；在应用运行与关闭两种状态下均可检索并读取条目 |
| M2 | 完成 R-08、R-09、R-10；重复导入被正确识别；PDF 能成功附加 |
| M3 | 完成 A-02、A-03、A-04；在至少一个 MCP 客户端和一个编码 Agent 中完成端到端演示 |
| M4 | 完成 W-01 到 W-05（取决于接口确认） |
| M5 | 发布到 PyPI（或等价渠道），提供 Release 与安装文档 |

## 8. 待决问题

1. TensorWriting 的对外接口形式、端口、鉴权与数据格式。
2. TensorReading HTTP API 是否会有版本号或鉴权变化，CLI 如何做兼容检测。
3. 技术栈最终选择（Python 或 TypeScript）。
4. 是否需要支持多个文献库或多个 profile。
5. PDF 全文提取是否由 CLI 实现，还是复用 TensorReading 已有的解析结果。
6. 发布渠道与项目命名（PyPI 包名、可执行文件名 `tensor` 是否冲突）。
