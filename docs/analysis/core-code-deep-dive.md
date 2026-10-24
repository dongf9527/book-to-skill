# 核心代码深度分析 — extractor 侧逐模块通读

> 范围：`book_to_skill/` 全部 extractor 源码（约 3570 行 Python）逐模块通读。
> SKILL.md（879 行，生成器侧）已于阶段一缺口补齐时全文精读（见
> `docs/testing/2026-10-24-phase1-gap-closure.md`），本文补齐 extractor 侧。
> 行号基于当前 HEAD（上游 `a24e62f`）。本文是 `EVALUATION.md`（架构级）与
> `DEVELOPMENT.md`（改动点级）之下的第三层：模块级设计与隐藏约束。

## 0. 数据流总览

```
extract.py(shim) → cli.main() → install_pdf_inspector_hook() → utils.main()
  → parse_arguments / resolve_input_files / prepare_output_dir
  → 逐源 extract_single_file()   ← 已被 hook 包裹（PDF text 快路径）
      → 魔数嗅探 → prepare_dependencies → 分格式解析器链
      → sanitize_extracted_text → estimate_tokens → detect_structure
  → 合并写 full_text.txt + metadata.json → enrich_pdf_inspector_metadata()
```

| 模块 | 行数 | 一句话职责 |
|------|-----:|-----------|
| `scripts/extract.py` | 34 | 兼容入口 shim（字节码抑制 + UTF-8 + sys.path） |
| `cli.py` | 35 | hook 安装 → 跑提取 → 回写元数据的三段式 |
| `config.py` | 61 | 路径/常量/扩展名/依赖名单的唯一事实源 |
| `utils.py` | 1695 | 章节检测 + token 估算 + 分发 + 多源合并 + main |
| `parsers/pdf.py` | 208 | pdftotext/pypdf/pdfminer/docling 四链 + 页眉脚清洗 |
| `parsers/epub.py` | 202 | ebooklib + stdlib zipfile 双路径（spine 阅读序） |
| `parsers/rtf.py` | 174 | striprtf + 自研正则兜底（目标组剥离状态机） |
| `parsers/docx.py` | 159 | python-docx + stdlib XML 双路径 + XXE 预检 |
| `parsers/html.py` | 142 | trafilatura → bs4 → stdlib 三级 + 边界重建 |
| `dependencies.py` | 343 | 依赖探测/安装协商/`--check` 报告 |
| `pdf_inspector_integration.py` | 282 | monkey-patch hook（加速器 + 信任门） |
| `sanitize.py` | 241 | 9 组隐形字符清洗 + 康熙部首折叠 |

## 1. 入口三小件

**extract.py** 三个前置职责都写在 import 之前：`sys.dont_write_bytecode`
防止 `__pycache__` 污染已部署的技能目录；stdout/stderr 强制 UTF-8
reconfigure（Windows GBK/cp936 控制台下 ✓/✗ 字形会直接
UnicodeEncodeError）；`sys.path.insert` 保证任意工作目录可导入包。
`cli.py` 里同样的 reconfigure 再做一次——说明走 console script 入口
（不经 shim）时也要安全。

**cli.py** 的主流程是 hook 三段式：`install_pdf_inspector_hook()`（不可用时
no-op）→ `utils_main()` → `enrich_pdf_inspector_metadata()`。注意 hook 对
technical 模式也会安装，但其快路径只对 text 模式生效（见 §6）。

**config.py** 藏着一段重要修复史：工作目录从固定
`$TMPDIR/book_skill_work` 改为 `book_skill_work-<pid>`（L5-21）。旧方案下
两个并发运行后者静默覆盖前者的 `full_text.txt`/`metadata.json`——轮询
`metadata.json` 的 agent 可能在零报错情况下拿**另一本书**的提取结果去生成
技能。目录是旧路径的**兄弟**而非子目录，让旧清理例程"找不到而不是误删在跑
的运行"。另有 `or` 而非 `get()` 默认值的细节：环境变量设为空串会变
`Path("")`（当前目录）并被 chmod 0700。`BOOK_SKILL_WORKDIR` 仍可整体覆盖。

RFC 01 关联：`CJK_CHARS_PER_TOKEN = 1.5`（L35）是 token 校准子项目的唯一
常数锚点；`default_output_dir` 的 per-PID 方案是 OCR 缓存目录设计要对齐的
并发先例。

## 2. utils.py（1695 行，四块）

### 2.1 token 估算（L62-103）

`_CJK_RE`（L80）覆盖：康熙部首 ⼀-⿟、CJK 标点/全角、假名、谚文、兼容
表意、BMP 统一表意、**平面 2/3 整段（U+20000-3FFFF）端到端截取**——注释
明说是为了"未来扩展不会像 Extension H 那样静默漏掉"（#103 修复的 ~1000x
低估同型问题，升一个平面重演）。康熙部首入集是因为某些电子书把正文汉字
整体编码成部首形（⽹/⼤/⼀）。

`estimate_tokens`（L87）公式：无 CJK 时 `words / 0.75`；有 CJK 时
`latin_words / 0.75 + cjk / 1.5`。docstring 明示**刻意无依赖保确定性**
（同一本书永远得到同一个数）——这是 RFC 01 方案 C（默认 heuristic + 可选
tiktoken）必须绕开的约束：可选注入不能改默认路径。

### 2.2 章节检测（L106-994）——extractor 最大子系统

**判定链**：`detect_structure`（L905）先走 numeric 分支（≥2 个去重章号即
采用，ToC 条目与正文标题同号不重复计），否则 structural 分支（ATX + setext
下划线标题，按深度分组取**最浅且 ≥2 个不同标题**的层），两级都空则 Part
兜底；最终取 `max(numeric, structural)`——单个游离罗马数字（书中复现的
论文示例）不足以压制 structural 计数。`chapters_method` 字段随结果上报：
注释原话是"这是全项目唯一曾静默做出的决策"。

**多语言正则的分层策略**（这是上游最容易低估的资产）：

- **组合式**：中文 `第N章回卷节篇讲`（L185，中文数字十百千组合 1-999，
  L691 `_cn_numeral_to_int`）；波斯语词序数 1-34 最长前缀匹配
  （`چهاردهم` 须赢过 `چهارم`，L420）；
- **位置式**：泰/印地/孟加拉/奥里亚/泰米尔/僧伽罗/泰卢固/卡纳达/
  马拉雅拉姆/古吉拉特/旁遮普——数字块各自重映射（`str.maketrans`），
  词形要求后随数字防止屈折形式误报；
- **直读式**：俄/希腊（全大写去变音符 `κεφ[άα]λαιο`）、韩
  `제N장` + `의N` 插入章（法条惯例）+ 韩语专用尾部守卫（无大小写可依）。

**防误报五件套**（每件都有 issue 编号和语料校准记录）：

1. `_HEADING_TAIL`（L120）：编号后须行尾/标点/**大写词**——"Chapter 6
   explores…" 是散文；
2. `_PERIOD_TAIL` + `_is_prose_period_tail`（L131/L737）：句点尾在上下文
   中判断——行独立则保标题，段落续行须像标题（≤8 词）；阈值用 Rust Book
   112 文件实测校准（#237）；
3. `_closed_fence_line_numbers`（L526）：**只认闭合围栏**——未闭合围栏当
   普通文本，"数行代码当散文"远比"丢掉半本书的结构"便宜（CommonMark
   闭合规则：同字符、更长、无 info string）；
4. 纯数字标题 `^([1-9]\d{0,2})\s{2,}(\S)`（L784）要求 ≥2 空格防 "1. Item"
   列表项，且首字符非 diff 标记（`1  + use crate…`）；
5. `_is_non_chapter_section`（L501）：前后佐料整题白名单，但只限惯例
   结构——"Indexing Strategies"“Glossary of Terms" 必须仍是章节。

**数字标题的结构性判据**（L573）刻意不看数字本身：不要求从 1 连续——提取
丢一个标题、章列表从 0 起、多源语料重编号都会断链；改用**中位正文字数
≥200**（实测中位：教程步 ~20 / 文档节 ~500 / 论文节 ~2000 / 书章 ~5000，
L565-568 的注释是这个项目"用数据定阈值"文化的最佳标本）。

**structural 分支细节**：setext 下划线只在紧贴非空标题行且**不短于标题**
时成立（主题分隔线/表格边框/`---` 不误判）；ATX 的 `=` 前缀（AsciiDoc）
靠必需空格与 RST 下划线区分；围栏内标题跳过；数字引导的标题延后按深度
单独裁决。

**与 sanitize 的顺序耦合**：`sanitize_extracted_text` 先剥 U+200C，因此
波斯序数表必须同时注册去 ZWNJ 拼写（L408-414），僧伽罗梵语借词
（连字内嵌 ZWNJ）则永远匹配不到——这是跨模块时序约束的活例子。

### 2.3 分发与多源（L997-1090, L1176-1424）

`extract_single_file` 的错误纪律写在 L1188-1192 注释里：**本函数一切失败
必须抛 ExtractionError**——批量循环只捕这一种，其他异常会中止整批运行
（包括本可成功的源）。魔数嗅探兜底（%PDF → pdf；PK + mimetype → epub；
PK + word/document.xml → docx）。EPUB 链 = ebooklib → stdlib zipfile，
图片 >5 张才提示（散文书封面属正常）；PDF 链 = `looks_image_only` 预检
（扫描件直接报错并给出 ocrmypdf 建议，L1272-1278——**RFC 01 OCR hook 的
原定插入点**）→ technical 走 docling（失败/空即降级 text 链，
`fallback_reason` 三态）→ pdftotext → pypdf → pdfminer。

`reuse_is_safe`（L1101）：中断恢复的安全判据 = workdir 在 + 模式同 +
来源按**文件名 + sha256** 1:1 匹配；size 刻意不比（metadata 只存两位小数，
指纹完全覆盖它）；乱序与改内容给出不同的拒绝原因。注意判据**不含提取器
版本**——docling 升级后 resume 会复用旧提取结果。

### 2.4 输出与合并（L1426-1696）

`prepare_output_dir`（L1426）：共享 tmp 双防——拒绝 symlink（可预测路径
预埋）+ 拒绝他属主目录 + chmod 0700。`main` 里三处容易踩的坑都已被上游
修掉并留注释：写文件 `newline="\n"`（Windows 默认翻译会把 CRLF 变
CRCRLF，读回时破坏 setext 标题）；合并后的结构检测用**无 banner 的原文**
重新跑（`SOURCE` 分隔条里的 `====` 会变幻影 setext 标题，且结果依赖
source 路径长度）；`has_toc` 按源 OR 合并（30k 窗口只看第一个源，
否则答案随输入顺序翻转）。多源 structural 计数取各源之和（#272：重扫
合并语料会选中 H1 书名）。元数据自描述 workdir（消费者可精确清理），
`ensure_ascii=False` 必须配显式 UTF-8 编码（cp1252/LC_ALL=C 下全书提取完
才崩）。长文只检出 1 节时打警告——"检测失败远比一章节的书常见"。

隐藏约束：per-source 结果字典携带 `text` 全文直到进程结束，`main` 又拼
`consolidated_text`——大书峰值内存约 **×2**。RFC 01 若加 OCR 中间产物
需注意此峰值。

## 3. parsers/ 七模块

**text.py**：BOM 表按长度降序（UTF-32 LE `ff fe 00 00` 以 UTF-16 LE
`ff fe` 开头，顺序错则误判）；BOM 声明解码失败**拒绝**降级 cp1252/latin-1
——"BOM 是显式编码声明，静默接受 mojibake 比失败糟"；无 BOM 才走
utf-8 → cp1252 → latin-1 链。

**pdf.py**：`clean_pdftotext` 的页眉页脚判定 = 每页首/末非空行 Counter，
**>半数页**才算样板；单行页只投一票（分部页在 4 页中出 2 次若投两票会被
误删）；样板只在**页缘**删（运行页眉重复节标题时，曾把正文真标题一并删掉）。
罗马页码正则用**规范形状** `(?:xc|xl|l?x{0,3})(?:ix|iv|v?i{0,3})` 而非
字母集——旧写法把 "MIX"/"CIVIL"/"MILD" 当页码删掉（删真文本比留杂数字
更糟）。连字符换行合并 `(\w)-\n(\w)` 的已知局限（"well-known" →
"wellknown"）注释明示"咬到再上词典"。`looks_image_only` 前 5 页无文本即
扫描件签名，无 pdftotext 时 best-effort 返回 False。
**`extract_with_docling` 设 `do_ocr = False`**（L156）——docling 路径本身
不做 OCR，且异常不内捕（交给上层降级）。RFC 01 两条 OCR 路线（外层 hook
前置 ocrmypdf/rapidocr vs. 开 docling 自带 OCR pipeline）的对比点即在此。

**epub.py**：ebooklib 路径按 **spine 阅读序**重排并按 `id()` 去重，剩余
文档兜底追加；stdlib 路径走 container.xml → OPF（属性顺序无关的标签级
正则，兼容显式 `opf:` 前缀）→ manifest id→href → spine idref 真阅读序 →
剩余 .html/.xhtml 安全网；IRI 解链顺序 = `html.unescape → urlsplit →
unquote → normpath`（镜像真 XML 解析器的实体已解码状态）。

**docx.py**：双路径行序**实质不同**——python-docx 把所有表格追加在末尾，
stdlib 路径按文档序（`emit_block` 递归，`w:sdt` 内容控件下钻、`w:p`/
`w:tbl` 不下钻防单元格重计）；安全校验内聚在每个叶子 parser（"恰好一次"
无论走哪条路），且 `except ExtractionError: raise` 显式置于宽 except 之前
——否则 XXE 拒绝会被 `[warn] + None` 吞掉变成静默降级。校验本身是 5 种
编码逐个 decode 后字符串扫描 `<!DOCTYPE/<!ENTITY`。

**rtf.py**：`_strip_destination_groups` 是个括号深度状态机——只剥控制字
会把字体名/生成器/作者元数据**留在正文里**，必须整组删；`\*` ignorable
组按 RTF 规范跳过（顺带覆盖厂商扩展）；**未闭合目标组 → 返回原始文本**：
"截断整本书是比泄漏元数据残留更大的恶"。转义字面量暂存
（`\\→\x01` 等）必须在目标组扫描**之后**、控制字处理**之前**——时序
注释写得很清楚。

**html.py**：块级标签**开与闭都出边界**——没有闭边界
`<h2>Chapter 1</h2>Intro` 会连成 "Chapter 1Intro"，而
`_EXPLICIT_CHAPTER` 需要数字后词边界，"1I" 没有——**HTML 提取质量直接
决定章节检测成败**；表格单元格用 tab 连接（行保持单行，表格型目录可被
当作单行标题解析）；`_pending` 延迟最强边界（`\n` 不被 `<td>` 降级成
`\t`）；`convert_charrefs=True` 已解码实体，**不再二次 unescape**（否则
`&amp;amp;` 折叠错）。trafilatura 失败/不自信（返回 None/空白而非异常）
也降级而非传播。

**calibre.py**：每次转换用**独立 TemporaryDirectory**——共享 workdir 下
可预测文件名会被上次运行的残留满足（"转换成功但没写"会返回**别的书**的
文本并记到本书名下）；pid 复用都不够，必须 fresh 目录；目录创建失败
**raise 而非 return None**——None 与"成功但空"不可区分，正是该函数要
拒绝的静默失败形态。

## 4. sanitize.py（9 组攻击形态 + 1 个修复）

按攻击形状分组（可审查性优先）：零宽间隔符（含 U+2061-2064 不可见数学
算子、软连字符、蒙文元音分隔符）；**双向控制（Trojan Source
CVE-2021-42574）**——注释点透本质："模型读到的字符序列不变，**人类看到的
顺序**变"；隐形字母（韩文填充器——是字母不是空白，活过空白归一化，可给
隐藏文本垫宽）；标签块 U+E0000-E007F（整段 ASCII 载荷走私）；**变体选择
符**（每个携带 256 值之一，基字符后一串选择符 = 不可见任意载荷；是组合
记号所以 Cf 类别过滤抓不到）；弃用格式控制/行间注记/音乐格式控制。

`is_invisible_codepoint` 单独暴露的意图写在 L210-213：给生成技能扫描器
用——"两层集合漂移时，extractor 放过扫描器报警的字符，或更糟，两层都不
覆盖"。**这是"单点复用防漂移"原则的另一实例**（与章节正则共享同理）。

康熙部首折叠（L158-204）：NFKC **派生**映射表（与 Unicode 零同步成本）；
范围刻意限定在 U+2F00-2FDF——全文本 NFKC 会改写全角形式/连字/圈数字/
兼容表意（合法正文，`test_normalization_is_scoped_to_kangxi_radicals`
钉死）；补充区 U+2E80-2EF3 刻意不折（无兼容分解，且 U+2EEB 是普通汉字
⻫≠二）。折叠是**可读性修复而非安全清除**，不计入移除总数。RFC 01 中文
优化方向里"字体映射损坏"问题的兜底层已在此。

## 5. dependencies.py

8 个依赖组的**满意语义**分四级：`any_tool_suffices`（模块或系统工具任一）、
`any_of_modules`（至少一个模块）、默认全量、系统工具必备（Calibre 是唯一
硬依赖且仅限 MOBI/AZW）。`isolated_install_hint`（L84）处理 pipx 隔离
安装——"`✗ python: docling` 正确但没用"，给出 venv Python 路径且**永不谎称
可用**（"PATH 上的二进制不使 import 成立"）。ask 模式只在
`sys.stdin.isatty()` 时提问；`--check` 恒 exit 0（缺可选依赖不是错误）。
`prepare_dependencies`（L215）是格式 → 依赖协商的必经点——新格式接线的
第三站（`config.py` 注册 → 这里加组 → `extract_single_file` 加分支）。

## 6. pdf_inspector_integration.py（hook 先例解剖）

RFC 01 OCR 子项目声称"照 pdf_inspector hook 先例"，本文把该先例拆开验证：

- **安装**（L193）：monkey-patch 包裹 `utils.extract_single_file`，幂等
  属性守卫（`_book_to_skill_pdf_inspector_hook`）防重复包裹；
- **快路径条件**（L218）：仅 text 模式 + inspector 返回非空 markdown +
  信任门全过（`text_based` + confidence ≥ 0.90 + 无待 OCR 页 + 无编码
  问题）才走 native Markdown；technical 模式**原样通过**；
- **信任门哲学**（L47-55）：inspect 结果"刻意保守"——不信任时**只回
  元数据**，既有 fallback 链保持权威；降级原因人类可读（`_fallback_reason`）；
- **结果复刻**（L118）：`_result_from_inspector` 完整重建 per-source
  字典（sanitize → detect_structure → estimate_tokens → sha256）；sha256
  **惰性直 import** 而不走注入的 `utils_module`——现有测试传
  SimpleNamespace 当 utils，哈希必须是文件本身的纯函数（测试 seams 的
  设计意图）；
- **元数据回写**（L236）：按 resolve() 后的 source_file 匹配，tmp +
  replace 原子替换，单源时提升到顶层；
- `_reset_state_for_tests` 是显式测试钩子。

**结论：OCR hook 完全可以套这个形状**——wrap `extract_single_file`，在
`looks_image_only` 判定前插入"OCR 预处理"分支（或直接替换为 OCR 后的
文本层），信任门 + fallback_reason + 元数据回写三件套照抄。另一条路线
（开 docling 自带 OCR，`do_ocr=True`）改动更小但重（模型管线）且失去
"每页原因"可观测性。RFC 01 推荐前者，本文深读支持该判断。

## 7. 对 RFC 01 各子项目的验证结论

| RFC 假设 | 深读验证 | 结论 |
|----------|---------|------|
| OCR 走 hook 模式不侵入主链 | §6 全套先例（幂等守卫/信任门/原子回写/测试钩子）已验证 | **成立**，且插入点建议从"looks_image_only 之后"上移为"wrap 层判定之前"（省一次注定失败的探测）；注意 `extract_with_docling do_ocr=False` 意味着 technical 模式同样吃不到 OCR，hook 两条模式都要考虑 |
| token 校准 1.5→1.2 + 可选 tiktoken | 常数单点（config L35）；estimate_tokens 刻意确定性 | **成立**，但 tiktoken 必须走可选注入（默认路径不可加依赖）；metadata 与合并全文各估一次，两处共用同一函数无漂移风险 |
| docling 代码块换行重建 | docling 导出无坐标（`export_to_markdown()` 无参），重建只能文本启发式；插入点在 docling 成功分支与 sanitize 之间 | **成立**，双阈值设计必要；另注意重建须在 sanitize 前完成（否则零宽探针被剥） |
| 不碰中文章节检测 | L153-186 + L691 覆盖第N章回卷节篇讲/md 前缀/康熙数字/全角/中文数字 1-999 | **正确**，中文章节检测已是全项目覆盖最全的语言分支 |
| 回归防线用 evals replay | reuse_is_safe/hook 的注入式测试面（SimpleNamespace 先例）+ Rust Book 语料校准先例 | **成立**，fixture 习惯与测试 seams 均有现成模板 |

深读新登记的两点（RFC 01 评审时补充考虑）：① per-source 结果携带全文 +
合并文本 = 大书内存 ×2，OCR 中间产物注意峰值；② `reuse_is_safe` 不含
提取器版本指纹——若 OCR 引入缓存层，须自行把 OCR 引擎版本纳入指纹，
这是现成判据的已知盲区而非缺陷。

## 8. 测试锚点速查（阶段三回归防线挂点）

- 章节检测：语言矩阵 fixture（Rust Book 校准先例，#237/#91/#272/#274 均
  有测试背书）；
- hook/依赖：`SimpleNamespace` 注入 + `_reset_state_for_tests`；
- 恢复：`reuse_is_safe` 的 Counter 乱序/指纹缺失分支；
- 清洗：`test_normalization_is_scoped_to_kangxi_radicals` 一类"范围钉死"
  测试范式；
- 端到端：`evals/replay.py` 轨迹打分（缺口补齐记录已实测）。
