# TensorCLI

[English](README.md) | [简体中文](README.zh-CN.md)

`tensorx` 是 **TensorReading**（文献管理与阅读）和 **TensorWriting**（LaTeX 写作）的命令行。你、你的脚本，以及任何能运行终端命令的 AI Agent，都可以用它导入文献、生成知识库（大纲与知识图谱）、检索文献，以及编译 LaTeX 工程并拿到 PDF。

本仓库只提供下载地址、使用说明和 Agent Skill。`tensorx` 随 TensorReading 一起安装，不需要单独安装。

## 下载与安装

| 软件 | 下载 | 说明 |
|---|---|---|
| TensorReading | <https://www.tensorx.xin> | 自带 `tensorx` 命令行 |
| TensorWriting | <https://www.tensorx.xin/tensorwriting> | 使用 `tensorx writing` 命令时需要 |

安装 TensorReading 并启动一次后：

- **Windows**：安装目录会自动加入用户 PATH。新开一个终端即可使用 `tensorx`。
- **macOS / Linux**：会创建 `~/.local/bin/tensorx`。如果终端提示找不到命令，把 `~/.local/bin` 加入 PATH。

检查安装：

```bash
tensorx --version
tensorx status
```

## 使用前提

- **需要 TensorX Pro（专业版）会员**。对应应用必须已登录一个 Pro（或企业版）账号：文献命令看 TensorReading 的登录状态，编译命令看 TensorWriting 的登录状态。未登录时退出码为 7，不是 Pro 会员时退出码为 8。可以在应用里登录，也可以运行：

  ```bash
  tensorx reading login     # 在浏览器中打开登录页，登录完成后自动继续
  tensorx writing login
  ```

- 命令由桌面应用执行。应用没有运行时，`tensorx` 会自动启动它；加 `--no-launch` 则直接报错。
- TensorWriting 需要已安装 TeX Live（在 TensorWriting 的账户菜单中安装）。
- TensorReading 和 TensorWriting 可以同时打开，两者使用不同的本地端口，不会冲突。

## 快速开始

```bash
tensorx status                                        # 两个应用是否运行、是否登录、会员等级
tensorx writing compile ./my-paper                    # 编译 LaTeX 工程，PDF 写在主文件旁边
tensorx reading import --arxiv 1706.03762 --kg        # 导入论文，并加入知识库
tensorx reading search "transformer" --limit 5        # 检索文献库
```

## 命令总览

| 命令 | 作用 |
|---|---|
| `tensorx status` | 两个应用的运行状态、版本、登录账号和会员等级（不会启动应用） |
| `tensorx skill [-o <文件>]` | 输出或保存给 AI Agent 的 Skill 说明（不需要应用，也不需要登录） |
| `tensorx reading status` \| `login` | TensorReading 的状态与文献库位置；在浏览器中登录 |
| `tensorx reading search <关键词>` | 在标题和摘要中检索 |
| `tensorx reading get <item_key>` | 一篇文献的元数据、PDF 路径、大纲与知识图谱摘要 |
| `tensorx reading collections` | 分类（文件夹）树 |
| `tensorx reading import …` | 导入文献 |
| `tensorx reading kg build …` | 生成知识库（大纲与知识图谱一起生成） |
| `tensorx reading kg status` | 知识图谱的规模与构建时间 |
| `tensorx writing status` \| `login` | TensorWriting 的状态；在浏览器中登录 |
| `tensorx writing open <工程目录>` | 在 TensorWriting 中打开工程，不编译 |
| `tensorx writing compile [<工程目录>\|<主文件.tex>]` | 编译 LaTeX 工程并输出 PDF |

运行 `tensorx --help` 查看全部选项。

## TensorWriting：编译 LaTeX 工程

```bash
tensorx writing compile                            # 编译当前目录
tensorx writing compile ./my-paper                 # 编译工程文件夹
tensorx writing compile ./my-paper/main.tex        # 指定主文件（工程为它所在的文件夹）
tensorx writing compile ./my-paper -o out/paper.pdf
tensorx writing compile ./thesis --main chapters/main.tex --engine xelatex
tensorx writing open ./my-paper                    # 只在应用中打开，不编译
```

`compile` 会把 TensorWriting 窗口切到该工程，按应用自己的规则选择主文件和引擎，用应用内的编译器完成编译，然后把 PDF 写到磁盘：

- 默认写在主文件旁边，同名 `.pdf`（如 `main.tex` → `main.pdf`）；
- 用 `-o` 指定其他位置，必须以 `.pdf` 结尾，所在文件夹必须已存在。

| 选项 | 说明 |
|---|---|
| `--main <相对路径>` | 主文件，相当于应用里的“设为主文件”，会保存到工程设置 |
| `--engine pdflatex\|xelatex\|lualatex` | 本次编译使用的引擎，不改变工程设置 |
| `--mode full\|quick` | 完全编译（默认）或快速编译 |
| `-o, --output <文件.pdf>` | PDF 输出位置 |
| `--log` | 同时输出完整编译日志 |

编译失败时会列出错误位置（`文件:行: error: 信息`），退出码为 6。缺少的宏包会自动下载，进度显示在 stderr。工程中未保存的修改会先保存再编译。

## TensorReading：文献与知识库

### 导入文献

```bash
tensorx reading import --doi 10.1038/s41586-021-03819-2
tensorx reading import --arxiv 1706.03762 --collection "Transformers" --tag to-read
tensorx reading import --pdf ./paper.pdf --title "论文标题"
tensorx reading import --pdf ./paper.pdf --kg           # 导入后立即加入知识库
```

- 需要 `--doi`、`--arxiv`、`--pdf` 或 `--title` 之一。导入时自动查重、获取元数据，并下载 arXiv 或开放获取的 PDF。
- `--collection <名称>`：放入该分类，不存在则新建。`--tag <标签>`：可重复。`--url <网址>`：记录文献网址。
- `--kg`：导入后加入知识库（生成大纲与知识图谱）。`--no-download`：不下载 PDF。
- 文献已在库中时不会重复添加，退出码为 5，并返回已有文献的 `item_key`。

### 知识库（大纲与知识图谱）

大纲和知识图谱是绑定的，没有单独生成大纲的命令。

```bash
tensorx reading kg build <item_key>                     # 指定文献
tensorx reading kg build <key1> <key2> --force          # 已在知识库中也重新生成
tensorx reading kg build --all                          # 所有尚未加入知识库的文献
tensorx reading kg build --no-extract                   # 只重建知识图谱，不调用 AI
tensorx reading kg status
```

对每篇文献：

- 还没有大纲：从 PDF 一起生成大纲和知识图谱（文献需要有 PDF）；
- 已有大纲但未抽取：只做知识图谱抽取；
- 已在知识库中：跳过，除非加 `--force`。

最后会重建全局知识图谱。必须指定文献或使用 `--all`。生成知识库会调用 AI 服务、消耗账号积分；`--all` 在大文献库上可能涉及很多篇，请先确认。`--model <模型>` 可指定抽取模型。

### 查询

```bash
tensorx reading search "deep brain stimulation" --limit 10
tensorx reading get <item_key>
tensorx reading collections
```

- `search` 在标题和摘要中检索，返回 item_key、年份、标题、作者。`--json` 时为 CSL-JSON，可直接用于生成参考文献。
- `get` 返回元数据、作者、标签、分类、PDF 绝对路径、大纲，以及知识图谱摘要（实体数、关系数、一句话总结）。

`item_key` 可以从 `search` 或 `import` 的输出中得到。

## 通用选项与输出

| 选项 | 说明 |
|---|---|
| `--json` | 输出 JSON，错误也是 JSON：`{"ok": false, "error": {"code", "message"}, "exitCode"}` |
| `-q, --quiet` | 不输出进度 |
| `--no-launch` | 应用没有运行时直接失败，不自动启动 |
| `--timeout <秒>` | 等待任务完成的上限，默认 1800。超时后任务仍会在应用中继续 |

进度信息总是写到 stderr，所以 `--json` 的 stdout 可以直接解析。各命令 JSON 字段的说明见 [Skill](skills/tensorx/SKILL.md)。

| 退出码 | 含义 |
|---|---|
| 0 | 成功 |
| 1 | 一般错误（含超时、知识库中有文献处理失败） |
| 2 | 参数错误 |
| 3 | 应用或 TeX 运行时不可用 |
| 4 | 未找到（文献、工程、主文件、PDF） |
| 5 | 导入的文献已存在 |
| 6 | 编译失败 |
| 7 | 应用未登录 |
| 8 | 账号不是 Pro 会员 |

## AI Agent 集成

任何能运行终端命令的 AI Agent 都可以使用 `tensorx`。[`skills/tensorx/SKILL.md`](skills/tensorx/SKILL.md) 是一份 Agent Skill，告诉 Agent 何时、如何调用它、怎样读取 JSON 输出和退出码；`tensorx` 也自带同一份内容：

```bash
tensorx skill                                  # 输出 Skill
tensorx skill -o <Skill 目录>/tensorx/SKILL.md  # 保存到指定位置（自动创建文件夹）
```

**让 Agent 自己安装（推荐）**：把下面这段话发给你的 Agent，它知道自己从哪里加载 Skill。

> 请把 tensorx 安装为你的 Skill：
> 1. 运行 `tensorx skill`，阅读这份使用说明；
> 2. 运行 `tensorx skill -o <你加载 Skill 的目录>/tensorx/SKILL.md` 保存它；如果你不支持 Skill，就把说明内容加入你读取的项目指令文件（例如 AGENTS.md）；
> 3. 运行 `tensorx status --json` 确认可以连上 TensorReading 和 TensorWriting；
> 4. 告诉我 Skill 保存在哪里，以及是否需要新开会话才能生效。

**手动安装**：把 SKILL.md 放到 Agent 的 Skill 目录（通常是 `skills/tensorx/SKILL.md`），或把内容粘贴到它读取的指令文件、自定义指令里。TensorReading 工具箱中的 TensorCLI 页面也提供保存和复制按钮。

安装后可以直接对 Agent 说，例如：

- 用 tensorx 编译 ./my-paper，有错误就修好再编译，直到生成 PDF
- 把 arXiv 2404.03425 导入 TensorReading，放进“待读”分类，并加入知识库
- 在我的 TensorReading 文献库里找遥感基础模型相关的论文，把引用加到 refs.bib

## 常见问题

| 现象 | 处理 |
|---|---|
| 终端提示找不到 `tensorx` | 先启动一次 TensorReading，再新开一个终端；macOS / Linux 确认 `~/.local/bin` 在 PATH 中 |
| 退出码 7（未登录） | 在应用中登录 TensorX 账号，或运行 `tensorx reading login` / `tensorx writing login` |
| 退出码 8（不是 Pro） | `tensorx` 仅对 Pro（专业版）及企业版会员开放 |
| 退出码 3，提示 TeX Live 尚未安装 | 在 TensorWriting 的账户菜单中安装 TeX Live |
| 提示应用“启动后没有响应” | 应用版本过旧，不支持 `tensorx`，请更新到最新版 |
| 等待超时 | 首次编译需要下载宏包、知识库任务需要较长时间；用 `--timeout` 延长，任务会在应用中继续完成 |

## 安全与隐私

- `tensorx` 只连接本机 `127.0.0.1` 上的应用。应用每次启动都会生成新的随机令牌，写在当前用户的应用数据目录中；没有令牌的请求和来自浏览器的请求都会被拒绝。
- 联网的只有应用原有的功能：获取文献元数据和 PDF、知识库的 AI 抽取、下载缺失的 TeX 宏包，以及向 TensorX 确认会员等级。

## 问题反馈

欢迎在本仓库提交 Issue。

## 许可证

本仓库的文档与 Skill 采用 [MIT](LICENSE) 许可证。
