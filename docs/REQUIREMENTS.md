# TensorCLI 需求文档

版本：0.2（草案）
状态：规划中，功能待开发

修订记录：

- 0.2：技术栈定为 TypeScript（Node 22+）；补充 TensorWriting 编译接口的调研结论；新增 `tensor writing compile` 需求（W-07 到 W-13），并把它放到第一个里程碑。
- 0.1：初稿。

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
- 不直接写入应用的数据库或存储目录（例外见第 8 节待决问题 2：TeX 资源缓存）。

## 2. 用户与场景

| 用户 | 场景 |
|---|---|
| 研究者（命令行） | 在终端快速检索文献、查看元数据、导入论文；不开 App 编译 LaTeX 项目 |
| 脚本作者 | 批量导入 DOI 列表、导出某个集合的引文；在脚本或 CI 中编译论文 |
| AI Agent | 为写作或综述任务检索文献库、读取 PDF 与 AI 笔记、导入新文献、插入引文；修改 LaTeX 后编译并读取诊断信息 |

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

TensorWriting 是本地 LaTeX 编辑器（Tauri：React 前端 + Rust 后端），项目就是磁盘上的普通文件夹。调研结论（基于 TensorWriting 0.1.8）：

**对外接口**

- 没有本地 HTTP API，只有 deep link 和单实例插件。CLI 不能请求正在运行的 App 编译。
- 项目文件可以直接读取。App 的项目级数据放在项目内的 `.tensorwriting/` 目录，例如 `.tensorwriting/preview/preview.pdf`（最近一次预览，附带 `preview.json` 记录主文件）、`.tensorwriting/texmf`（项目本地宏包）。
- 用户级偏好（每个项目选定的主文件、引擎等）在 `%APPDATA%\com.tensorwriting.desktop\project-preferences.json`。

**编译链路**

- 编译在 App 的 WebView 里完成，不在 Rust 端：前端通过 `texlyre-busytex` 在 Worker 中运行 BusyTeX（TeX Live 2026 编译成的 WebAssembly），支持 pdflatex、xelatex、lualatex，以及 bibtex8、biber、makeindex。
- 引擎选择：先看 `% !TeX program = ...` 魔法注释，再看是否使用 fontspec、xeCJK、ctex 或包含中文字符（是则用 xelatex），否则用 pdflatex。
- 主文件选择：优先 `main.tex`，否则选目录层级最浅、路径最短、同时包含 `\documentclass` 和 `\begin{document}` 的 `.tex` 文件。
- 不支持 shell escape、minted、SVG/EPS 外部转换。

**TeX 运行时**

- 由 App 下载安装，不随安装包分发。Windows 正式版在 `%LOCALAPPDATA%\TensorWriting\texlive\`，开发版在 `%APPDATA%\com.tensorwriting.desktop\texlive\`。
- `active.json` 指向 `installed\<generation>\`，内含 `busytex.js`、`busytex.wasm`（约 32 MB）、`busytex_pipeline.js`、`texlive.js` 和 `texlive.data`（minimal 版约 47 MB）、`resource-catalog.json` 等，附签名清单，兼容标识为 `busytex-1.4.0-tl2026`。
- 有 minimal 和 full 两个版本。minimal 只预装常用文件，其余按需下载。

**按需补包（minimal 版）**

1. TeX 缺文件时，kpathsea 发同步请求 `resources/<格式编号>/<文件名>`。
2. 资源服务查 `resource-catalog.json`（TeX Live 2026 全量文件到宏包的索引）：缓存里有就返回 200；索引里没有就返回 404；有但没缓存就返回 202，并在后台下载。
3. 下载先尝试 `https://texlive2026.texlyre.org/<格式>/<文件名>` 获取单个文件（按 SHA-256 校验）；失败再从清华镜像 TeX Live ISO 按字节范围下载整个宏包的 `.tar.xz`（校验 SHA-512 后解压入缓存）。
4. 本遍编译结束后，等待下载完成再重新编译，直到没有新下载为止（最多 256 轮）。
5. 缓存位于运行时目录下的 `online\texlive2026-20260301\`（`paths\` 存回执，`objects\` 按内容哈希存文件）。biber（约 31 MB）和 newtx 支持文件是单独组件，存放在 `online\components\`。

**可行性验证（2026-10-03）**

已在纯 Node 22 中直接加载用户本机安装的运行时，并编译出 PDF：不需要浏览器，也不需要 App 运行。只需模拟四个浏览器接口：`self`、`importScripts`（用 `vm` 实现）、读本地文件的 `fetch`，以及一个同步 `XMLHttpRequest`（按上述规则从缓存取文件）。pdflatex 三遍编译耗时约 1.3 秒，初始化约 0.6 秒。验证过程中只读取缓存，未实现下载。

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

### 4.2 `tensor writing`

#### 4.2.1 编译（首要功能）

```
tensor writing compile [项目目录|主文件.tex] [--main <path>] [--engine auto|pdflatex|xelatex|lualatex]
                       [--mode quick|full] [-o <out.pdf>] [--offline] [--json]
tensor writing runtime
```

| ID | 优先级 | 需求 |
|---|---|---|
| W-07 | P0 | `runtime`：定位 TensorWriting 已安装的 TeX 运行时（环境变量 `TENSOR_WRITING_RUNTIME` > 正式版目录 > 开发版目录），读取 `active.json`，校验兼容标识与清单 SHA-256；输出路径、版本、variant（minimal/full）、缓存大小。未安装时提示在 TensorWriting 中安装，退出码 3 |
| W-08 | P0 | `compile`：在 Node 中直接运行运行时内的 BusyTeX 编译项目，不依赖浏览器，也不要求 App 运行。项目扫描、主文件与引擎检测与 App 规则一致（见 3.2）；`--main`、`--engine` 可覆盖，默认沿用 App 中该项目的偏好 |
| W-09 | P0 | 输出：PDF 默认写到主文件同目录同名 `.pdf`，`-o` 可指定。`--json` 返回 `{success, pdf, engine, mode, passes, elapsedMs, diagnostics[], downloads[]}`；诊断格式与 App 诊断面板一致（文件、行、列、级别、消息）。编译失败退出码 6 |
| W-10 | P0 | 按需补包：实现 3.2 所述流程（202、下载、校验、重编）。下载来源和校验方式与 App 相同；下载进度输出到 stderr，`--json` 结果列出本次下载的文件或宏包。`--offline` 只用缓存，不发起网络请求 |
| W-11 | P1 | 与 App 行为对齐：支持 quick/full 两种编译模式、`.tensorwriting/texmf` 项目本地宏包、newtx 支持文件、biber 组件按需获取 |
| W-12 | P1 | 可选把结果同步写入 `.tensorwriting/preview/`，使 App 打开项目时直接显示最新 PDF |
| W-13 | P2 | `compile --watch`：监听项目文件变化自动重编 |

#### 4.2.2 其他（待细化）

| ID | 优先级 | 需求 |
|---|---|---|
| W-01 | P0 | `status`：检测 TensorWriting 是否安装、是否运行，显示版本、运行时状态与数据位置 |
| W-02 | P1 | `list`：列出最近的写作项目（来自 App 的项目偏好记录） |
| W-03 | P1 | `get <file>`：读取项目内文件内容 |
| W-04 | P2 | 创建文件、向文件追加或替换内容 |
| W-05 | P1 | 在 LaTeX 项目中插入来自 TensorReading 的引文，并更新 `.bib` 文件 |
| W-06 | P2 | 导出项目源码 ZIP |

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
| G-01 | P0 | 退出码约定：0 成功；1 一般错误；2 参数错误；3 所需应用或运行时不可用且无降级路径；4 未找到；5 重复；6 编译失败 |
| G-02 | P0 | 配置：命令行参数 > 环境变量（如 `TENSOR_READING_URL`、`TENSOR_READING_STORAGE`、`TENSOR_WRITING_RUNTIME`）> 配置文件 > 默认值 |
| G-03 | P0 | 不上传任何本地数据。只在两种情况下发起远程请求：用户显式指定 DOI/arXiv/URL 导入时获取元数据；编译时按需下载 TeX 文件（来源与 TensorWriting 相同，`--offline` 可关闭） |
| G-04 | P1 | 人类可读输出（表格）与 JSON 输出二选一；`--quiet`、`--verbose` |
| G-05 | P1 | Shell 补全（PowerShell、bash、zsh） |
| G-06 | P2 | 多语言提示（中文、英文），跟随系统语言 |

## 5. 非功能需求

- **安全**：数据库只读打开；写操作仅经官方 HTTP API；HTTP 仅连接 `127.0.0.1`；不执行用户输入拼接的 SQL（使用参数化查询）。TeX 运行时文件只读使用并校验哈希；下载的 TeX 文件必须通过 SHA-256 或 SHA-512 校验后才写入缓存，写入采用原子重命名。
- **兼容性**：Windows 优先，后续支持 macOS 和 Linux。存储路径与运行时路径解析需处理各平台差异。
- **性能**：本地检索在 10 万条目规模下响应小于 1 秒；`status` 小于 200 ms；缓存命中时，单页文档编译（含引擎初始化）小于 5 秒。
- **稳定性**：应用运行时并发读取不产生锁错误（只读、短连接、必要时重试）。编译在 `worker_threads` 中运行，可超时中止，WASM 异常不影响主进程。App 升级运行时期间，每次编译只读取一次 `active.json`。
- **可测试性**：HTTP 和 SQLite 层可被模拟；提供包含示例数据库的测试夹具；编译测试使用 TensorWriting 仓库的 `release/smoke-assets`，不依赖网络。

## 6. 技术方案（已确定）

| 项 | 选择 | 理由 |
|---|---|---|
| 语言与运行时 | TypeScript，Node.js 22 LTS 及以上 | BusyTeX 的 Emscripten 胶水代码与 WASM 只能在 JS 运行时中执行；可复用 TensorWriting 的编译相关 TS 代码；与 Tauri 前端技术栈一致 |
| CLI 框架 | commander 或 citty | 子命令组织清晰，支持补全 |
| HTTP 客户端 | Node 内置 `fetch` | 无额外依赖 |
| SQLite | `node:sqlite`（Node 22.5+）或 `better-sqlite3` | 只读访问 TensorReading 数据库 |
| MCP | 官方 `@modelcontextprotocol/sdk` | 官方 TS SDK |
| 编译 | 直接加载 TensorWriting 已安装运行时中的 BusyTeX，在 `worker_threads` 中运行 | 已验证可行（见 3.2）；与 App 使用同一编译器，输出一致 |
| `.tar.xz` 解压 | Windows 自带 `tar -xJf`，或 WASM 版 xz 库 | Node 无内置 xz |
| 分发 | npm 包（`npm i -g` / `npx`），可执行文件名 `tensor`；后续可选 Node SEA 单文件可执行包 | 安装简单 |
| 测试 | `node:test` 或 vitest | 通用 |

## 7. 里程碑与验收

| 里程碑 | 验收标准 |
|---|---|
| M0 | 仓库公开，README 与需求文档完成 |
| M1 | CLI 骨架、配置、A-01、G-01、G-02；完成 W-07 到 W-10：在 App 未运行时用 `tensor writing compile` 编译 pdflatex 和 xelatex 项目，缺失宏包自动下载，`--json` 诊断与 App 一致 |
| M2 | 完成 R-01 到 R-05；在应用运行与关闭两种状态下均可检索并读取条目 |
| M3 | 完成 R-08、R-09、R-10、W-11；重复导入被正确识别；PDF 能成功附加；biber 项目可编译 |
| M4 | 完成 A-02、A-03、A-04；在至少一个 MCP 客户端和一个编码 Agent 中完成"修改 LaTeX、编译、读诊断"和"检索文献、插入引文"的端到端演示 |
| M5 | 完成 W-01 到 W-05、W-12；发布到 npm，提供 Release 与安装文档 |

## 8. 待决问题

1. **编译代码复用与许可证**：TensorWriting（AGPL-3.0）中的编译纯函数（`compile.ts`、`busytexModes.ts`、`engineRequirements.ts`）如何供 CLI（MIT）复用。倾向于抽成 App 与 CLI 共同依赖的共享包（需先去掉对 i18n 的依赖），许可证待定。运行时中的 BusyTeX 文件由 CLI 在运行时从用户安装目录加载，不随 CLI 分发。
2. **TeX 资源缓存是否与 App 共用**：共用（写入 App 运行时目录下的 `online\`）可避免重复下载和重复占用磁盘，但属于第 1 节"不写入应用存储目录"的例外；另一选项是只读 App 缓存、新下载另存，或下载到临时目录并在编译后删除。
3. 运行时未安装时，CLI 是否负责下载安装运行时（可复用 App 的签名校验逻辑），还是只提示用户去 App 中安装。第一版倾向后者。
4. TensorReading HTTP API 是否会有版本号或鉴权变化，CLI 如何做兼容检测。
5. 是否需要支持多个文献库或多个 profile。
6. PDF 全文提取是否由 CLI 实现，还是复用 TensorReading 已有的解析结果。
7. 发布渠道与项目命名（npm 包名、可执行文件名 `tensor` 是否冲突）。
