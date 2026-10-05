# TensorCLI

[English](README.md) | [简体中文](README.zh-CN.md)

`tensorx` 是 **TensorReading**（文献管理与阅读）和 **TensorWriting**（LaTeX 写作）的命令行。你、你的脚本和 AI Agent（Claude Code、Codex、Copilot 等）可以用它导入文献、生成大纲和知识图谱、编译 LaTeX 工程并拿到 PDF。

本仓库只提供下载地址、使用说明和 Agent Skill。`tensorx` 随 TensorReading 一起安装，不需要单独安装。

## 下载

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

- 命令由桌面应用执行。应用没有运行时，`tensorx` 会自动启动它（加 `--no-launch` 可禁止）。
- **对应应用必须已登录 TensorX 账号**，否则命令会被拒绝（退出码 7）。可以在应用里登录，也可以运行：

```bash
tensorx reading login
tensorx writing login
```

- TensorWriting 需要已安装 TeX Live（在 TensorWriting 的账户菜单中安装）。

## TensorWriting：编译 LaTeX 工程

```bash
tensorx writing compile ./my-paper                 # 编译工程文件夹
tensorx writing compile ./my-paper/main.tex        # 指定主文件
tensorx writing compile ./my-paper -o out/paper.pdf
tensorx writing compile ./thesis --main chapters/main.tex --engine xelatex
tensorx writing open ./my-paper                    # 只在应用中打开，不编译
```

`compile` 会把 TensorWriting 窗口切到该工程，按应用自己的规则选择主文件和引擎，完成编译，然后把 PDF 写到磁盘：

- 默认写在主文件旁边，同名 `.pdf`（如 `main.tex` → `main.pdf`）；
- 用 `-o` 指定其他位置，必须以 `.pdf` 结尾，所在文件夹必须已存在。

| 选项 | 说明 |
|---|---|
| `--main <相对路径>` | 主文件，相当于应用里的“设为主文件” |
| `--engine pdflatex\|xelatex\|lualatex` | 本次编译使用的引擎，不改变工程设置 |
| `--mode full\|quick` | 完全编译（默认）或快速编译 |
| `-o, --output <文件.pdf>` | PDF 输出位置 |
| `--log` | 同时输出完整编译日志 |

编译失败时会列出错误位置（`文件:行: error: 信息`），退出码为 6。缺少的宏包会自动下载，进度显示在 stderr。

## TensorReading：文献、大纲与知识图谱

```bash
# 导入文献（自动查重、获取元数据、下载开放获取 PDF）
tensorx reading import --doi 10.1038/s41586-021-03819-2
tensorx reading import --arxiv 1706.03762 --collection "Transformers" --tag to-read
tensorx reading import --pdf ./paper.pdf --outline      # 导入本地 PDF 并立即生成大纲

# 生成大纲，并抽取进知识图谱
tensorx reading outline <item_key>
tensorx reading outline <key1> <key2> --force           # 已有大纲时重新生成

# 知识库（知识图谱）
tensorx reading kg build                                # 抽取尚未入图谱的文献，再重建图谱
tensorx reading kg status

# 查询
tensorx reading search "deep brain stimulation" --limit 10
tensorx reading get <item_key>                          # 元数据、PDF 路径、大纲、知识图谱摘要
tensorx reading collections
```

| 命令 | 说明 |
|---|---|
| `import` | 需要 `--doi`、`--arxiv`、`--pdf` 或 `--title` 之一。可加 `--collection <名称>`（不存在则新建）、`--tag <标签>`（可重复）、`--outline`（导入后生成大纲）、`--no-download`（不下载 PDF）。文献已存在时退出码为 5 |
| `outline` | 使用应用内的 AI 大纲流程，生成大纲并把该文献写入知识图谱。文献必须有 PDF。会消耗账号积分 |
| `kg build` | 对有大纲但从未抽取的文献做知识图谱抽取，然后重建全局图谱。`--no-extract` 只重建图谱；`--model` 指定模型 |
| `search` | 在标题和摘要中检索，返回 item_key、年份、标题、作者 |
| `get` | 一篇文献的完整信息 |

`item_key` 可以从 `search` 或 `import` 的输出中得到。

## 通用选项与输出

| 选项 | 说明 |
|---|---|
| `--json` | 输出 JSON，错误也是 JSON：`{"ok": false, "error": {"code", "message"}, "exitCode"}` |
| `-q, --quiet` | 不输出进度 |
| `--no-launch` | 应用没有运行时直接失败，不自动启动 |
| `--timeout <秒>` | 等待任务完成的上限，默认 1800 |

进度信息总是写到 stderr，所以 `--json` 的 stdout 可以直接解析。

| 退出码 | 含义 |
|---|---|
| 0 | 成功 |
| 1 | 一般错误 |
| 2 | 参数错误 |
| 3 | 应用或 TeX 运行时不可用 |
| 4 | 未找到（文献、工程、主文件、PDF） |
| 5 | 导入的文献已存在 |
| 6 | 编译失败 |
| 7 | 应用未登录 |

## AI Agent 集成

[`skills/tensorx/SKILL.md`](skills/tensorx/SKILL.md) 是一份 Agent Skill，告诉 Agent 何时、如何调用 `tensorx`。以 Claude Code 为例：

```bash
# macOS / Linux
mkdir -p ~/.claude/skills/tensorx && cp skills/tensorx/SKILL.md ~/.claude/skills/tensorx/
```

```powershell
# Windows PowerShell
New-Item -ItemType Directory -Force "$HOME\.claude\skills\tensorx" | Out-Null
Copy-Item skills\tensorx\SKILL.md "$HOME\.claude\skills\tensorx\"
```

其他支持 Skill 或自定义指令的 Agent，可以直接使用这份文件的内容。

## 安全与隐私

- `tensorx` 只连接本机 `127.0.0.1` 上的应用。应用每次启动都会生成新的随机令牌，写在当前用户的应用数据目录中；没有令牌的请求和来自浏览器的请求都会被拒绝。
- 联网的只有应用原有的功能：获取文献元数据和 PDF、AI 大纲与知识图谱抽取、下载缺失的 TeX 宏包。

## 问题反馈

欢迎在本仓库提交 Issue。

## 许可证

本仓库的文档与 Skill 采用 [MIT](LICENSE) 许可证。
