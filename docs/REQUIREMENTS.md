# TensorCLI 需求文档

版本：0.3（草案）
状态：规划中，功能待开发

修订记录：

- 0.3：TensorWriting 新增运行时位置文件 `runtime-location.json`（T-01，已实现），W-07 改为优先读取该文件；新增第 8 节"对 TensorWriting 的改动"，写明编译共享包的抽取方案（T-02）。
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
- 不直接写入应用的数据库或存储目录（例外见第 9 节待决问题 2：TeX 资源缓存）。

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
- 运行时位置文件（T-01，TensorWriting 0.1.8 之后的版本提供）：`%APPDATA%\com.tensorwriting.desktop\runtime-location.json`，格式见 8.1。

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
| W-07 | P0 | `runtime`：定位 TensorWriting 已安装的 TeX 运行时（环境变量 `TENSOR_WRITING_RUNTIME` > `runtime-location.json`（见 8.1）> 旧版 App 的默认目录：正式版 `%LOCALAPPDATA%\TensorWriting\texlive`、开发版 `%APPDATA%\com.tensorwriting.desktop\texlive`），读取 `active.json`，校验兼容标识、`layoutVersion` 与清单 SHA-256；输出路径、版本、variant（minimal/full）、缓存大小。未安装时提示在 TensorWriting 中安装，退出码 3 |
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

## 8. 对 TensorWriting 的改动

CLI 的编译功能不要求 App 修改即可工作，但为了不依赖 App 的内部细节，App 侧需配合以下改动。编号 T-xx。

| ID | 内容 | 状态 |
|---|---|---|
| T-01 | App 写出运行时位置文件 `runtime-location.json` | 已实现（TensorWriting 工作区，未提交；随 0.1.8 之后的版本发布） |
| T-02 | 编译相关纯函数抽成 App 与 CLI 共用的共享包 | 方案已定，待实施 |

### 8.1 T-01 运行时位置文件

**位置**：App 数据目录下，Windows 为 `%APPDATA%\com.tensorwriting.desktop\runtime-location.json`。开发版（debug 构建）写 `runtime-location.debug.json`，避免开发版把 CLI 指向开发用的运行时。

**写入时机**：App 启动时（后台线程，不阻塞界面，也覆盖了运行时目录迁移）、安装或升级运行时成功后、删除运行时后。内容不变时不重写文件。

**格式**（`schemaVersion` 1）：

```json
{
  "schemaVersion": 1,
  "layoutVersion": 1,
  "appVersion": "0.1.9",
  "executable": "C:\\Users\\<user>\\AppData\\Local\\TensorWriting\\tensorwriting.exe",
  "root": "C:\\Users\\<user>\\AppData\\Local\\TensorWriting\\texlive",
  "cache": "C:\\Users\\<user>\\AppData\\Local\\TensorWriting\\texlive\\online",
  "status": "installed",
  "runtimeDir": "C:\\...\\texlive\\installed\\205887230a2db771-staging-KY91VS",
  "generation": "205887230a2db771-staging-KY91VS",
  "version": "2026.2.1",
  "variant": "minimal",
  "snapshotId": "texlive2026-20260301",
  "compatibility": "busytex-1.4.0-tl2026"
}
```

| 字段 | 说明 |
|---|---|
| `layoutVersion` | 运行时目录与在线缓存结构的版本。App 改变 `installed/`、`active.json`、`online/` 的结构时递增；CLI 遇到不认识的版本应报错，不猜测 |
| `executable` | App 可执行文件路径，供 `status`、打开项目等功能使用 |
| `root`、`cache` | 运行时存储根目录与在线资源缓存目录 |
| `status` | `installed`、`missing`（未安装）或 `invalid`（`active.json` 校验失败）。只有 `installed` 时才有 `runtimeDir` 到 `snapshotId` 这些字段 |
| `compatibility` | App 要求的运行时兼容标识，未安装时也给出 |

**CLI 使用规则**：

1. 只把它当"去哪里找"的指针。编译前仍读取 `<root>/active.json`，以其中的 generation 为准，并校验清单（App 可能在写指针前崩溃，或用户手动改过目录）。
2. 文件不存在时（App 版本不高于 0.1.8），退回 W-07 中的默认目录。
3. `schemaVersion` 或 `layoutVersion` 不认识时，报错并提示升级 CLI。

**实现位置**：TensorWriting `src-tauri/src/runtime.rs`（`runtime_location`、`publish_runtime_location`、`write_location`，以及安装与删除流程中的调用），`src-tauri/src/lib.rs`（启动时调用）。单元测试覆盖已安装、未安装、校验失败三种状态，以及内容不变时不重写。

### 8.2 T-02 编译共享包

**目标**：引擎检测、编译选项检测、诊断解析、BusyTeX pipeline 改写只有一份代码，App 与 CLI 共用，规则不会分叉。CLI 编译出的诊断与 App 诊断面板一致。

**抽取范围**：

| 来源（TensorWriting `src/lib/`） | 抽取内容 | 依赖处理 |
|---|---|---|
| `compile.ts` | `detectEngine`、`detectCompileOptions`、`unsupportedFeatures`、`fileAtLines`、`parseDiagnostics`、`parseCompilationDiagnostics`、`directRuntimeResources`、`rewriteBusyTexWorkerBootstrap`；引擎到 BusyTeX driver 的映射（现为私有常量 `DRIVER`，改为导出） | `unsupportedFeatures` 有 3 处、`rewriteBusyTexWorkerBootstrap` 有 1 处调用 `t()`，见下文 i18n 解耦 |
| `busytexModes.ts` | 整个文件：`rewriteBusyTexPipeline`、`executionSource`、`cacheSource` | 无依赖，直接迁移 |
| `engineRequirements.ts` | 整个文件：`findEngineRequirements`、`checkEngineCompatibility`、`ENGINE_LABELS`、`EngineRequirement` | 1 处 `t()` |
| `src/types.ts` | `LatexEngine`、`CompileFile`、`CompileDiagnostic`、`EngineCompatibility` | 迁入共享包，App 的 `types.ts` 改为从共享包重新导出，其余代码不用改 import |

**不抽取**：`BusyTexCompiler`、`CompilerClient`、`useCompile`。它们依赖浏览器（Blob URL、替换全局 `Worker`、资源会话的 `fetch`）和 Tauri `invoke`，CLI 不需要。CLI 自己实现 Node 版宿主（`worker_threads`、浏览器接口模拟、资源服务）。`COMPILE_STOPPED` 是 App 停止编译的内部标记，留在 App。

**i18n 解耦**：共享包不 import App 的 `i18n`。需要文案的函数增加可选参数 `translate?: (key: string, params?: Record<string, unknown>) => string`，默认原样返回 key。App 的翻译 key 本身就是中文原文，所以 App 传入 `t` 后行为完全不变；CLI 可以传入自己的翻译（G-06），不传则输出中文。

**包的形式**：

- 源码放在 TensorWriting 仓库 `packages/compile-core/`（npm workspaces）。规则跟着 App 和它固定的 BusyTeX 版本走，所以 App 仓库是唯一来源。
- 纯 TypeScript、ESM，不依赖 DOM 或 Node API，可在浏览器和 Node 中直接 import。
- 发布到 npm，包名暂定 `@tensorx/writing-compile-core`。导出 `COMPATIBILITY = "busytex-1.4.0-tl2026"`；CLI 编译前检查它与运行时的兼容标识一致（`rewriteBusyTexPipeline` 在 pipeline 格式不匹配时本来就会报错，这里提前给出明确提示）。
- 版本规则：改变诊断格式或检测规则时升 minor 版本；兼容标识变化时升 major 版本。

**实施步骤**：

1. 建立 workspace 包，迁移上述函数与类型。原文件改为从共享包 import；需要改 import 的地方有 `compile.ts`、`useCompile.ts`、`App.tsx`、`DiagnosticsPanel.tsx`，以及测试 `tests/compile.test.mjs`、`tests/compileModes.test.mjs`、`tests/runtime.test.mjs`（`tests/benchDriver.ts` 只用 `BusyTexCompiler`，不受影响）。
2. 按上文做 i18n 解耦，App 调用处传入 `t`。
3. 把纯函数相关的单元测试迁入共享包，用 `node --test` 在 Node 中运行，证明包不依赖浏览器。
4. 检查 Vite 和 `tsc` 能解析 workspace 包，确认 Tauri 构建不受影响。
5. 发布第一个版本，CLI 依赖它。

**验收**：

- App 行为不变：`npm run check`、`test:compile`、`test:diagnostics` 通过，`bench:compile` 无性能回退。
- 共享包可在纯 Node 中 import，测试通过。
- CLI 使用同一组诊断用例，结果与 App 一致。

**不在范围内**：项目扫描与主文件检测目前在 Rust 端（`lib.rs` 的 `scan_files`、`detect_main_file`），不属于这个包。CLI 先用 TS 重新实现，以后另立一项，用共享测试用例保证两边一致。

许可证见第 9 节待决问题 1。

## 9. 待决问题

1. **编译共享包的许可证**：TensorWriting 是 AGPL-3.0，TensorCLI 是 MIT。共享包（T-02）采用什么许可证，需要在实施前确定。运行时中的 BusyTeX 文件由 CLI 在运行时从用户安装目录加载，不随 CLI 分发。
2. **TeX 资源缓存是否与 App 共用**：共用（写入 App 运行时目录下的 `online\`）可避免重复下载和重复占用磁盘，但属于第 1 节"不写入应用存储目录"的例外；另一选项是只读 App 缓存、新下载另存，或下载到临时目录并在编译后删除。
3. 运行时未安装时，CLI 是否负责下载安装运行时（可复用 App 的签名校验逻辑），还是只提示用户去 App 中安装。第一版倾向后者。
4. TensorReading HTTP API 是否会有版本号或鉴权变化，CLI 如何做兼容检测。
5. 是否需要支持多个文献库或多个 profile。
6. PDF 全文提取是否由 CLI 实现，还是复用 TensorReading 已有的解析结果。
7. 发布渠道与项目命名（npm 包名、可执行文件名 `tensor` 是否冲突）。
