# book-to-skill 二次开发指南

> 配套评估报告见 [EVALUATION.md](EVALUATION.md)。本指南基于对 master 分支源码的逐模块分析，所有改动点均已核对到具体文件与函数。

---

## 一、快速上手（5 分钟跑通）

```bash
# 1. 依赖自检——告诉你每个格式用什么提取器、缺什么怎么装
python3 -m book_to_skill --check

# 2. 提取（Python 层，产出 full_text.txt + metadata.json）
python3 -m book_to_skill ~/books/my-book.pdf --mode text      # 散文书，秒级
python3 -m book_to_skill ~/books/my-book.pdf --mode technical # 技术书，走 docling

# 3. 完整转换（在 Claude Code / Copilot CLI 等 agent 宿主中）
/book-to-skill ~/books/my-book.pdf my-book-slug

# 4. 查看提取产物（agent 生成 skill 的输入）
cat /tmp/book_skill_work-*/metadata.json

# 5. 运行测试 / 校验生成的 skill
python3 -m pytest tests/ -q
python3 tools/validate_skill.py ~/.agents/skills/<slug>/SKILL.md --lens claude
```

**两种使用形态要分清**：
- **当工具用**：直接跑 `python3 -m book_to_skill`，只做提取，拿到干净文本和结构元数据，喂给你自己的任何 pipeline；
- **当 agent skill 用**：把仓库放进 `~/.claude/skills/book-to-skill`（或 `~/.agents/skills/`、`~/.copilot/skills/`），在 agent 里输入 `/book-to-skill 书.pdf`，agent 会按 SKILL.md 规程走完全部 11 步，产出完整技能包。

## 二、架构导览（二开必读）

```
book_to_skill/
├── cli.py                    # 入口：UTF-8 强制 → 挂 PDF 加速 hook → utils_main() → 元数据回写
├── utils.py        (1695行)  # 核心调度：参数解析/多源展开/章节检测/分发/元数据组装
│   ├── resolve_input_files()      # 文件/目录/glob → 确定性排序的文件列表
│   ├── _match_chapter_number()    # 15+ 语言章节编号正则（中文在这里）
│   ├── detect_structure()         # 数字层→结构层→Part 兜底的三层章节检测
│   ├── extract_single_file()      # 按扩展名分发到 parsers/，失败收敛为 ExtractionError
│   ├── reuse_is_safe()            # sha256 指纹校验（中断恢复契约）
│   └── main()                     # 批处理循环 + metadata.json 组装 ← 大多数二开的必经之路
├── config.py                 # 支持的扩展名表、依赖映射、token 估算比率（CJK=1.5字符/token）
├── dependencies.py           # 声明式依赖分组 DEPENDENCY_GROUPS + --check + 运行时安装
├── sanitize.py               # 安全清洗：Trojan Source/零宽字符/康熙部首折叠
├── pdf_inspector_integration.py  # 可选加速层（monkeypatch hook 模式，二开可模仿）
└── parsers/                  # pdf / epub / docx / html / rtf / calibre / text —— 每个独立可替换
```

理解一个关键事实可以少走 80% 弯路：**Python 层只产出两个文件**（`full_text.txt` 纯文本 + `metadata.json` 结构元数据），"拆章、写摘要、生成 glossary" 全部由 agent 按 `SKILL.md` 规程执行。所以二开分两条线：**改提取逻辑 → 改 Python**；**改生成物的结构与风格 → 改 SKILL.md**。

## 三、五大二开方向：改动点 + 源码解读

### 方向 1：中文书籍优化（推荐首选，差异化最大）

**痛点**：中文书的章节检测已支持（`第N章/回/卷/节/篇/讲`），但 token 估算是近似值，且"目录"页识别只有简单模式；扫描版中文 PDF 的 OCR 路径完全依赖外部 `ocrmypdf`。

**改动点 A — token 估算**（`config.py` L35 + `utils.py` `estimate_tokens()`）：

```python
# config.py — 现在用固定比率近似，可替换为 tiktoken o200k_base（对中文更准）
CJK_CHARS_PER_TOKEN = 1.5

# utils.py L87 — 现状：先按 CJK 字符数估算，剩余部分按英文单词估算
def estimate_tokens(text: str) -> int:
    cjk = len(_CJK_RE.findall(text))
    other_words = len(text.split()) - cjk  # 简化示意，实际实现分开统计
    return int(cjk * CJK_CHARS_PER_TOKEN + other_words * WORDS_PER_TOKEN)
```

**改动点 B — 中文目录识别**（`utils.py` `_TOC_PATTERN` L461）：现有正则已含 `目\s*录|目\s*錄|目\s*次`，但中文书常见的"第1章 …… 15"点线路径未专门处理，可在 `_TOC_PATTERN` 追加点线模式 `\.{2,}\s*\d+$`。

**改动点 C — 扫描版中文 PDF**：在 `utils.extract_single_file()` 的 `looks_image_only()` 判定后（L93-109），集成 PaddleOCR/`rapidocr-onnxruntime` 作为中文 OCR 引擎——只需在 OCR 分支新增一个 parser 调用，不碰主链路。

### 方向 2：Web / API 服务化

把 CLI 包成 FastAPI 服务，供内部系统远程转换文档：

```python
# 新文件 book_to_skill/api.py —— 复用现有分发逻辑，约 60 行即可
from fastapi import FastAPI, UploadFile
from book_to_skill import utils

app = FastAPI(title="book-to-skill API")

@app.post("/extract")
async def extract(file: UploadFile, mode: str = "text"):
    # 1. 落盘到临时目录（复用 config.default_output_dir 的安全建目录逻辑）
    # 2. 调 utils.extract_single_file(path, ext, mode, ...) 拿结构化 dict
    # 3. 连同 full_text 一起返回 JSON —— metadata 字段现成可用
    ...
```

**注意**：`extract_single_file()` 会 `print` 进度到 stdout/stderr，服务化时需在 `utils.main()`（L1499）之下绕过批处理循环直接调用单文件函数，并捕获 `ExtractionError` 映射为 4xx。大文件用后台任务 + 轮询 metadata 的 `workdir` 路径即可。

### 方向 3：独立 CLI 工具（pip install 分发）

仓库已有 `pyproject.toml`（console-script 入口 `book-to-skill = book_to_skill.cli:main`），打包基础是现成的。需要补的是：

1. `pyproject.toml` 声明 optional-dependencies 分组（`[pdf]`、`[epub]`、`[all]`），把 `config.PYTHON_DEPENDENCIES` 的映射搬进去；
2. 加 `--version`（当前从 CHANGELOG 手工同步）；
3. 处理 `scripts/extract.py` 与 `python -m book_to_skill` 双入口的一致性（前者是给 SKILL.md 兼容脚本用的薄 wrapper，勿删）。

### 方向 4：新增输入格式（网页抓取 / Notion 导出）

以"URL 抓取"为例，完整改动链（这是项目里最标准的一条扩展路径）：

```python
# ① book_to_skill/parsers/web.py —— 新解析器，遵守项目隐式契约
def extract_with_trafilatura(url: str) -> str | None:
    """失败返回 None（不抛异常，让 fallback 链继续）；
    安全校验类失败才抛 ExtractionError；打印 'Trying X... OK' 进度。"""
    import trafilatura
    downloaded = trafilatura.fetch_url(url)
    if not downloaded:
        return None
    return trafilatura.extract(downloaded, output_format="markdown", include_links=False)

# ② config.py L40 —— 注册扩展名/URL scheme
# ③ dependencies.py L16 DEPENDENCY_GROUPS —— 声明式加一组（--check 自动遍历显示）
# ④ utils.py extract_single_file() L1343 的 elif 链 —— 加分发分支
# ⑤ tests/test_web_extractor.py —— 照 test_html_boilerplate_extraction.py 的样式写测试
```

Notion 导出的 zip 本质是"目录级 HTML 集合"，现有**多源合并机制**（`/book-to-skill docs/ 文件夹`）已经天然支持，甚至不用写新 parser。

### 方向 5：自定义输出模板（改生成物结构）

产物结构由 `SKILL.md` 的 Step 7-9 决定，改这里即可。例如改成"问答卡片式"：

```markdown
<!-- SKILL.md Step 8 的 glossary.md 生成指令段（L524 附近），改造示例 -->
### glossary.md
Generate one card per term in Q/A form:
| Q (term) | A (one-sentence answer, ≤25 words) | Chapter |
|----------|-------------------------------------|---------|
```

改完用自带的校验器把关：`python3 tools/validate_skill.py SKILL.md --lens claude`（校验 name/description 长度、allowed-tools 与正文中 bash 调用的一致性等 6 家宿主规则）。**若想让 Python 层也产出拆章文件**（而非依赖 agent），插入点是 `utils.main()` L1544（写完 full_text.txt 之后）——可按 `detect_structure()` 返回的章节边界切分落盘。

## 四、还有哪些二开用途（除以上五个方向）

| 用途 | 思路 | 价值 |
|---|---|---|
| **企业私有知识库** | 把内部 ADR/运维手册/onboarding 文档转 skill，发到私有 GitHub org，团队 `npx skills add` 统一安装 | 比 RAG 系统轻 10 倍，无向量库运维 |
| **MCP Server 化** | 把生成的 skill 查询封装成 MCP 工具（`query_book(slug, question)` → 定位章节返回），服务不支持 Agent Skills 标准的 agent | 扩大宿主覆盖面 |
| **CI 文档守护** | GitHub Action 监听 `docs/**` 变更，自动重新 fold-in 更新团队 skill（SKILL.md 的 Mode 4 已支持增量合并） | 知识库永不腐烂 |
| **多语言对照 skill** | 同一本书中/英两版 fold-in 进同一 skill，glossary 生成双语条目 | 外版技术书阅读利器 |
| **课程 AI 助教** | 讲义/课件 PDF → skill，发给学生挂在各自 Claude/Copilot 里 | 每个学生人手一个"助教" |
| **论文簇研究助手** | 一个课题组的相关论文 + 笔记合并成统一 skill，新论文随时 fold-in | 替代"文献综述笔记" |
| **evals 质量回归** | 利用自带 `tools/evals/`（replay/score/manifest），为自己的书建立"章节召回正确率"基准，防止 SKILL.md 改坏 | 二开时的安全网 |

## 五、扩展点速查表

| 要改什么 | 必动文件 → 位置 |
|---|---|
| 新文件格式 | `parsers/新格式.py` → `config.py` SUPPORTED_EXTENSIONS(L40) → `dependencies.py` DEPENDENCY_GROUPS(L16) → `utils.py` extract_single_file() elif 链(L1343) → 新测试 |
| 新章节语言 | `utils.py` L106-450 正则区 + `_match_chapter_number()`(L754) 匹配链 |
| 新宿主 lens | `tools/validate_skill.py` LENSES(L66) + `tests/test_docs_host_lists.py` |
| 新注入扫描规则 | `tools/scan_generated_skill.py` `_CONTENT_RULES`(L35) 加三元组 |
| 新安全清洗规则 | `sanitize.py`（`is_invisible_codepoint` 会被扫描器自动复用，单点防漂移） |
| metadata 新字段 | `utils.py` extract_single_file() 返回 dict(L1406) + main() 组装(L1607) |
| token 估算策略 | `config.py` 两个比率常量 + `utils.estimate_tokens()`(L87) |

## 六、二开工程守则（从源码里读出的约定）

1. **失败一律收敛为 `ExtractionError`**——批量处理"单文件坏不拖垮整批"是测试锁死的契约；
2. **遵守 `"Trying X... OK"` 打印约定**——多个测试断言这个格式，跨 parser 保持一致；
3. **改任何检测逻辑前，先读对应测试的 docstring**——测试用 issue 编号（#91/#237/#272/#274…）锚定行为动机，是最快理解约束的方式；
4. **保持"单点复用防漂移"**——不要复制正则，import 那一份；
5. **保留上游 remote**：`git remote add upstream https://github.com/virgiliojr94/book-to-skill.git`，定期 rebase 合入社区修复（项目迭代很快，228 commits 中近半是近几个月的修复）。
