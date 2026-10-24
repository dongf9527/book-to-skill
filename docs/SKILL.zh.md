# SKILL.md 中文参考译本 —— book-to-skill 技能规格书

> 译本性质：本文件是仓库根目录 `SKILL.md`（上游 `a24e62f`，879 行）的**中文参考译本**，供人类读者理解规格使用。
> **宿主代理生成技能时实际执行的是英文原文**——本译本不进入任何生成的技能包；如有出入，以英文原文为准。
> 翻译约定：① 可执行代码块（bash）与生成模板块保留英文原文（由脚本按行号从原文逐字节拼入，见文末译注⑤），块后附中文结构说明；② 正文叙述全部翻译；③ 路径、命令、环境变量、占位符（如 `<skill_name>`）保持原样；④ 小节标题采用「中文（英文原词）」格式；⑤ 上游笔误如实保留并加译注，不静默修正。
> 关联：阶段一精读记录 [testing/2026-10-24-phase1-gap-closure.md](testing/2026-10-24-phase1-gap-closure.md)（SKILL.md Step 3–11 十要点）· [architecture.zh.md](architecture.zh.md)（整体架构）

---

## 文件头 frontmatter（原文 L1–4）

```yaml
name: book-to-skill
description: "将书籍与文档（PDF、EPUB、DOCX、HTML、Markdown、纯文本、RTF，MOBI/AZW 需借助 Calibre）转换为结构化的 agent 技能，提取其中的框架、思维模型、原则、技巧与反模式。当用户想通过 GitHub Copilot CLI、Amp、Claude Code、Hermes Agent、OpenCode 或 OpenClaw 研读一份文档，想在工作中应用某位作者的框架，或想从一份文件构建可复用的知识库时，使用本技能。"
```

> **译注①**：`description` 是宿主代理做技能发现/匹配的功能性元数据。实际技能包中建议保留英文原文（技能检索通常以英文关键词触发），此处中文仅为便于理解，原文见根目录 `SKILL.md` L3。

## 跨主机说明（原文 L6–21 的 HTML 注释；信息性内容，宿主代理会忽略）

- **兼容的技能根目录**：GitHub Copilot CLI（`~/.copilot/skills`、`~/.agents/skills`、`.github/skills`、`.claude/skills`、`.agents/skills`）、Amp（`.agents/skills`、`~/.config/agents/skills`、`~/.config/amp/skills`）、Claude Code（`~/.claude/skills`）、Hermes Agent（`$HERMES_HOME/skills`、`.hermes/skills`、`.agents/skills`）、OpenClaw（`${OPENCLAW_STATE_DIR:-~/.openclaw}/skills`、`.agents/skills`、`skills/`；`~/.agents/skills` 仅默认状态可用）、OpenCode（`~/.config/opencode/skills`、`~/.claude/skills`、`.opencode/skills`、`~/.agents/skills`、`.agents/skills`）。
- **刻意省略 `allowed-tools` 以保持 agent 中立**：Copilot CLI 用 `shell`/MCP 服务器名，Claude 用 `Bash`/`Read`/`Write`/`Glob`/`Grep`，Amp 额外有 `shell_command`。本技能需要 shell（运行 extract.py）与文件读写——各宿主首次使用时会就此请求授权。
- **参数提示**：`<path-to-document-folder-or-glob>... [skill-name-slug]`

# Book-to-Skill 转换器

把书面知识转化为可落地的 agent 技能——提取的是结构，不是摘要。

## 设计哲学（Philosophy）

书籍中沉淀着专家经验：框架、原则与技巧，往往经年累月才打磨成形。本技能把这些知识提取成 GitHub Copilot CLI、Amp、Claude Code、Hermes Agent、OpenCode、OpenClaw 或其他兼容 agent 可以反复调用的形式。

**提取结构，而非摘要。** 技能不是读书报告。它是一套工具箱：

- 具名框架（有明确应用场景的思维模型）
- 可行动的原则（指导决策的规则）
- 技巧（分步方法）
- 反模式（要避开什么、为什么）
- 作者语气校准（作者如何思考与表达）

**保留作者的精确表述。** 框架的命名往往有特定理由。"The 5 Whys（五问法）"不能替换成"多问几次为什么"。要捕获原文的准确提法。

**分层控制深度。** 简单的书 → 简单的技能。含 10+ 框架的复杂书 → 带参考文件与按需章节的技能。

---

## 运行模式（Modes of Operation）

四条路径。按用户的请求路由：

### 1. 完整转换（默认）

**触发**：用户给出一或多个文档/目录/glob 路径，无特殊指示
**动作**：执行下列全部步骤（Step 0–9）
**输出**：完整技能，含 SKILL.md、chapters/、glossary、patterns、cheatsheet

### 2. 仅分析（Analyze Only）

**触发**：用户说"分析一下""先只提取"，或"生成前我先审一遍"
**动作**：执行 Step 0–3，然后产出结构化提取报告（找到的框架、原则、技巧）。到此停止——**不要**生成技能文件。
**输出**：供用户审阅的分析报告

### 3. 从既有分析生成（Generate from Prior Analysis）

**触发**：用户已有分析笔记，或之前跑过"仅分析"
**动作**：跳过 Step 0–3，以所提供的分析为输入，执行 Step 4–9
**输出**：基于该分析生成的技能文件

### 4. 更新/并入（Update / Fold-in，针对既有技能）

**触发**：用户提供一或多个新源路径，并表示要更新既有技能（指向既有技能文件夹、给出 `SKILLS_HOME` 中已存在的技能 slug，或明确要求更新）
**动作**：执行 Step 0（范围外检查）、Step 1（校验输入）、Step 1.5（识别内容类型）与 Step 2（提取新文件），然后跳到 Step 5（定位/探测既有技能路径），按**更新/并入工作流**把新内容合并进既有技能文件。
**输出**：更新后的既有技能，含新增/修订的章节摘要与合并后的索引/词汇表。

---

## 技能位置（Skill Locations）

本转换器可在多种技能系统中运行。查找本转换器的辅助脚本、或写入生成的书籍技能时，按以下优先顺序选择位置：

1. GitHub Copilot CLI 个人技能：`~/.copilot/skills/`
2. 跨 agent 个人技能（Copilot、Amp、Codex；默认状态下的 OpenClaw）：`~/.agents/skills/`
3. Claude Code 个人技能：`~/.claude/skills/`
4. 项目级 Copilot 技能：`.github/skills/`
5. 项目级 Claude 技能：`.claude/skills/`
6. 项目级 Amp / Copilot / OpenClaw 技能：`.agents/skills/`
7. Amp 全局技能：`~/.config/agents/skills/`
8. Amp 旧版全局技能：`~/.config/amp/skills/`
9. Hermes Agent 个人技能：`$HERMES_HOME/skills/`（默认 `~/.hermes/skills/`）
10. Hermes Agent 项目技能：`.hermes/skills/` 或 `.agents/skills/`
11. OpenClaw 个人技能：`${OPENCLAW_STATE_DIR:-~/.openclaw}/skills/`（活跃状态；`~/.agents/skills/` 仅与默认状态共享）
12. OpenClaw 项目技能：`.agents/skills/` 或 `skills/`
11. OpenCode 个人技能：`~/.config/opencode/skills/` 或 `~/.agents/skills/`
12. OpenCode 项目技能：`.opencode/skills/` 或 `.agents/skills/`

> **译注②**：本列表沿用上游原始编号——原文在此出现两个"11."与两个"12."（OpenClaw 两条与 OpenCode 两条），实际共 14 项，疑为编号笔误。为便于与英文原文对照，译本未重新编号。

对于**生成的**书籍技能，优先使用用户级跨 agent 根目录 `~/.agents/skills/`——一份物理副本即可同时服务各跨 agent 宿主，以及（使用默认状态时的）OpenClaw。Copilot CLI 与 Amp 原生即可发现它；Claude Code 需要 `~/.claude/skills/<skill_name>` 处的符号链接（在 Step 10 创建，规则见 Step 5）。仅当用户明确要求时，才选择宿主私有或项目级根目录。`BOOK_TO_SKILL_SCOPE=project` 或 `personal` 可为自动化场景显式指定作用域；不要仅仅因为两种作用域都存在就强制追问作用域问题。

---
## Step 0 —— 范围外检查（Out-of-scope check）

如果没有提供任何参数，停止并回复：

> "book-to-skill 需要一个受支持的文档路径、文件夹或 glob 模式。用法：`book-to-skill <path-to-document-folder-or-glob>... [skill-name-slug]`"

在整个工作流中：

- **先确定目标根目录。** 在任何既有技能查找之前，先确定本次运行会写到哪里：按 Step 5 的表格与选择规则，取用户所用宿主的个人根目录或项目级根目录——项目级请求使用其项目级根目录，绝不写个人根目录。下文所说"已解析的根目录"（the resolved root）即指该目标。
- 识别输入路径与可选的技能 slug。
- 若最后一个参数不是存在的文件/文件夹、也不是能匹配到文件的 glob，且形如技能 slug（小写连字符、字母数字），则把它当作 `SKILL_NAME`。
- 其余参数全部视为 `INPUT_PATHS` 列表。
- 若任一输入路径本身就是技能目录（含 `SKILL.md` 与 `chapters/` 子目录），或 `SKILL_NAME` 与已解析根目录中的既有技能 slug 匹配，则把本次运行标记为**更新/并入**（Mode 4）。
- **重跑防护。** 开始完整转换的提取之前，按 Step 5 的命名规则推导候选 slug——"作者-概念"形、"书名"形、以及用户给定的 `SKILL_NAME`——并在已解析根目录中检查是否已有匹配。命中即**停止**并询问用户："`<skill-name>` 已存在。请选择：(1) 更新/并入（Mode 4），(2) 校验既有技能的完整性后停止，或 (3) 强制完整重建。"用户选择之前不得重新提取。运行中断后恢复上下文时（网络断开、对话重放或续接），仅当上次运行报告的提取工作目录（其输出中的 `Workdir ->` 路径，或你在 `BOOK_SKILL_WORKDIR` 中设定的路径）**仍完好、且仍与本次输入和选项匹配**时才可复用——目录存在、其 `metadata.json` 可读、所列 `sources` 与本次转换的文件在文件名**与内容指纹**上一一对应（提取器在 `metadata.json` 中为每个源记录 `sha256`；对每个文件按当前状态重算指纹并要求精确匹配——`book_to_skill/utils.py` 中的 `reuse_is_safe()` 就是这一检查的可执行形态）、且其记录的 `extraction_mode` 与本次使用的模式一致（即 Step 2 的"确认提取的确实是你要的那份文档"规则）。若目录缺失（临时目录被清理）、`metadata.json` 不存在、sources 不再匹配（文件名不同或内容指纹变化）、记录的 metadata 中某源没有 `sha256`（指纹机制引入之前记录的，无法确认新鲜度）、或模式不一致——如实说明并改为重新提取，不要续用。拿不准时，先问用户再决定丢弃还是续用。

---

## Step 1 —— 校验输入（Validate input）

确认 `INPUT_PATHS` 中至少有一个受支持的文件、目录或 glob 模式。
对目录与 glob 进行展开，找出匹配的受支持文件（`.pdf`、`.epub`、`.docx`、`.txt`、`.md`、`.markdown`、`.rst`、`.adoc`、`.html`、`.htm`、`.rtf`、`.mobi`、`.azw`、`.azw3`）。

找不到受支持文件时，停止并给出清晰的错误信息。

---

## Step 1.5 —— 识别内容类型（Identify content type）

提取之前，询问用户：

> "这些源文件属于哪类内容？这能帮我选择最合适的提取方法。
>
> 1. **技术类（Technical）**——含代码块、表格、公式、图示（如编程书、学术论文、架构指南）
> 2. **文本为主（Text-heavy）**——以散文为主，几乎没有表格/代码（如管理、效率、叙事非虚构）
> 3. **不确定**——我会用快速方法，若质量受限会提示你"

把回答存为 `BOOK_TYPE`：

- 选项 1 → `BOOK_TYPE=technical`
- 选项 2 → `BOOK_TYPE=text`
- 选项 3 → `BOOK_TYPE=text`

**若 `BOOK_TYPE=technical`**，先告知用户：

> "📐 已选择技术模式——使用 Docling 做结构感知提取（表格、代码块、公式保存为 markdown）。每页约 1.5 秒，较长的源文件预计需要几分钟。现在开始……"

**若 `BOOK_TYPE=text`**，告知：

> "📄 已选择文本模式——对每种文件类型使用最快的可用提取器。纯文本/Markdown/HTML 通常几秒就绪；PDF 在可用时使用 pdftotext。"

---
## Step 2 —— 从源文档提取文本（Extract text）

运行提取脚本，传入输入路径：

```bash
SCRIPT_PATH=""
HERMES_HOME_RESOLVED="${HERMES_HOME:-$HOME/.hermes}"
OPENCLAW_STATE_DIR_RESOLVED="${OPENCLAW_STATE_DIR:-$HOME/.openclaw}"
PROJECT_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || true)"
HERMES_PROJECT_TRUSTED=false
if [ -n "$PROJECT_ROOT" ] && [ "${HERMES_AGENT:-}" = true ] && \
  command -v hermes >/dev/null 2>&1 && \
  command -v python3 >/dev/null 2>&1 && \
  hermes config get skills.trusted_project_dirs --json 2>/dev/null | PROJECT_ROOT="$PROJECT_ROOT" python3 -c 'import json, os, pathlib, sys; root=pathlib.Path(os.environ["PROJECT_ROOT"]).resolve(); sys.exit(not any(pathlib.Path(p).expanduser().resolve() == root for p in json.load(sys.stdin)))' 2>/dev/null
then
  HERMES_PROJECT_TRUSTED=true
fi

CANDIDATES=(
  "$HOME/.copilot/skills/book-to-skill/scripts/extract.py"
  "$HOME/.agents/skills/book-to-skill/scripts/extract.py"
  "$HOME/.claude/skills/book-to-skill/scripts/extract.py"
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills/book-to-skill/scripts/extract.py"
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/book-to-skill/scripts/extract.py
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/*/book-to-skill/scripts/extract.py
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/*/*/book-to-skill/scripts/extract.py
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/*/*/*/book-to-skill/scripts/extract.py
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/*/*/*/*/book-to-skill/scripts/extract.py
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/*/*/*/*/*/book-to-skill/scripts/extract.py
  "$HOME/.config/opencode/skills/book-to-skill/scripts/extract.py"
  "$HERMES_HOME_RESOLVED/skills/book-to-skill/scripts/extract.py"
  "$HERMES_HOME_RESOLVED"/skills/*/book-to-skill/scripts/extract.py
)
if [ "${HERMES_AGENT:-}" != true ]; then
  # Project-local roots are resolved against the git worktree, not the current
  # directory: an agent can be invoked from anywhere inside the project (e.g.
  # `src/nested`) and these roots still have to be found — that is how OpenCode
  # and the other hosts discover project skills. The CWD-relative forms are kept
  # so the probe also works outside a git repository.
  if [ -n "$PROJECT_ROOT" ]; then
    CANDIDATES+=(
      "$PROJECT_ROOT/.github/skills/book-to-skill/scripts/extract.py"
      "$PROJECT_ROOT/.claude/skills/book-to-skill/scripts/extract.py"
      "$PROJECT_ROOT/.agents/skills/book-to-skill/scripts/extract.py"
      "$PROJECT_ROOT/.opencode/skills/book-to-skill/scripts/extract.py"
    )
  fi
  CANDIDATES+=(
    ".github/skills/book-to-skill/scripts/extract.py"
    ".claude/skills/book-to-skill/scripts/extract.py"
    ".agents/skills/book-to-skill/scripts/extract.py"
    "skills/book-to-skill/scripts/extract.py"
    "skills"/*/book-to-skill/scripts/extract.py
    "skills"/*/*/book-to-skill/scripts/extract.py
    "skills"/*/*/*/book-to-skill/scripts/extract.py
    "skills"/*/*/*/*/book-to-skill/scripts/extract.py
    "skills"/*/*/*/*/*/book-to-skill/scripts/extract.py
    "skills"/*/*/*/*/*/*/book-to-skill/scripts/extract.py
    ".opencode/skills/book-to-skill/scripts/extract.py"
  )
  if [ -n "$PROJECT_ROOT" ]; then
    CANDIDATES+=(
      "$PROJECT_ROOT/skills/book-to-skill/scripts/extract.py"
      "$PROJECT_ROOT/skills"/*/book-to-skill/scripts/extract.py
      "$PROJECT_ROOT/skills"/*/*/book-to-skill/scripts/extract.py
      "$PROJECT_ROOT/skills"/*/*/*/book-to-skill/scripts/extract.py
      "$PROJECT_ROOT/skills"/*/*/*/*/book-to-skill/scripts/extract.py
      "$PROJECT_ROOT/skills"/*/*/*/*/*/book-to-skill/scripts/extract.py
      "$PROJECT_ROOT/skills"/*/*/*/*/*/*/book-to-skill/scripts/extract.py
    )
  fi
fi
CANDIDATES+=(
  "$HOME/.config/agents/skills/book-to-skill/scripts/extract.py"
  "$HOME/.config/amp/skills/book-to-skill/scripts/extract.py"
)
if [ "$HERMES_PROJECT_TRUSTED" = true ]; then
  CANDIDATES=(
    "$PROJECT_ROOT/.hermes/skills/book-to-skill/scripts/extract.py"
    "$PROJECT_ROOT/.hermes/skills"/*/book-to-skill/scripts/extract.py
    "$PROJECT_ROOT/.agents/skills/book-to-skill/scripts/extract.py"
    "$PROJECT_ROOT/.agents/skills"/*/book-to-skill/scripts/extract.py
    "${CANDIDATES[@]}"
  )
fi
for candidate in "${CANDIDATES[@]}"
do
  if [ -f "$candidate" ]; then
    SCRIPT_PATH="$candidate"
    break
  fi
done

if [ -z "$SCRIPT_PATH" ]; then
  echo "Could not find scripts/extract.py for book-to-skill" >&2
  exit 1
fi

PYTHON_BIN="${PYTHON_BIN:-python3}"
if ! command -v "$PYTHON_BIN" >/dev/null 2>&1; then
  PYTHON_BIN="python"
fi

"$PYTHON_BIN" "$SCRIPT_PATH" $INPUT_PATHS --mode <BOOK_TYPE> --install-missing ask
```

（上方脚本原样保留。它在多种宿主根目录中探测 `scripts/extract.py`：包含 Hermes 受信项目目录的优先分支（`hermes config get skills.trusted_project_dirs` 校验）、项目级根目录（相对 git worktree 根而非当前目录解析，使 agent 从项目内任意位置调用都能命中）、以及多层嵌套 `skills/` 的 glob 兜底；最后以 `--mode <BOOK_TYPE> --install-missing ask` 调用，`<BOOK_TYPE>` 用 Step 1.5 的取值替换。）

提取开始前，脚本会按检测到的格式检查所需的可选 Python 包。若缺少更好的提取器，它会向用户提示可用的回退方案。非交互会话默认走回退，除非安装模式显式设为 `yes`。

**提示——环境预检：** 运行 `"$PYTHON_BIN" "$SCRIPT_PATH" --check` 可打印按格式的报告：哪些提取器已安装、缺什么、缺的确切安装命令，且不处理任何文件。排查安装或质量问题时有用了。

这会创建一个**每次运行独立**的工作目录——默认 `<tempdir>/book_skill_work-<pid>/`，或你在 `BOOK_SKILL_WORKDIR` 中设定的确切路径——其中包含：

- `full_text.txt` ——所有源文件合并提取的文本，源之间有清晰的视觉分界。
- `metadata.json` ——总体合并大小、词数、页数、token 数、被丢弃的 EPUB 图片数、解析后的 `workdir`，以及逐源处理详情的 `sources` 列表。

运行结束时打印全部三个路径（`Workdir ->`、`Text ->`、`Meta ->`）。**从该输出（或 `metadata.json` 自身的 `workdir` 字段）取路径，不要假定固定位置**——目录名每次运行都不同，这样同一台机器上的并发提取才不会互相覆盖结果。

读取本次运行的 `metadata.json` 查看结果。

**生成任何内容之前，务必确认提取到的确实是你要的那份文档**：检查 `metadata.json` 里的 `filename` / `source_file`，或 `full_text.txt` 首行的 `SOURCE:` 头。若在等待后台运行，就等**它那个**工作目录——轮询共享路径可能读到另一次运行的输出。

---

## Step 2.5 —— 生成前成本预估（Pre-flight cost estimate）

读取本次运行的 `metadata.json`（提取输出中 `Meta ->` 的路径），在**做任何生成之前**向用户出示预估：

```
📖 Sources detected: <total_sources> source(s)
<list each source filename and format from the sources metadata list>
<if images_dropped > 5: warn that N source images were not read>
📄 Combined Pages/Sections: ~<N> | Words: ~<N> | Total tokens: ~<N>K

💰 Estimated token cost (Full Conversion / Update):
   Input  (reading + prompts): ~<N>K tokens
   Output (skill files generated/updated):  ~<N>K tokens
   Total:                           ~<N>K tokens

   Cost: multiply the token counts above by your model's current
   input/output per-1M-token rates (prices and model names change often —
   do not hardcode them; quote today's rate and label it as an estimate).

   ⏱  Estimated time: ~<N> minutes

📁 Files to be generated/updated:
   SKILL.md + chapter files + glossary + patterns + cheatsheet

➡  Proceed with Full Conversion / Update? (or type "analyze only" to preview first)
```

（模板提示行保留英文原文；实际对话语言由宿主与用户决定。模板含义：检测到的源数与格式清单；图片丢弃数 > 5 时提示有 N 张源图未被读取；合并页数/词数/总 token 数；完整转换或更新的 token 成本预估——输入（读取 + 提示开销）与输出（生成的技能文件）分开列；成本按用户模型当前的每百万 token 单价折算；预估耗时；将生成/更新的文件清单；最后请用户确认继续，或输入 "analyze only" 先看预览。）

**估算方法：**

- 输入 token ≈ metadata 的 `estimated_tokens` × 1.3（每章一遍的提示开销）
- 输出 token ≈ 章节数 × 每章预算 + 4,000（SKILL.md）+ 4,500（glossary + patterns + cheatsheet）
  - 每章预算中点按 `BOOK_TYPE` 取（`DEPTH` 在 Step 4 决定，可能上调预算）：`text` ≈ 1,000，`technical` ≈ 1,800。若用户已表明"仅参考"还是"深度研读"，用 Step 7 矩阵中对应行的数值。
- 成本：报出 token 数，乘以用户当前模型的每百万 token 输入/输出单价。**不要硬编码金额**——模型名与价格经常变；若展示金额，标注为估算并注明日期。

等用户确认后再继续。用户若说 "analyze only"，切换到 Mode 2。

---

## Step 2.6 —— 大部头书籍的 REPL 式访问（REPL-style access，> 50k token）

灵感来自递归语言模型（Recursive Language Model, RLM）范式：把 `full_text.txt` 当作可查询的语料，而不是一次性整体读取。把整个文件载入上下文，会烧掉你在生成阶段还需要的预算。

对约 50k token 以上的书，优先使用程序化探针，而不是无上界的 `Read(full_text.txt)`：

```bash
# Size check before any Read
wc -w "$FULL_TEXT_PATH"

# Find chapter offsets without loading the whole file
grep -n -E "^\s*(Chapter|CHAPTER)\s+[0-9]+" "$FULL_TEXT_PATH" | head -40

# Pull only the chapter you need (lines start..end inclusive)
sed -n '<start>,<end>p' "$FULL_TEXT_PATH"

# Verify a framework is actually mentioned before claiming it in SKILL.md
grep -c -i "westrum\|dora" "$FULL_TEXT_PATH"

# Targeted Read with offset/limit avoids dumping the full file
# Read(file_path=full_text.txt, offset=<line>, limit=<lines>)
```

（探针含义：任何 Read 之前先 `wc -w` 查词数；`grep -n` 不载入全文就找到章节偏移；`sed -n '<start>,<end>p'` 只拉取需要的章；在 SKILL.md 里宣称某框架存在之前先用 `grep -c` 验证；用带 offset/limit 的定向 Read 替代整文件读取。）

在 Step 3（结构分析）、Step 7（逐章摘要）与 Step 8（词汇表/模式提取）中都采用这一做法。50k token 以下的书，一次 `Read` 即可。

为什么这很重要：一本 200 页的书约 75k token。每章重读一遍全文（28 遍）要花约 2M 输入 token；用 grep + sed 只拉取相关片段，能让生成成本正比于**产出**而非**源文件**。

---

## Step 3 —— 分析书籍结构（Analyze book structure）

读取提取文本 `full_text.txt` 的前 8,000 字符，识别：

- 书**名**与**作者**
- **章节结构**（找 "Chapter N"、"PART I"、编号标题、目录）
- **核心主题**与题材领域
- 章节的大致数量

然后读取目录（Table of Contents）部分（若存在），把所有章节映射出来。

**若模式为"仅分析"：** 现在就产出提取报告并停止。结构如下：

```
## Extraction Report — <Title>

### Author's Core Frameworks
- **<Framework Name>**: <what it is and when to apply>

### Key Principles
- <Principle>: <actionable rule>

### Techniques & Methods
- <Technique>: <step-by-step or how-to>

### Anti-patterns
- <What to avoid>: <why>

### Suggested Skill Name
`{author-lastname}-{core-concept}` — e.g. `cialdini-influence`

### Chapters Detected
| # | Title | Main Frameworks |
```

（模板字段含义：`<书名>` 的提取报告——作者核心框架（名称 + 是什么 + 何时应用）/ 关键原则（可行动规则）/ 技巧与方法（分步或怎么做）/ 反模式（避免什么 + 为什么）/ 建议技能名 `{author-lastname}-{core-concept}`（如 `cialdini-influence`）/ 检测到的章节表（#、标题、主要框架）。）

---

## Step 4 —— 询问用途（Ask purpose，仅完整转换）

生成之前，询问用户：

> "你想让这个技能帮你做什么？（选一项或多项）
> 1. 工作时应用作者的框架
> 2. 用作者的思维模型思考
> 3. 查阅具体章节与概念
> 4. 以上全部"

用回答来加权 SKILL.md 核心区（Core section）的内容取舍。

**从回答推导 `DEPTH`（不额外追问）：**

- 回答**只有**选项 3（查阅）→ `DEPTH=reference`——精简、快速查找型章节。
- 回答含选项 1、2 或 4 → `DEPTH=study`——更深的章节，带更多演算细节、例子与推理。

`DEPTH` 与 `BOOK_TYPE` 共同决定 Step 7 的每章 token 预算。**不要**再单独问"研读还是查阅"——它在这里推断。（Mode 2/3 跳过 Step 4，默认 `DEPTH=study`。）

---
## Step 5 —— 确定技能名（Determine skill name）

若提供了 `SKILL_NAME`，直接用作技能 slug。
否则提出两个方案，让用户选：

- **按作者-概念**：`{author-lastname}-{core-concept}`（如 `cialdini-influence`、`meadows-systems`）
- **按书名**：书名转小写连字符（如 `designing-data-intensive-apps`）

若该书有鲜明的方法论个性，默认选作者-概念格式。

选定目标技能根目录（`SKILLS_HOME`）。先从用户显式请求或 `BOOK_TO_SKILL_SCOPE` 解析**作用域（scope）**，再探测**宿主（host）**。请求项目级/项目输出 → 选项目级行；请求个人/全局输出 → 选个人行。两者都未请求时，维持既定的个人默认（非 Hermes 宿主为 `~/.agents/skills`）。不要仅仅因为存在项目级根目录就强制追问作用域。选定的根目录在写入前可能仍需宿主授权。

| 宿主 agent | 个人技能根目录 | 项目级根目录 |
|---|---|---|
| **GitHub Copilot CLI** | `~/.agents/skills`（原生发现） | `.github/skills` → `.claude/skills` → `.agents/skills` |
| **Amp** | `~/.agents/skills`（原生发现） | `.agents/skills` |
| **OpenAI Codex** | `~/.agents/skills`（原生发现；跟随符号链接） | `.agents/skills` |
| **Hermes Agent** | `$HERMES_HOME/skills/<category>`（默认 `~/.hermes/skills/<category>`） | `.hermes/skills/<category>` → `.agents/skills` |
| **Claude Code** | `~/.agents/skills` + 从 `~/.claude/skills/<skill_name>` 建符号链接 | `.claude/skills` |
| **OpenClaw** | `${OPENCLAW_STATE_DIR:-~/.openclaw}/skills`（活跃状态；`~/.agents/skills` 仅默认状态） | `.agents/skills` → `skills/` |
| **OpenCode** | `~/.agents/skills`（原生发现；也会读取 `~/.config/opencode/skills` 与 `~/.claude/skills`） | `.opencode/skills` → `.agents/skills` → `.claude/skills` |

> **译注③**：原文中 OpenCode 这一行（L405）出现在表格之外、位于 Hermes 说明段之后，与表格分离；译本将其并回表格以便阅读，行内容未改动。

Hermes Agent 是唯一保留自己个人根目录的宿主：它按类别（category）划分个人技能，不扫描跨 agent 根目录。使用活跃 profile 的 `HERMES_HOME`，并选择与生成技能主题匹配的类别。不要手工构造 profile 路径。若用户选择 Hermes 项目级根目录，生成后运行 `hermes skills trust <project-root>`，并用 `hermes skills list` 验证可发现性；项目受信之前，项目技能不可用。

> **译注④**：上一段在原文 L402 与 L407 逐字重复出现两次，译本合并为一处。

对 OpenClaw，使用活跃状态目录的 `skills/` 根：`${OPENCLAW_STATE_DIR:-~/.openclaw}/skills/`。共享的 `~/.agents/skills` 兼容根仅在 `OPENCLAW_STATE_DIR` 未设置或为默认 `~/.openclaw` 时可被发现；使用非默认状态时，不得声称 OpenClaw 能看到共享根里的安装。生成后用 `openclaw skills list` 验证可发现性。

OpenCode 原生扫描跨 agent 根 `~/.agents/skills`，因此上面的生成技能默认已经覆盖它——无需符号链接、无需信任步骤。它也会读取自己管理的 `~/.config/opencode/skills` 根与 Claude 兼容根 `~/.claude/skills`；仅当用户主动要求时才用 OpenCode 管理的根。项目技能靠从工作目录向上遍历到 git worktree 来发现，因此生成后若技能未出现，开一个新会话。（`~/.cache/opencode/skills` 是 `skills.urls` 的远程技能下载缓存，不是创作根目录——绝不要把生成的技能写进去。）

选择规则（Selection rules）：

1. 个人安装：`SKILLS_HOME` 设为 `~/.agents/skills`（不存在则创建）。一个例外，避免在别人家发明约定：若 `~/.agents/skills` 不存在**且**宿主私有根已有技能，改用私有根，并在报告中说明原因。
2. **Claude Code 不扫描 `~/.agents/skills`**——生成完成后，Step 10 用 `ln -sfn "$HOME/.agents/skills/<skill_name>" "$HOME/.claude/skills/<skill_name>"` 把技能链接过去。
3. **Hermes Agent 个人安装用上表 Hermes 行**，不用跨 agent 根，也不打符号链接。
4. 用户明确要求宿主私有根（`~/.copilot/skills`、`~/.claude/skills`、`~/.config/agents/skills`、`~/.config/amp/skills`）时，照办并跳过符号链接。
5. 用户明确要求项目级输出时，用其宿主的项目级行。
6. 该选择需要知道宿主（项目级输出、Hermes 个人根、OpenClaw 状态根或 Claude Code 符号链接）而你无法判断时，问："你在哪个 agent 里运行——OpenCode、OpenClaw、Hermes Agent、GitHub Copilot CLI、Amp、Codex 还是 Claude Code？"
7. OpenClaw 个人输出用 `${OPENCLAW_STATE_DIR:-$HOME/.openclaw}/skills`。仅当 `OPENCLAW_STATE_DIR` 未设置或等于默认 `$HOME/.openclaw` 时，共享根 `~/.agents/skills` 才是有效的 OpenClaw 目的地；否则用活跃状态根或项目/额外目录。
8. 用户明确要求 OpenClaw 管理的个人根时，用活跃状态根，并用 `openclaw skills list` 验证可发现性。

把 `SKILLS_HOME` 设为选定的根，并检查 `$SKILLS_HOME/<skill_name>/` 是否已存在。Claude Code 下还要检查 `~/.claude/skills/<skill_name>` 是否为**真实目录**（而非符号链接）——可能有一次旧安装在那里；若有，先提议迁移（把该目录移入 `~/.agents/skills/`，并把原路径替换为符号链接）再继续。
若技能已存在，提示用户三选一：

1. **更新/并入**（Mode 4）——把新文件/内容整合进既有技能组件。
2. **覆盖**——删除并从零重建技能。
3. **改名**——追加 `-2` 或换一个自定义 slug。

用户选**更新/并入**时，立即进入 Step 2.5 之后的**更新/并入工作流**一节（跳过 Step 3、4、6、7、8、9）。

---

## Step 6 —— 创建技能目录结构（Create skill directory structure）

```bash
mkdir -p "$SKILLS_HOME/<skill_name>/chapters"
```

---

## Step 7 —— 生成章节摘要（Generate chapter summaries）

**TOKEN 预算规则——关键（自适应）：**

每章预算随 `BOOK_TYPE` 与 `DEPTH` 缩放。技术章节需要容纳代码与表格的空间；研读深度需要容纳演算推理的空间。从下述矩阵取预算：

| | `DEPTH=reference` | `DEPTH=study` |
|---|---|---|
| `BOOK_TYPE=text` | 800–1,200 token | 1,000–1,800 token |
| `BOOK_TYPE=technical` | 1,200–1,800 token | 2,000–3,000 token |

- 这些是每文件的目标值，不是硬上限——内容密的章节可以超一些，内容薄的可以少一些。密度仍胜过长度（质量规则 #3）：绝不为了凑数而注水。
- 文件按需加载，更大的章节只在该章被实际读取时才花 token。
- 在两个格之间拿不准（如混合内容书）时，用较低的预算，深度来自精度而非体量。

**`DEPTH=study` 要靠内容挣得，而不是靠更大的数字。** 标准章节模板（Core Idea → Connects To）的自然落点是一篇 700–900 token 左右的密实散文。要**诚实地**达到研读预算——而不是注水——研读级章节必须增加具体材料：

- 从本章**复现一个演算示例或成品工件**（如示例新闻稿、一段样例对话、一份填好的模板、作者走完一遍的决策过程），放进 `## Worked Example` 小节。这是最大的杠杆，也是学习者回访的主要原因。
- 把每个框架的"How"**展开**为明确的步骤或判据，不要一行带过。
- 给最重要的 1–2 个框架**补一条 "Why it works / failure mode" 简注**。

若某章确实没有演算示例、也难以扩展，就让它落在研读下限之下，不要注水——并在该章的 Core Idea 中注明它偏薄。相比之下，`reference` 深度的章节刻意省略演算示例，只保留可直接用于决策的要点。

对 Step 3 识别出的**每一章/主要小节**：

读取提取文本 `full_text.txt` 的对应部分（用字符偏移或 grep 章节标题定位）。

按下述结构创建 `$SKILLS_HOME/<skill_name>/chapters/ch<NN>-<slug>.md`。

**按 `BOOK_TYPE` 调整侧重：**

- `technical` → 优先 "Code Examples"、"Reference Tables"、"Commands & APIs" 小节；保留精确语法
- `text` → 优先 "Frameworks Introduced"、"Mental Models"、"Key Takeaways"；空置的技术小节直接省略

```markdown
# Chapter N: <Full Title>

## Core Idea
<1–2 sentences: the single most important thing this chapter teaches>

## Frameworks Introduced
- **<Framework Name>**: <exact formulation — preserve the author's naming>
  - When to use: <specific situation>
  - How: <steps or criteria>

## Key Concepts
- **<Term>**: <precise definition in 1 sentence>
(5–10 most important terms from this chapter)

## Mental Models
<2–4 frameworks or thinking tools. Write as "Use X when Y" or "Think of X as Y">

## Anti-patterns
- **<What to avoid>**: <why it fails>

## Code Examples *(technical books only — omit if BOOK_TYPE=text)*
<!-- Copy the most instructive snippet from the chapter. Preserve indentation exactly. -->
```<language>
<key code example from this chapter>
```
- **What it demonstrates**: <one line>

## Reference Tables *(technical books only — omit if BOOK_TYPE=text)*
<!-- Reproduce any comparison matrix, parameter table, or decision table from the chapter in markdown. -->

## Worked Example *(DEPTH=study only — omit for DEPTH=reference)*
<!-- Reproduce or reconstruct one concrete example the author works through: a
     sample document, a dialogue, a filled-in template, a before/after, or a
     decision walked end-to-end. This is what makes a study chapter worth its
     budget. Keep it faithful to the source; never copy long raw passages —
     reconstruct the example compactly. -->

## Key Takeaways
1. <Actionable insight>
2. <Actionable insight>
3. <Actionable insight>
(3–7 takeaways a practitioner must remember)

## Connects To
- **Ch N**: <why this chapter relates>
- **<Concept>**: <external concept or standard it connects with>
```

**章节模板结构说明**（模板保留英文原文；各小节用途与约束）：

| 模板小节 | 用途与约束 |
|---|---|
| `# Chapter N: <Full Title>` | 章标题，N 为章号 |
| `## Core Idea` | 1–2 句：本章教的最重要的一件事 |
| `## Frameworks Introduced` | 具名框架 + 适用时机（When to use）+ How（步骤或判据）；保留作者的原命名 |
| `## Key Concepts` | 本章 5–10 个最重要术语，各配 1 句精确定义 |
| `## Mental Models` | 2–4 个框架或思维工具，写成 "Use X when Y" / "Think of X as Y" 句式 |
| `## Anti-patterns` | 要避免什么 + 为何失败 |
| `## Code Examples` | 仅技术书（`BOOK_TYPE=text` 时省略）；复制本章最有教学价值的片段，缩进原样保留，附一行"演示了什么" |
| `## Reference Tables` | 仅技术书；用 markdown 复现对比矩阵、参数表或决策表 |
| `## Worked Example` | 仅 `DEPTH=study`（`reference` 省略）；复现或紧凑重构作者走完的一个具体示例（样例文档/对话/填好的模板/前后对照/端到端决策）；忠实于源，绝不整段照抄原文长段落 |
| `## Key Takeaways` | 3–7 条从业者必须记住的可行动要点 |
| `## Connects To` | 与其他章（Ch N）及外部概念/标准的关联 |

---
## Step 8 —— 生成支撑文件（Generate supporting files）

### glossary.md

创建 `$SKILLS_HOME/<skill_name>/glossary.md`：

- 书中所有重要术语，按字母序排列
- 格式：`**Term** — definition (Ch N)`
- 上限 1,500 token

### patterns.md

创建 `$SKILLS_HOME/<skill_name>/patterns.md`：

- 书中所有具体技巧、设计模式、算法
- 格式：`## Pattern Name\n**When to use**: ...\n**How**: ...\n**Trade-offs**: ...`
- 上限 2,000 token

### cheatsheet.md

创建 `$SKILLS_HOME/<skill_name>/cheatsheet.md`：

**这是技能中区分度最高的一层——把它当作推理辅助，而不是关键词清单。** 任何人都能在词汇表里 grep 到一个术语。速查表捕获的是作者的*判断力*：他们会做什么决定、为什么。它就是把"我知道这些词"变成"我会像作者那样行动"的那份文件。

按以下顺序排定优先级：

1. **决策规则**——"遇到 X 就做 Y，因为 Z。"作者应用的 if/then 逻辑，表述到读者无需重读全书就能套用的程度。
2. **决策树 / 流程图**（嵌套列表或小表格）——用于超过两个分支的选择。
3. **权衡矩阵**——按作者在意的维度给相互竞争的选项打分，读者可在自己的约束下做选择。
4. **阈值与默认值**——作者承诺遵循的具体数字、比率或经验法则（如"函数保持在 ~20 行以内"、"错误预算 < 10% 时告警"）。
5. **信号与坏味道（Tells & smells）**——快速识别处境的启发式（"看到 X，你可能正陷入 Y"）。

避免：术语→定义的干条目（那是词汇表的活），以及成段的散文（那是章节的活）。每一行都应帮读者**决定**点什么。

- 格式以紧凑表格与决策规则为主；内容相当于工作时放在手边的一页打印纸。
- 上限 1,200 token。

---

## Step 9 —— 生成主 SKILL.md（Generate the master SKILL.md）

**关键 token 预算：SKILL.md 正文保持在 4,000 token 以内。**
压缩（compaction）从尾部截断——把最重要的内容放最前面。

创建 `$SKILLS_HOME/<skill_name>/SKILL.md`：

```markdown
---
name: <skill_name>
description: "Knowledge base from \"<Full Title>\" by <Author(s)>. Use when applying <author>'s frameworks for <key topics, 3–6 terms>, studying the book, or referencing its concepts."
---

<!-- argument-hint: [topic, framework name, or chapter number] -->

# <Full Title>
**Author**: <Author(s)> | **Pages**: ~<N> | **Chapters**: <N> | **Generated**: <YYYY-MM-DD>

## How to Use This Skill

- **Without arguments** — load core frameworks for reference
- **With a topic** — ask about `replication`, `pricing`, or another indexed topic; I find and read the relevant chapter
- **With chapter** — ask for `ch05`; I load that specific chapter
- **Browse** — ask "what chapters do you have?" to see the full index

When you ask about a topic not covered in Core Frameworks below, I will read
the relevant chapter file before answering.

---

## Core Frameworks & Mental Models
<!-- ~2,000 tokens: the author's most important named frameworks and principles.
     Preserve exact names. Write as "Use X when Y", "Prefer X over Y because Z".
     This is a toolkit, not a summary. -->

<generate 2,000 tokens of the most critical frameworks and insights here>

---

## Chapter Index

| # | Title | Key Frameworks |
|---|-------|----------------|
| [ch01](chapters/ch01-<slug>.md) | <Title> | <framework1>, <framework2> |
| [ch02](chapters/ch02-<slug>.md) | <Title> | <framework1>, <framework2> |
...

## Topic Index

<!-- Alphabetical. Major terms/frameworks → chapter(s) that cover them. -->
- **<Term>** → ch<N>[, ch<N>]
- **<Term>** → ch<N>

## Supporting Files

- [glossary.md](glossary.md) — all key terms with definitions
- [patterns.md](patterns.md) — all techniques and design patterns
- [cheatsheet.md](cheatsheet.md) — quick reference tables and decision guides

---

## Scope & Limits

This skill covers the book content only. For hands-on implementation in your codebase,
combine with project-specific tools. For topics beyond this book, check related skills
or ask the agent directly.
<if images_dropped > 5: state that N source images were not read>
```

**主文件模板结构说明**（模板保留英文原文；各部分用途与约束）：

| 模板部分 | 用途与约束 |
|---|---|
| frontmatter `name` / `description` | 技能标识与发现用描述（引书名 + 作者 + 3–6 个关键主题词） |
| `# <Full Title>` + 元信息行 | 作者 / 约页数 / 章数 / 生成日期（YYYY-MM-DD） |
| `## How to Use This Skill` | 四种用法：无参数（载核心框架）/ 带主题（查索引主题，先读相关章再答）/ 带章号（如 `ch05`，载该章）/ 浏览（问"有哪些章节"） |
| `## Core Frameworks & Mental Models` | 约 2,000 token：最重要的具名框架与原则；保留精确命名；写成 "Use X when Y" / "Prefer X over Y because Z"；是工具箱不是摘要 |
| `## Chapter Index` | 章节表：链接 `chapters/chNN-<slug>.md` + 标题 + 关键框架 |
| `## Topic Index` | 按字母序的"术语 → 章号"映射（`**<Term>** → ch<N>`），agent 导航的关键 |
| `## Supporting Files` | glossary.md / patterns.md / cheatsheet.md 三个链接与一句话说明 |
| `## Scope & Limits` | 声明仅覆盖本书内容，实操需结合项目工具；源图丢弃数 > 5 时如实声明 N 张图片未读 |

---

## Step 9.5 —— 扫描生成的技能（Scan the generated skill）

在报告成功、在另一会话加载技能、或发布之前，运行咨询性安全扫描：

```bash
SKILL_CONVERTER_ROOT="$(cd "$(dirname "$SCRIPT_PATH")/.." && pwd)"
"$PYTHON_BIN" "$SKILL_CONVERTER_ROOT/tools/scan_generated_skill.py" "$SKILLS_HOME/<skill_name>"
```

扫描器以非零码退出时：停下，请人类复查其文件/行级发现。不要悄悄改写生成的文件；在发现被解决或被明确接受之前，不要加载或发布该技能。

---

## Step 10 —— 清理与报告（Cleanup and report）

若宿主是 Claude Code 且 `SKILLS_HOME` 为 `~/.agents/skills`（默认个人安装），用符号链接把技能暴露给 Claude Code——Claude Code 只扫描 `~/.claude/skills`：

```bash
mkdir -p "$HOME/.claude/skills"
LINK="$HOME/.claude/skills/<skill_name>"
TARGET="$HOME/.agents/skills/<skill_name>"
if [ -d "$LINK" ] && [ ! -L "$LINK" ]; then
  CLAUDE_STATUS="skipped-realdir"                 # Step 5 migration declined; leave the old dir
else
  ln -sfn "$TARGET" "$LINK" 2>/dev/null || true
  # Read the link back — do NOT trust that `ln` did what was asked. On Windows/MSYS
  # `ln -s` may COPY instead of link (or need Developer Mode / an elevated shell), and
  # PowerShell/cmd have no `ln` at all. The report must reflect what is on disk, not the
  # fact that the command ran.
  if [ -L "$LINK" ] && [ "$(readlink "$LINK")" = "$TARGET" ]; then
    CLAUDE_STATUS="linked"
  elif [ -e "$LINK" ]; then
    CLAUDE_STATUS="copy"                           # a real file/dir landed instead of a link
  else
    CLAUDE_STATUS="absent"                         # ln unavailable or refused
  fi
fi
```

真实目录守卫是必需的：向已存在的真实目录执行 `ln -sfn` 会把链接**嵌进它里面**（`~/.claude/skills/<skill_name>/<skill_name>`），导致 Claude Code 加载旧副本。若用户拒绝了 Step 5 的迁移，跳过符号链接并在报告中说明——在完成迁移之前，Claude Code 会继续使用旧目录。

**报告任何与链接有关的事之前，先读回链接。** 符号链接是一个"声称"，不是事实：根据 `CLAUDE_STATUS`（磁盘上的实际状态）填写 "Discoverable by" 行，绝不依据"命令已执行"这一事实。**链接缺失或退化成副本时不要硬性失败**——技能本体在 hub 上，其他所有宿主仍能找到它；诚实的报告是"已写入 `~/.agents/skills/<skill_name>`；在链接创建之前 Claude Code 看不到它"，而不是中止。（Windows 线索，未验证：目录联接（junction）——提权 `cmd` 中的 `mklink /J`，或 PowerShell 中的 `New-Item -ItemType Junction`——既不需要开发者模式也不需要符号链接特权；若你尝试它，"读回再报告"的规则不变。）

用户选择宿主私有或项目级根目录时（Step 5 规则 3–4）跳过此步。

然后清理提取工作目录：

```bash
PYTHON_BIN="${PYTHON_BIN:-python3}"
if ! command -v "$PYTHON_BIN" >/dev/null 2>&1; then
  PYTHON_BIN="python"
fi
```

移除**本次运行实际使用的工作目录**——即提取输出中 `Workdir ->` 所指的路径，也是 `metadata.json` 里存的 `workdir`。绝不删除不是你创建的目录：可能还有另一次提取正在旁边运行。

```bash
# WORKDIR is the path this run reported; quote it in case of spaces.
rm -rf "$WORKDIR"
```

若手头还有 metadata 文件，等价做法：

```bash
"$PYTHON_BIN" - "$WORKDIR_METADATA_JSON" <<'PY'
import json
import shutil
import sys
from pathlib import Path

meta_path = Path(sys.argv[1])
workdir = json.loads(meta_path.read_text(encoding="utf-8")).get("workdir")
if workdir:
    shutil.rmtree(workdir, ignore_errors=True)
PY
```

本文件的旧版本删除的是单个固定的 `book_skill_work` 目录。该路径已不再使用，因此那种清理现在是无害的空操作，而不会误删并发运行的输出。

然后向用户报告：

```
✅ Skill created: $SKILLS_HOME/<skill_name>/

📚 Book: <Full Title> — <Author>
📄 Pages: ~<N> | Chapters: <N>

Files generated:
  SKILL.md         — core frameworks + index   (~X tokens)
  chapters/        — <N> chapter summaries     (~X tokens each, ~X total)
  glossary.md      — key terms                 (~X tokens)
  patterns.md      — techniques & patterns     (~X tokens)
  cheatsheet.md    — quick reference           (~X tokens)
  ─────────────────────────────────────────────────────
  Total skill size: ~X tokens (loaded on-demand, not all at once)

💡 Tip: check your agent's session cost/usage command to see actual token usage.

Usage:
  Ask for <skill_name>                  → load core frameworks
  Ask <skill_name> about <topic>        → find and explain a topic
  Ask <skill_name> for ch<N>            → dive into a specific chapter

Discoverable by: <only what is true for the chosen destination — see below>

Somewhere else?  mv ~/.agents/skills/<skill_name> <dest_root>/<skill_name> \
                   && ln -sfn <dest_root>/<skill_name> ~/.claude/skills/<skill_name>

Prompted for permission on every file? That is your host gating writes outside the
working directory. Say "save it in this project" and re-run to write inside it.

Reload (if your agent doesn't auto-detect new skills):
  GitHub Copilot CLI:  /skills reload
  Claude Code:         restart the session
  Amp:                 restart the session
  Hermes Agent:         start a new session
  OpenClaw:             openclaw skills list (new session if watcher disabled)
  OpenCode:             start a new session

Share this skill (optional):
  GitHub repo, installable on any host (Step 11):  say "publish"
  Copilot ecosystem:  gh skill publish $SKILLS_HOME/<skill_name>
```

**报告模板结构说明**：技能创建路径 / 书名与作者 / 约页数与章数 / 生成的文件清单及各自 token 估算（SKILL.md、chapters/、glossary.md、patterns.md、cheatsheet.md，总量按需加载而非一次全载）/ 查看实际 token 用量的提示 / 三种用法示例 / "Discoverable by" 可发现性声明 / 换宿主迁移命令 / 宿主逐文件要权限时的应对（改存项目内并重跑）/ 各宿主重载方式（Copilot CLI `/skills reload`、Claude Code 与 Amp 重启会话、Hermes 与 OpenCode 新会话、OpenClaw `openclaw skills list`）/ 可选发布渠道（Step 11 的 GitHub 发布、Copilot 生态的 `gh skill publish`）。

"Discoverable by" 行必须根据 `CLAUDE_STATUS`（读回结果）填写，绝不依据"ln 执行过"这一事实——对 `~/.agents/skills` 安装：

- `linked` → "Copilot CLI, Amp, Codex（原生）；Claude Code 经符号链接 ~/.claude/skills/<skill_name>"
- `skipped-realdir` → "Copilot CLI, Amp, Codex（原生）；**不含** Claude Code——先迁移 ~/.claude/skills/<skill_name> 处的真实目录"
- `copy` 或 `absent` → "Copilot CLI, Amp, Codex（原生）；**不含** Claude Code——宿主未能创建符号链接（普通副本会在下次更新/并入时漂移）。启用开发者模式 / 手动建链，或直接从 ~/.agents/skills 运行技能"
- Hermes Agent 个人根 → "Hermes Agent（来自 `$HERMES_HOME/skills/<category>`）"；不声称符号链接，也不声称跨 agent——其他宿主不扫描 Hermes 根
- 其他宿主私有或项目级根 → 只列出会扫描该根的宿主；不声称符号链接

"Somewhere else?" 迁移行必须与实际采取的路径一致，绝不能弄坏本次刚创建的符号链接。**`mv` 的目标始终是最终的技能目录 `<dest_root>/<skill_name>`，绝不是 `<dest_root>` 本身。** `mv ~/.agents/skills/mybook ~/.copilot/skills && ln -sfn ~/.copilot/skills ~/.claude/skills/mybook` 看起来有效、实际不对：技能落在 `~/.copilot/skills/mybook`，而链接指向上级根目录，Claude Code 会解析到一个没有 `SKILL.md` 的目录——这条规则要防的正是这种破坏。代入用户实际指定的目的地，让打印出的命令携带真实路径、无需再解释：

- `~/.agents/skills` + 符号链接 → `mv ~/.agents/skills/<skill_name> <dest_root>/<skill_name> && ln -sfn <dest_root>/<skill_name> ~/.claude/skills/<skill_name>`
- 宿主私有根（含 Hermes Agent）→ `mv <src_root>/<skill_name> <dest_root>/<skill_name>`
- 项目级根 → `mv <project_root>/<skill_name> <dest_root>/<skill_name>`

"Prompted for permission on every file?" 行是给"宿主对工作目录之外的写入设门禁"的场景（任何个人作用域根都在 cwd 之外）准备的：目的地已在上方声明，一行修复——重跑时要求项目级根——就放在它旁边。仅在个人作用域安装时保留；用户已选项目级时删除此行。

---
## Step 11 —— 把生成的技能发布到 GitHub（Publish，可选）

Step 10 报告之后、且仅当 Step 9.5 扫描通过时，提议一次：

> "要我把这个技能发布到 GitHub，让任何 Agent Skills 宿主都能用 `npx skills add` 安装吗？（yes / skip）"

用户拒绝则到此为止。前提：`gh` CLI 已认证（用 `gh auth status` 检查）。`gh` 缺失或未认证时，提议安装配置（`brew install gh` 或 https://cli.github.com，然后 `gh auth login`）——或走无 `gh` 路径：用户在 GitHub 网页 UI 创建所选可见性的空仓库，然后你执行下面的 `git init`/`add`/`commit` 命令，再 `git remote add origin <repo-url> && git push -u origin main`。下述可见性规则对网页创建的仓库同样适用。

**可见性是一个独立的封闭问题——绝不推断，绝不从早前的回答里读出。** 用户接受后，单独提出这个问题，并要求一个词的回答：

> "私有还是公开仓库？请用一个词回答：`private` 或 `public`。"

**回答必须*就是* `public`，而不只是*包含*它——这是硬规则，不是建议。** 除一种情况外，一律带 `--private` 运行 `gh repo create`：唯一例外是对可见性问题的回答恰好是光秃秃的单词 `public`。禁止子串匹配，因为一句关于**源材料许可**的话不是可见性回答——"it's public domain"、"the book is public domain"、"it's publicly available" 描述的都是材料而非仓库，全部落到 `--private`。转述、成句的回答、含糊其辞、沉默、或你自己的推断都**不是**同意：再问一次；回答仍不是光秃秃的单词时，用 `--private` 并在报告中说明。私有仓库以后可以转公开；书籍衍生内容的公开推送无法撤回发布。

**版权门（Copyright gate）——创建仓库前一律适用：** 章节文件是综合摘要而非原文，但仍衍生自源材料。按 README 的版权与合理使用政策：从**第三方版权书籍**生成的技能必须保持**私有**；只有当源是用户本人作品、开放许可内容、或用户明确确认有权公开再分发的材料时，才提供公开选项——并说明适用的是哪种情形。能访问公司内部材料不等于有权披露：来自内部文档的技能保持**私有**，除非用户声明拥有发布权。

用户接受后：

1. 在 `$SKILLS_HOME/<skill_name>/` 内添加仓库 `README.md`（绝不覆盖既有文件）——技能标题、一段描述（"Agent skill generated from *<Title>* by <Author> with [book-to-skill](https://github.com/virgiliojr94/book-to-skill)"）、下述第 3 步的安装命令、文件清单，以及"内容为综合摘要、并非书原文"的说明。
2. 把技能文件夹初始化为 git 仓库并创建远端（默认仓库名 `<skill_name>`；允许用户改——有人偏好 `<skill_name>-skill` 后缀）。**嵌套仓库守卫（Nested-repo guard）：** 先检查技能文件夹是否已位于某个 git 仓库之内（`git -C "$SKILLS_HOME/<skill_name>" rev-parse --show-toplevel`——项目级根如 `.claude/skills/` 下必然如此）。若在，**不要**原地 `git init`：外层仓库会把该文件夹记录为嵌入仓库（gitlink，mode 160000）而没有 `.gitmodules`，外层项目的全新克隆会静默丢失该技能。改为把技能文件夹复制到一个临时目录，在副本中执行下列命令，并告知用户：远端的工作副本是发布出去的仓库，而不是项目本地文件夹。

```bash
cd "$SKILLS_HOME/<skill_name>"
git init -b main
git add -A
git commit -m "Add <skill_name> skill"
gh repo create <repo_name> --private --source . --push
# --private is the default; substitute --public ONLY under the visibility rule above
# (the visibility answer WAS the bare word "public" AND the copyright gate allows it)
```

3. 报告仓库 URL 与跨宿主安装命令：

```
✅ Published: https://github.com/<owner>/<repo_name> (<private|public>)

Install on any Agent Skills host:
  npx skills add https://github.com/<owner>/<repo_name> --skill <skill_name>
```

   触发嵌套仓库守卫、从副本发布时，加一行——该本地文件夹永远不会获得远端，因此"更新/并入时推送"的提议也不会出现在它身上：

```
⚠️  Published from a copy: <skill folder> sits inside another git repository, so it has
    no remote of its own. To publish a later update, re-run Step 11, or clone
    https://github.com/<owner>/<repo_name> and fold new material into the clone.
```

根级 `SKILL.md` 布局正是 `skills` CLI 识别的结构，仓库可直接安装——无需调整结构。嵌套仓库情形之外，本地文件夹仍是本机的活动安装、也是远端的工作副本，后续的更新/并入可以把变更提交并推送到同一远端。

---

## 更新/并入工作流（Update / Fold-in Workflow）

对 `$SKILLS_HOME/<skill_name>/` 的既有技能执行更新/并入时：

### 1. 读取既有技能结构（Read Existing Skill Structure）

读取并解析既有技能的文件：

- 读 `$SKILLS_HOME/<skill_name>/SKILL.md`，解析既有**章节索引（Chapter Index）**、**主题索引（Topic Index）**、元数据（作者、总章数）与**核心框架（Core Frameworks）**。
- 列出 `$SKILLS_HOME/<skill_name>/chapters/` 的全部文件，找出最高章号（如 `ch12`）。
- 读 `$SKILLS_HOME/<skill_name>/glossary.md`、`$SKILLS_HOME/<skill_name>/patterns.md`、`$SKILLS_HOME/<skill_name>/cheatsheet.md`，了解已索引了哪些术语与框架。

### 2. 匹配内容，区分"修订"与"新增"（Match Content & Identify Revisions vs. Additions）

分析本次运行 `full_text.txt`（提取输出中 `Text ->` 的路径）中的新提取文本，判断新内容属于：

- **对既有章节的更新/修订**：若新内容的某节直接更新或扩展既有章节的主题，读该章文件，把新细节合并进去，并重写该文件。
- **新增内容**：若内容引入新的章节、论文或独立小节，在 `chapters/` 下创建**新的章节摘要文件**。编号从既有最高章号之后开始（如既有章节止于 `ch12`，则创建 `ch13-*.md`、`ch14-*.md` 等）。

### 3. 生成或更新章节摘要文件（Generate or Update Chapter Summary Files）

对每个新增或修订的章节：

- 读取新提取文本的对应部分。
- 按 **Step 7** 的格式指南构建摘要。
- 写入/更新 `$SKILLS_HOME/<skill_name>/chapters/` 下的文件。

### 4. 合并支撑文件（Merge Supporting Files）

- **合并 glossary.md**：
  - 读既有 `$SKILLS_HOME/<skill_name>/glossary.md`。
  - 从新内容提取全部新术语与定义（Step 8 词汇表指南）。
  - 合并既有与新术语，按字母序排列。
  - 术语已存在时，把新的章节/来源引用追加到该条目（如 `**Term** — definition (Ch 4, Ch 13)`）。
  - 用合并后的完整字母序列表重写 `$SKILLS_HOME/<skill_name>/glossary.md`。
- **合并 patterns.md**：
  - 读既有 `$SKILLS_HOME/<skill_name>/patterns.md`。
  - 从新内容提取新技巧、算法或模式。
  - 追加新模式，保持格式一致，总长度保持精炼（2,500 token 以内）。
- **合并 cheatsheet.md**：
  - 读既有 `$SKILLS_HOME/<skill_name>/cheatsheet.md`。
  - 提取新的比较规则、决策表或参数指南。
  - 干净地并入速查表结构。

### 5. 重新生成主 SKILL.md（Re-generate the Master SKILL.md）

更新主技能文件 `$SKILLS_HOME/<skill_name>/SKILL.md`：

- **元数据**：章数递增，更新估算页数，酌情加入新源名称。`Generated` 日期更新为当前日期。
- **核心框架**：把新内容中影响最大的思维模型或原则并入（确保整个文件仍低于 4,000 token）。
- **章节索引**：把新章节追加进索引表，链接到新建文件。
- **主题索引**：新主题按字母序合并。既有主题在新章节中也有覆盖时，在该行追加新章链接（如 `- **Topic** → ch05, ch13`）。

### 6. 扫描、清理与报告（Scan, Cleanup, and Report）

文件全部写好并合并后，运行 **Step 9.5**，然后进入 **Step 10** 完成清理，并打印一份定制的更新报告——汇总新增章节、合并的词汇表条目与更新后的索引。若技能文件夹是带远端的 git 仓库（经 **Step 11** 发布），提议提交并推送本次更新。

---

## 质量规则（Quality Rules）

1. **提取结构，而非摘要**——捕获具名框架、精确表述、反模式；不是章节复述
2. **保留作者的精确表述**——"The 5 Whys" ≠ "多问几次为什么"；保留精确命名
3. **密度胜过完备**——1,000 token 的摘要胜过 10,000 token 的原文节选
4. **从业者语气**——写 "Use X when Y"，不要写 "The book explains X（本书解释了 X）"
5. **SKILL.md 前置加载**——压缩保留前 5,000 token；最重要的内容放最前
6. **章节文件按需加载**——在被读取之前不占用技能预算
7. **绝不照抄书籍原文**——永远综合、摘要、提取信号
8. **主题索引至关重要**——它是 agent 导航到正确章节文件的方式

---

## 译注汇总

| # | 位置 | 内容 |
|---|---|---|
| ① | 文件头 | `description` 为功能性元数据，实际技能包建议保留英文原文以获得最佳跨宿主匹配 |
| ② | 技能位置 | 列表沿用上游原始编号：11、12 各出现两次（OpenClaw 与 OpenCode），实际 14 项，疑为上游编号笔误 |
| ③ | Step 5 宿主表 | OpenCode 行在原文中脱离表格、位于段落之后（L405），译本并回表格，内容未改 |
| ④ | Step 5 | Hermes Agent 说明段在原文 L402 与 L407 逐字重复，译本合并为一处 |
| ⑤ | 全文 | 所有代码块（bash 与 markdown 生成模板）由脚本按行号从上游 `a24e62f` 的 SKILL.md 逐字节拼入（共 15 块），未作任何改写；块后所附"结构说明"为译本新增的理解辅助，不是规格的一部分；运行时提示语（blockquote 中的话术）译为中文仅供参考，实际语言由宿主与用户决定 |
| ⑥ | 全文 | 行号引用（如 L405）对应上游 `a24e62f` 的根目录 SKILL.md；上游更新后本译本需重新校对 |
| ⑦ | 译本维护 | 2026-10-24 起本仓库分叉的根目录 SKILL.md 在文件头注释块内新增 2 行中文版互链（指向本译本），位于原 L21 `-->` 之前：引文中 L21 及以后的行号换算到分叉文件需 +2，L20 及以前不变；行号引用仍以上游 `a24e62f` 为锚，不受影响 |
