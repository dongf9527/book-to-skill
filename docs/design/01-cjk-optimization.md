# RFC 01 —— 中文书籍优化（P1 方向）

> 状态：**草案（待评审）** ｜ 提出日期：2026-10-24 ｜ 优先级：P1
> 关联：[ROADMAP.md](../ROADMAP.md) 阶段二 · [DEVELOPMENT.md](../DEVELOPMENT.md) 方向 1 · [阶段一实测记录](../testing/2026-10-24-phase1-gap-closure.md)
> 评审通过后冻结为 v0.1 范围，进入阶段三开发。

---

## 1. 动机

上游以英文书籍生态为主，中文用户面临四个具体问题。本 RFC 旨在让 book-to-skill 对中文技术书达到与英文书同等的开箱体验，同时**不破坏上游的工程契约**（无强依赖、失败收敛、确定性输出）。

1. **扫描版中文 PDF 完全不可用**：无文字层的 PDF 会在前置检查即被拒绝（`looks_image_only` → 抛错建议 `ocrmypdf`）。中文书扫描版比例高，而 `ocrmypdf` + `tesseract-chi_sim` 的安装门槛对普通用户过高。
2. **docling 代码块换行丢失**（阶段一实测新发现）：简单排版的 6 行代码被压成 1 行 + `\_` 转义噪声，直接损害 technical 模式对中文技术书的核心价值。
3. **中文 token 成本估算偏乐观**：固定比率 `CJK_CHARS_PER_TOKEN = 1.5`（1 token ≈ 1.5 汉字）低于主流 tokenizer 的实际密度，Step 2.5 成本预估会低估约 20%–40%，用户按预估预算转换可能中途断供。
4. **中文行为缺回归防线**：上游 901 测试中中文专项极少，`_CN_CHAPTER` 等已实现的中文能力没有被样例语料锁定，未来改动容易无声退化。

## 2. 现状分析（源码位置已核实）

### 2.1 已良好工作、只补测试的部分

| 能力 | 位置 | 现状 |
|---|---|---|
| 中文章节编号 | `utils.py` `_CN_CHAPTER`（L185） | `第[0-9０-９中文数字]+[章回卷节篇讲]`，中文数字/全角数字/部首数字（`_KANGXI_NUMERAL_TRANS` L175 NFKC 不映射的补丁）全覆盖 |
| 中文 Markdown 标题 | `utils.py` `_MD_CN_HEADING`（L188） | `# 第1章`、`# 一、` 均识别 |
| 中文目录标题 | `utils.py` `_TOC_CJK_PATTERN`（L458） | `目[ \t\u3000]*(录\|錄\|次)`，含全角空格；已实测"第N章"样例书 4 章全中 |
| CJK 安全清洗 | `sanitize.py` `fold_cjk_radicals()` | 康熙部首折叠为正字（NFKC 兼容分解） |

### 2.2 需要改造的部分

**A. token 估算** —— `config.py` L31/L35 + `utils.py` `estimate_tokens()`（L87，`_CJK_RE` L80）
`estimate_tokens` 的 docstring 明示"**Kept dependency-free on purpose** so the same book always yields the same number"——上游刻意无依赖保确定性。任何引入 tokenizer 的方案必须保留这个回退基线。另注意 `tools/discovery_tax.py` 已采用"tiktoken 优先、`words/0.75` 兜底"的双层模式，可照抄该先例。

**B. 扫描版 OCR** —— `utils.py` `looks_image_only()`（L93）与 `extract_single_file()` 的 PDF 分发段（L1270 附近）
现状：前 5 页无文本 → 直接抛 `ExtractionError` 并建议外部 `ocrmypdf`。上游已有的**可选加速层先例**是 `pdf_inspector_integration.py`：monkeypatch hook 包装 `extract_single_file`，不可用即 no-op——OCR 应复用这个模式而不是侵入主链路。

**C. docling 代码块修复** —— `parsers/pdf.py` `extract_with_docling()`（L145）
现状：`PdfPipelineOptions(do_ocr=False, do_table_structure=True)` → `export_to_markdown()`。代码块换行丢失源于 docling 布局模型对"无字体/行距暗示"代码的分类边界，非参数可调项（阶段一已验证无现成开关）。

## 3. 方案

### 3.1 子项目 A：扫描版中文 OCR（rapidocr-onnxruntime）

- **引擎选型**：`rapidocr-onnxruntime`（PP-OCRv4 ONNX 版）。理由：纯 pip 安装（运行时 + 模型共约 25MB，对比 paddlepaddle 500MB+ / tesseract 系统包）、中文识别为第一目标、CPU 即可、失败可干净回退。
- **集成模式**：新文件 `book_to_skill/ocr_integration.py`，仿 `pdf_inspector_integration.py` 的 hook 结构：
  - `install_ocr_hook()` 包装 `extract_single_file`：当 `looks_image_only()` 为真时，先尝试 rapidocr 逐页 OCR，成功则组装文本返回（`extraction_method="rapidocr"`）；失败（未安装/识别为空/异常）则**落回原错误路径**（错误信息不变）。
  - `enrich_ocr_metadata()` 把 OCR 页数、置信度均值等溯源信息原子写入 metadata.json（复用 pdf_inspector 的 tmp + replace 原子写法）。
  - 幂等保护：hook 函数挂 `_book_to_skill_ocr_hook` 属性防重复包装。
- **依赖声明**：`dependencies.py` 新增 `DEPENDENCY_GROUPS["ocr-cjk"]`（`required=False`，`--check` 自动展示）；`config.py` `PYTHON_DEPENDENCIES` 加 `rapidocr_onnxruntime` 映射。安装模式沿用 `--install-missing ask|yes|no` 现有机制。
- **换行策略**：OCR 逐行输出后按"行尾无句末标点且下一行行首非缩进"启发式合并段落——与 PDF 版面重建解耦，保守合并，宁可不并不可错并。

### 3.2 子项目 B：docling 代码块换行重建（后处理，严格触发）

- **位置**：`parsers/pdf.py` 新增 `rebuild_code_linebreaks(text: str) -> tuple[str, int]`，在 `extract_with_docling()` 返回前调用；返回重建行数入 metadata（`code_lines_rebuilt`）。
- **触发条件（双阈值，缺一不可，防误伤散文）**：单行长度 > 120 字符 **且** 代码特征密度 ≥ 3 个/行（特征词表：`def |import |return |= |;|(){}[]|_`、常见关键字、行内 `>=2` 个运算符）。仅处理 docling 输出（text 模式的 pdftotext 已保留换行，不经过此函数）。
- **重建规则**：按特征词表边界做保守断行（`def`/`return`/赋值号前优先），**不做**通用语句解析；转义噪声 `\_` → `_`（仅在触发段内）。
- **诚实降级**：未达触发条件的代码段落维持原样，metadata 记 `code_rebuild_skipped` 计数，让下游可见质量边界。

### 3.3 子项目 C：token 估算校准（常数修正 + 可选精确模式）

- **主改动（无依赖路线，尊重上游哲学）**：`CJK_CHARS_PER_TOKEN` 由 `1.5` 校准为 `1.2`。依据：cl100k_base 中文实测约 0.75–0.9 汉字/token、o200k_base 约 1.0–1.2 汉字/token，取 1.2 覆盖主流区间中位（即 1 token ≈ 1.2 汉字，较现值高估成本 25%，宁可保守）。同步修正 `docs/` 中引用该比率的说明。
- **可选精确模式**：新增 `--token-estimator heuristic|tiktoken`（默认 `heuristic`；`BOOK_TO_SKILL_TOKEN_ESTIMATOR` 环境变量同义）。`tiktoken` 路径用 `o200k_base` 并把实际所用 estimator 写入 metadata（`token_estimator` 字段）——完全照抄 `discovery_tax.py` 的"打印实际所用方法"原则。tiktoken 不进 `PYTHON_DEPENDENCIES` 强制清单，仅在用户显式选择且未安装时提示 `pip install tiktoken`。
- **不改动**：`estimate_tokens` 函数签名与返回语义（全库多处调用，按协作规则 #7 不做顺手重命名）。

### 3.4 子项目 D：中文回归语料 + ToC 点线增强（测试加固）

- **中文 fixture 集**：`tests/fixtures/cjk/` 新增三个确定性样例（沿用上游 fixture 风格）：
  1. `cjk-book-toc.txt` —— 带点线目录页（`第1章 …… 15`）+ 正文 4 章（混合中文数字"第四章"与阿拉伯"第5章"）；
  2. `cjk-book-technical.md` —— 含中文表格标题、代码块、全角标点；
  3. `cjk-scan-mock.pdf` 的替代 —— OCR 不放二进制 fixture，用 mock 单测（见 §5）。
- **断言锁定**：章节计数与方法（numeric/structural）、`has_toc`、token 估算区间、标题样本列表——全部进 `tests/test_cjk_regression.py`。
- **ToC 点线增强**：`utils.py` `_TOC_PATTERN` 旁新增 `_TOC_DOTTED_LINE = re.compile(r"^\s*.+\s*\.{3,}\s*[0-9０-９]+\s*$")`；`detect_structure()` 的 `has_toc` 判定改为"标题行命中 **或** 前 30k 字符内点线条目 ≥ 3"。中英文同等受益，改动面一个函数。

## 4. 改动点清单（文件级）

| 文件 | 改动 | 子项目 |
|---|---|---|
| `book_to_skill/ocr_integration.py` | **新增**：OCR hook + metadata 回写 | A |
| `book_to_skill/dependencies.py` | `DEPENDENCY_GROUPS` 加 `ocr-cjk` 组；`prepare_dependencies()` 加分支 | A |
| `book_to_skill/config.py` | `PYTHON_DEPENDENCIES` 加 rapidocr 映射；`CJK_CHARS_PER_TOKEN` 1.5→1.2；新增 `TOKEN_ESTIMATORS` 常量 | A/C |
| `book_to_skill/utils.py` | `main()` 参数加 `--token-estimator`；`estimate_tokens()` 加 tiktoken 分支（默认不变）；`detect_structure()` 的 `has_toc` 加点线判定；metadata 增加 `token_estimator`/`ocr_*`/`code_lines_rebuilt` 字段 | A/C/D |
| `book_to_skill/parsers/pdf.py` | 新增 `rebuild_code_linebreaks()`；`extract_with_docling()` 尾部调用 | B |
| `book_to_skill/cli.py` | 无需改动（hook 由 `ocr_integration.install_ocr_hook()` 在 cli.main 挂载，仿 pdf_inspector 的 L28/L30） | A |
| `tests/test_cjk_regression.py` | **新增**：中文回归断言集 | D |
| `tests/fixtures/cjk/*` | **新增**：2 个确定性中文样例 | D |
| `tests/test_ocr_integration.py`、`tests/test_docling_code_rebuild.py`、`tests/test_token_estimator.py` | **新增**：mock 单测（不依赖真实模型/网络） | A/B/C |
| `docs/DEVELOPMENT.md` | 方向 1 小节补"实现中"状态与实测结论 | 全部 |
| `SKILL.md` | **v0.1 不改**（影响所有宿主行为，留给 P4 输出模板方向统一处理） | — |

## 5. 测试计划

1. **mock 单测（CI 稳定，无网络/模型依赖）**：
   - OCR：mock rapidocr 引擎返回预置行序列 → 断言 hook 触发条件（`looks_image_only` 为真才进 OCR）、失败回退路径（引擎缺失/识别为空/抛异常三种）、metadata 字段；
   - 代码重建：正例（长代码行恢复 N 行）、反例（长散文行不触发）、转义修复；
   - token：heuristic 数值锁定（改常数后的黄金值）、tiktoken mock 路径、estimator 字段落盘。
2. **fixture 回归**：`cjk/` 两样例过完整 `detect_structure()` 断言集。
3. **evals 扩展**：仿 `replay_trajectories.json` 增加中文路由轨迹（问题指向"第3章"应命中 ch03 文件），用 `tools/evals/replay.py` 确定性打分。
4. **全量门禁**：`pytest tests/ -q` 901 + 新增全绿；`python3 -m book_to_skill --check` 正常展示 OCR 组。
5. **真实环境冒烟（一次性，不入 CI）**：真实扫描版中文 PDF（用户提供）跑 OCR 链路，记录置信度与耗时到 `docs/testing/`。

## 6. 风险与回退

| 风险 | 等级 | 缓解 / 回退 |
|---|---|---|
| rapidocr 识别质量不达预期（竖排、繁混、低分辨率） | 中 | v0.1 明确定位"及格线可用"；失败路径永远回退上游原报错；真实样本冒烟先行验证再合并 |
| 代码重建误伤散文 | 中 | 双阈值触发 + 反例测试 + `code_lines_rebuilt` 计数可审计；出问题可整体 flag 关闭（`--no-code-rebuild`） |
| token 常数变更引发上游测试失败 | 低 | 先跑全量定位断言依赖旧值的测试，逐个按新黄金值更新并在 docstring 注明校准依据 |
| tiktoken 精确模式引入非确定性 | 低 | 默认关闭；metadata 记录 estimator 与版本；上游"同书同数"哲学由 heuristic 路线保底 |
| 与上游 rebase 冲突 | 低 | 全部为新增文件或新增分支，`elif` 链与 `estimate_tokens` 内部改动极小；改前跑 `git fetch upstream` 确认基线（协作规则 #6） |
| 磁盘/安装负担（本环境已发生一次） | 低 | rapidocr ~25MB，无 torch 依赖；`--check` 明示可选 |

## 7. v0.1 范围冻结

**包含**：§3.1 OCR hook（rapidocr）· §3.2 代码换行重建 · §3.3 常数校准 1.2 + 可选 tiktoken · §3.4 中文 fixture + ToC 点线。
**不含**（显式延期）：`SKILL.md` 行为修改（→P4）· Web/API（→P3）· EPUB 中文增强 · 繁体/竖排专项 · PaddleOCR 引擎。

**里程碑**：D1 mock 单测与 fixture → D2 OCR hook 合入 → D3 代码重建 → D4 token 校准 → D5 全量门禁 + 冒烟 → 标记 v0.1 完成。

## 8. 变更记录

| 日期 | 变更 |
|---|---|
| 2026-10-24 | 初版草案，待评审 |
