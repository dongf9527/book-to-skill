---
description: "book-to-skill 是如何构建的：一个确定性的 Python 提取器，加一个规格驱动的 agent 生成器。涵盖流水线、组件地图，以及两者背后的设计权衡。"
seo_title: "架构 - book-to-skill 如何做提取与生成"
---

# 架构

> 本文是 [`architecture.md`](architecture.md) 的中文翻译，与英文原文保持同步；如有出入，以英文原文为准。译注见文末。

book-to-skill 由两半组成：一个**确定性提取器**（Python）和一个**规格驱动的生成器**（遵循 `SKILL.md` 行事的 agent）。提取器把任意文档变成干净的文本 + 元数据；agent 再把这份产出加工成结构化的技能（skill）。

```
            ┌────────────── 提取器 EXTRACTOR（Python · 确定性）──────────────┐
            │   scripts/extract.py (shim)  →  book_to_skill/                 │
            │     ├─ cli.py · utils.py     CLI 解析 · 多源归并 · 运行器      │
            │     ├─ config.py             支持的扩展名 · 路径 · 依赖映射    │
            │     ├─ dependencies.py       可选依赖探测 · --check 报告       │
            │     ├─ sanitize.py           剥离不可见 / 零宽 Unicode 字符    │
            │     └─ parsers/              pdf · epub · docx · html · rtf ·  │
            │                              calibre · text（择优 · 逐级回退） │
            │                                                                │
            │   输出 → <tempdir>/book_skill_work/                            │
            │     full_text.txt   （全部来源合并输出 · 带来源标记）          │
            │     metadata.json   （页数 · 词数 · token 数 · 章节 · 目录）   │
            └────────────────────────────────────────────────────────────────┘

                                   │
                                   ▼

            ┌─────────── 生成器 GENERATOR（agent · 遵循 SKILL.md）───────────┐
            │   Step 1.5   询问内容类型 → BOOK_TYPE (technical | text)       │
            │   Step 2/2.5 提取 · 成本估算 · 与用户确认                      │
            │   Step 2.6   大部头书 REPL 式探查（grep/sed · 不整体重读）     │
            │   Step 3     分析结构（书名 · 作者 · 章节 · 目录）             │
            │   Step 4     用途 → DEPTH (reference | study)                  │
            │   Step 7     逐章摘要（预算 = BOOK_TYPE × DEPTH）              │
            │   Step 8     术语表 · 模式库 · 速查表（决策层）                │
            │   Step 9/9.5 SKILL.md 核心 + 各类索引                          │
            └────────────────────────────────────────────────────────────────┘

                                   │
                                   ▼

                 <SKILLS_HOME>/<slug>/  ← 安装位置按宿主与作用域选择：
                  ~/.copilot/skills/   GitHub Copilot CLI（个人）
                  ~/.agents/skills/    Copilot CLI 或 Amp（跨 agent · 个人）
                  ~/.claude/skills/    Claude Code（个人）
                  ~/.openclaw/skills/  OpenClaw（个人 · 仅默认状态目录）
                  $HERMES_HOME/skills/<category>/  Hermes Agent（个人）
                  .github/skills/ | .claude/skills/ | .agents/skills/  项目级（任意宿主）
                  .hermes/skills/<category>/                         Hermes 项目级
                  skills/                                           OpenClaw 工作区
                  ~/.config/opencode/skills/                        OpenCode（独立根 · 按需启用）
                  .opencode/skills/                                 OpenCode 项目级
                  作用域：个人 vs 项目级（Step 5 · BOOK_TO_SKILL_SCOPE）
                  SKILL.md         核心框架 + 章节与主题索引（约 4K token）
                  chapters/*.md    按需加载 · 被问到才读取
                  glossary.md      术语
                  patterns.md      技法
                  cheatsheet.md    决策规则 / 决策树 / 权衡 / 特征信号
```

OpenClaw 的共享兼容根目录 `~/.agents/skills` 只在 `OPENCLAW_STATE_DIR` 未设置、或取默认值 `~/.openclaw` 时才会被探测发现；非默认的 state 目录改用它自己生效的 `<state-dir>/skills` 根目录。转换器不会去推断任意的 `skills.load.extraDirs` 配置。

## 设计原则

1. **提取结构，而非摘要** —— 命名框架、决策规则、反模式；绝不照搬原文段落。
2. **编译期优于运行期** —— 导航与结构化只付一次成本；查询时只加载相关章节。参见[性能与成本（英文）](performance.md)。
3. **章节按需加载** —— `SKILL.md` 保持精简；章节文件只在被读取时才消耗 token。
4. **重要内容前置到 `SKILL.md`** —— 最关键的内容放最前面（上下文压缩会从末尾截断）。
5. **优雅降级** —— 每种格式都有标准库兜底；单个坏来源会被跳过，而不是让整体失败。

## 关键组件

| 路径 | 职责 |
|------|------|
| `scripts/extract.py` | 薄入口 shim → `book_to_skill.cli`（保留它是为了让旧调用方式继续可用） |
| `book_to_skill/cli.py`、`utils.py` | CLI 参数解析、多来源解析、章节/目录检测、运行器 |
| `book_to_skill/parsers/` | 每种格式一个模块（`pdf`、`epub`、`docx`、`html`、`rtf`、`calibre`、`text`） |
| `book_to_skill/config.py` | 支持的扩展名、输出路径、依赖映射 |
| `book_to_skill/dependencies.py` | 可选依赖探测 + `--check` |
| `book_to_skill/sanitize.py` | 从提取文本中剥离零宽 / Unicode 标签区字符，并把康熙部首折叠回对应汉字（见下文「安全」一节） |
| `tools/discovery_tax.py` | 度量 token 成本，对比「整本倾倒上下文 / 逐轮翻找」两种访问方式 |
| `tools/validate_skill.py` | 按宿主规则审计生成的 SKILL.md（`--lens claude\|copilot\|amp\|hermes\|openclaw\|opencode`，见译注 ①） |
| `tools/scan_generated_skill.py` | 对生成的技能做提示注入扫描（建议性步骤，见下文「安全」一节） |
| `SKILL.md` | 生成器规格书（Step 0–10 + fold-in 归并工作流） |

## 安全

不可信文档流入 agent 的上下文，随后进入一个日后会被其他 agent 加载的生成技能——这是一条「文档 → 上下文」供应链。加固是分层的：

- **提取期清洗**（`book_to_skill/sanitize.py`）—— 在统计指标、写出 `full_text.txt` 之前，从每个解析器的输出中剥离零宽字符（`U+200B/200C/200D/2060/FEFF`）与 Unicode 标签区（`U+E0000–E007F`），让文档夹带的不可见指令永远到不了 agent。会报告清除数量；若清除后已无可见内容，则拒绝该来源。
- **康熙部首折叠**（`book_to_skill/sanitize.py`）—— 若 PDF 内嵌的字体子集缺少正确的 Unicode 映射，本应是汉字的位置会输出康熙部首（`U+2F00–U+2FDF`），于是中文书会提取出「判断⼒」而不是「判断力」。该区段内每个部首都有 Unicode 兼容分解，因此折叠等价于只对这些码位做 NFKC：其余文本逐字节原样通过。不删除任何内容，所以不计入安全清除总数。康熙部首补充区（`U+2E80–U+2EF3`）刻意不折叠——Unicode 未为它定义分解式，且其中一些条目本来就是普通汉字。
- **DOCX XXE / Billion-Laughs 防护**（`parsers/docx.py`）—— 解析前拒绝任何声明了 DTD 或实体的 XML 部件。
- **子进程参数注入防护** —— 文件路径在传给 `pdftotext` / `pdfinfo` / `ebook-convert` 之前先做绝对化，这样以 `-` 开头的文件名不会被误读成命令行旗标。
- **生成技能扫描**（`tools/scan_generated_skill.py`）—— 生成器中的建议性步骤（Step 9.5），对生成的 `SKILL.md`、`chapters/*.md`、`glossary.md`、`patterns.md`、`cheatsheet.md` 全面扫描：指令覆盖类短语、模型控制标签、残留不可见 Unicode、扩权 frontmatter，以及形似数据外传的内容。报告只给出规则名与文件位置——绝不显示匹配到的文本。
- **CI** —— PR 上运行 CodeQL、Bandit（HIGH 级别设为门禁）、Zizmor，以及依赖 CVE 审查。

## 扩展

- **新增格式** → 新增 `book_to_skill/parsers/<fmt>.py`，在 `config.py` 注册扩展名，在 `dependencies.py` 接入依赖探测，在 `utils.extract_single_file` 添加分支。
- **新增生成行为** → 编辑 `SKILL.md` 中对应的 Step；保持精简，并为改动附上证据（见 CONTRIBUTING.md）。

## 译注

1. 英文原文「关键组件」表中 `tools/validate_skill.py` 出现两行重复条目，两行的 lens 列表并不一致。经核对源码 `tools/validate_skill.py` 的 `LENSES` 字典，实际支持 6 个 lens：`claude | copilot | amp | hermes | openclaw | opencode`，译文已合并为一行。
2. 架构图中的模块名、CLI 旗标与配置名保留英文原文，图内的说明性文字译为中文；图中 token 数、预算等均为上游设计值，未做改动。
3. 「性能与成本」（`performance.md`）暂无中文版，链接指向英文原文。
4. 术语处理：`token`、`lens`、`fold-in`、`frontmatter`、`shim` 等保留英文；「context-dump / discovery loop」译作「整本倾倒 / 逐轮翻找」；「tells」译作「特征信号」。
