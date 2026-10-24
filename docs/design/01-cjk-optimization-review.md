# RFC 01 技术评审记录 —— 基于 deep-dive §7 的逐项验证

> 评审对象：[01-cjk-optimization.md](01-cjk-optimization.md)（草案，2026-10-24）
> 评审依据：[core-code-deep-dive.md](../analysis/core-code-deep-dive.md)（§6 hook 先例解剖 / §7 假设验证 / §2 模块级约束）
> 评审日期：2026-10-24 ｜ 评审性质：技术评审（评审人：助手；冻结决定权留所有者）
> **结论：有条件通过 —— 7 项修订（R1–R7）已全部落实为 RFC 01 修订稿 v0.1，2 组假设确认无需改动**

---

## 1. 评审方法

本轮评审不重复 deep-dive 的通读结论，而是做两件事：① 把 deep-dive §7 验证表逐条映射回 RFC 01 的方案段落，标注「成立 / 需修正 / 需补充规格」；② 对评审要引用的关键证据做**现场源码复核**（不依赖 deep-dive 文本转述），复核点包括：`looks_image_only` 在 PDF 分发段的位置（utils.py `.pdf` 分支顶端、`extraction_mode` 分叉**之前**）、`extract_with_docling` 的 `do_ocr = False`（parsers/pdf.py L156）、cli.py 三段式 hook 挂载、`has_toc` 判定表达式（utils.py L986，30k 窗口）、`CJK_CHARS_PER_TOKEN` 单点引用（config.py 定义 → utils.py L103 使用）。评审中发现的 RFC 草案事实错误与规格缺口，以修订项 R1–R7 形式记录，并直接落实到 RFC 修订稿。

## 2. deep-dive §7 假设验证 → RFC 映射总表

| deep-dive §7 结论 | 对应 RFC 段落 | 评审裁定 |
|---|---|---|
| OCR hook 模式成立；插入点建议从「looks_image_only 之后」上移为「wrap 层判定之前」 | §3.1 集成模式 | **需修正** → R1 |
| 「hook 两条模式都要考虑」（`do_ocr=False` 使 technical 同样吃不到 OCR） | §3.1（未提及 technical） | **需补规格** → R2 |
| 代码块重建成立；「重建须在 sanitize 前完成（否则零宽探针被剥）」 | §3.2（位置正确但未钉死） | **需钉死约束** → R3 |
| 大书内存 ×2 峰值（per-source 全文 + 合并全文），OCR 中间产物注意 | §3.1 / §6（未提及） | **需补充** → R4 |
| `reuse_is_safe` 不含提取器版本指纹（现成判据的已知盲区） | §6 / §7（未提及） | **需登记并划界** → R5 |
| ToC 点线增强「改动面一个函数」成立（L986 单表达式已复核） | §3.4 | **正则有缺陷** → R6 |
| token 校准成立（常数单点；两处共用同一函数无漂移风险） | §3.3 | ✅ 确认，无需改 |
| 中文章节检测不动（全项目覆盖最全的语言分支） | §2.1 / §3.4 | ✅ 确认，维持 |
| 回归防线成立（fixture 习惯 + SimpleNamespace 测试 seams 有模板） | §5 | ✅ 确认，微补 |

## 3. 修订意见（R1–R7，已全部落实到修订稿）

### R1（子项目 A）OCR hook 判定上移：wrap 层直判，不走异常路径

**发现**：草案 §3.1 写「当 `looks_image_only()` 为真时，先尝试 rapidocr 逐页 OCR」，但 hook 包在 `extract_single_file` 外层——若等内层感知，唯一手段是捕 `ExtractionError` 后解析错误文本再 OCR，这是对错误消息措辞的脆弱耦合，且每次都要先跑完一次注定失败的探测（前 5 页 pdftotext 探测 + 异常构造）。

**证据**（现场复核）：`utils.py` PDF 分发段中 `if looks_image_only(input_str): raise ExtractionError(...)` 位于 `.pdf` 分支顶端，早于 `extraction_mode == "technical"` 判断；`looks_image_only` 是可导入的模块级函数。

**修订**：hook 在 wrap 层**直接调用** `looks_image_only()`（与内层预检同一实现——单点复用防漂移，与 sanitize/is_invisible_codepoint 的复用先例同构）：为真 → 进 OCR 分支；为假 → 原样委托被包裹函数（文本 PDF 零额外开销）；OCR 失败（未安装/识别为空/异常）→ 委托被包裹函数，落回上游原错误路径（错误信息原样保留，含 `ocrmypdf` 建议）。

### R2（子项目 A）technical 模式行为显式化

**发现**：草案 §3.1 只写了 text 视角。实际上扫描件在 technical 模式同样直接抛错（预检在模式分叉之前），且 `extract_with_docling` 硬编码 `do_ocr = False`（parsers/pdf.py L156，现场复核）——即使预检放过，technical 路径也没有任何 OCR 能力。deep-dive §7 明确提示「hook 两条模式都要考虑」。

**修订**：§3.1 补 technical 模式规格——hook 对两条模式同样生效；OCR 产物是纯文本，**无法**提供 docling 的版面/表格结构，因此 technical 模式命中 OCR 分支时：返回 OCR 文本（优于报错），但 metadata 显式记录 `technical_analysis_unavailable: true`，stdout 如实打印，不把 text 质量伪装成 technical 质量（延续「降级原因人类可读」的信任门哲学）。同时 §3.1 注明：开 docling 自带 OCR（`do_ocr=True`）与「hook 前置 OCR」是互斥路线，v0.1 选后者（deep-dive §6 结论：前者改动更小但重、失去每页原因可观测性）。

### R3（子项目 B）重建时序钉死：必须在 sanitize 之前

**发现**：deep-dive §7 验证代码重建假设时附带一条时序警告：「重建须在 sanitize 前完成（否则零宽探针被剥）」。草案 §3.2 的现有位置（`extract_with_docling()` 返回前调用）**天然满足**，但未写成显式约束——若未来有人把重建搬到 utils 侧（sanitize 之后），特征密度统计面对的将是被清洗改形后的文本，探针字符已被剥掉，触发条件静默失真。

**修订**：§3.2 增加「时序约束（钉死）」小节，并在 `rebuild_code_linebreaks` docstring 中写明：本函数只允许在 docling 成功分支与 sanitize 之间执行。

### R4（子项目 A）OCR 逐页流式 + 内存峰值登记

**发现**：deep-dive 新登记 ①：per-source 结果携带全文直到进程结束、`main` 再拼 `consolidated_text`，大书峰值内存约 ×2。OCR 若把整本页面图像批量转出（300dpi × 数百页）会在这个峰值上再叠一层，本环境已发生过一次磁盘满事故，不应再引入同类风险。

**修订**：§3.1 补「逐页流式」硬性要求：单页转图 → 识别 → 拼行 → 释放图像缓冲，禁止批量载入整本页面图像；§6 风险表加一行，真实样本冒烟时记录实测峰值。

### R5（子项目 A）v0.1 不引入 OCR 缓存；resume 指纹盲区划界

**发现**：deep-dive 新登记 ②：`reuse_is_safe` 的安全判据（workdir 在 + 模式同 + 文件名+sha256 匹配）**不含提取器版本**——docling 升级后 resume 会复用旧结果，这是上游现成判据的已知盲区而非缺陷。RFC 若引入 OCR 中间缓存（引擎/模型会升级），会把盲区放大成正确性问题。

**修订**：§7 范围明确 v0.1 **不引入** OCR 中间结果缓存，OCR 文本走标准 full_text + 既有 resume 语义（盲区不因本 RFC 扩大）；§6 登记：未来若加缓存，指纹必须含 OCR 引擎名 + 模型版本（列为 v0.2 考虑项）。

### R6（子项目 D）ToC 点线正则缺陷：省略号 …… 不匹配 ASCII `\.{3,}`

**发现**：草案 §3.4 自拟正则 `r"^\s*.+\s*\.{3,}\s*[0-9０-９]+\s*$"` 只认 ASCII 点号；而中文目录页的分隔符实际是**省略号 `…`（U+2026）**——草案自拟 fixture 的样例行 `第1章 …… 15` 用的就是 `……`，按草案正则**连自家 fixture 都匹配不上**，该子项目会直接测出失败。

**修订（两轮收敛）**：第一轮改为字符类 `[.\u2026]{3,}` 后做正则执行验证，暴露第二层问题——中文惯用的 `……` 只是 **2 个** U+2026 字符，`{3,}` 阈值仍匹配不上（`………` 三个才匹配）。最终方案：leader = `(?:(?:\.{3}|\u2026{2})[.\u2026]*)`，即「≥3 个 ASCII 点 **或** ≥2 个 `…`，后接点类字符任意混排」。执行验证通过：`第1章 …… 15` / ASCII 点线 / 混排 `…. ` 全部命中；反例（散文行、`Version 2..15`、`A .. 12`、单 `…`）全部拒绝，保守性不降。同时复核确认插入点为 utils.py L986 `has_toc = bool(_TOC_PATTERN.search(text[:30000]))` 单表达式处，「改动面一个函数」的判断成立，且天然继承多源 OR 合并与无 banner 重跑语义（deep-dive §2.4）。

### R7（§4 清单更正）cli.py 并非「无需改动」+ 挂载顺序规格

**发现**：草案 §4 写 cli.py「无需改动（hook 由 `install_ocr_hook()` 在 cli.main 挂载）」——在 `cli.main` 里加挂载调用**本身就是改动**，表述自相矛盾。现场复核 cli.py 三段式：`install_pdf_inspector_hook()` → `utils_main()` → `enrich_pdf_inspector_metadata()`。

**修订**：§4 表 cli.py 行更正为「新增一行 `install_ocr_hook()` 挂载」，并补挂载顺序规格：置于 `install_pdf_inspector_hook()` **之后**——monkey-patch 后装者包裹在外层，扫描件先被 OCR 分支拦截，不消耗 pdf-inspector 探测；文本 PDF 走内层 inspector 快路径不受影响。OCR 失败回退目标是「安装时刻已包裹 inspector 的函数」（组合正确性：OCR hook 不感知、不绕过其他 hook）。§5 测试计划补挂载顺序组合测试。

## 4. 确认项（无需改动）

1. **token 校准方案成立**：常数单点（config.py 定义、utils.py L103 唯一使用），metadata 逐源估算与合并全文估算共用同一函数——改常数两处同步，无漂移风险；`estimate_tokens` 的确定性哲学由 heuristic 默认路径保持，tiktoken 仅可选注入，与 deep-dive §2.1 结论完全一致。
2. **中文章节检测维持不动**：deep-dive §7 确认 `第N章回卷节篇讲` + 中文数字 1-999 + 康熙数字 + 全角 + md 前缀已是全项目覆盖最全的语言分支，v0.1 只补 fixture 锁定、不改判定逻辑，正确。
3. **测试计划与上游锚点对齐**：mock 单测 / fixture 回归 / evals replay / 全量门禁四层结构与 deep-dive §8 的测试锚点速查一致；OCR hook 单测照抄 `SimpleNamespace` 注入先例即可。

## 5. 评审结论与后续

- **技术评审：有条件通过。** R1–R7 已全部落实到 RFC 01 修订稿（状态：草案 → 修订稿 v0.1）；2 组确认项原样保留。
- RFC 01 冻结（进入阶段三开发）仍按 ROADMAP 约定**由所有者确认**；确认后按里程碑 D1（mock 单测与 fixture）→ D5（全量门禁 + 冒烟）执行。
- 本评审未发现动摇「P1 改动面最小、最独立」优先级判断的新事实；R4/R5 均属范围划界而非方案否定。
