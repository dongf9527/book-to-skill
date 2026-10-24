# 阶段一缺口补齐记录（2026-10-24）

> 对应 [ROADMAP.md](../ROADMAP.md) 阶段一的三个缺口。全部在本环境（Python 3.12 / Linux）实测完成。
> 结论先行：三个缺口全部闭合，阶段一可标记 100%；docling 实测发现一个此前未记录的质量风险（代码块换行丢失），已登记为二开设计的输入。

---

## 缺口 1：SKILL.md Step 3-11 精读

879 行规格书全文精读完毕（此前仅精读 Step 0-2）。对二开有直接影响的 10 个要点：

1. **成本估算公式**（Step 2.5）：input ≈ `estimated_tokens × 1.3`；output ≈ `章节数 × 章节预算 + 4,000（SKILL.md）+ 4,500（辅助文件）`。二开 Web 化时可直接复用该公式做转换前报价。
2. **RLM 式大书访问**（Step 2.6）：>50k token 的书强制用 `grep`/`sed` 探针按需取段，禁止无界 `Read` 全文——这是"成本与输出成正比、与源体积脱钩"的关键机制。
3. **DEPTH 由用途推导**（Step 4）：选"参考查阅"→ reference（精简章），含"应用/思考"→ study（含 Worked Example），不单独提问。
4. **宿主根选择**（Step 5）：默认枢纽 `~/.agents/skills/`；**Claude Code 不扫该根**，必须 symlink 到 `~/.claude/skills/`；Hermes 有独立分类根。二开新宿主支持时照抄这套"枢纽+适配"模式。
5. **章节预算矩阵**（Step 7）：`BOOK_TYPE × DEPTH` → 800–3,000 tokens/章；**study 深度必须用 Worked Example"赚得"**，禁止注水。
6. **cheatsheet 是差异化层**（Step 8）：装的是作者的"判断"（决策规则/决策树/权衡矩阵/阈值/信号），不是术语表——输出模板二开时此层最值得重设计。
7. **主文件 <4,000 tokens**（Step 9）：compaction 从尾部截断，重要内容必须前置。
8. **生成后必跑注入扫描**（Step 9.5）：`scan_generated_skill.py` 非零退出则停，交人工审查，禁止静默改写。
9. **symlink 读回验证**（Step 10）：`CLAUDE_STATUS` 四态（linked / skipped-realdir / copy / absent），报告必须基于磁盘事实而非"命令执行过"。
10. **发布双门禁**（Step 11）：可见性必须收到**裸词 `public`** 才公开（"it's public domain"这类描述书的句子不算）；版权门禁——第三方版权书强制私有；嵌套仓库守卫防 gitlink 陷阱。

另：Update/Fold-in 工作流（增量合并）与 8 条 Quality Rules（结构非摘要/保留作者精确性/密度优先/从业者口吻/前置加载/按需章节/永不复制原文/Topic Index 是导航关键）已完整理解，输出模板方向的 RFC 将以此为约束基线。

## 缺口 2：tools/evals/ 评测框架实跑

**框架定位**：为"渐进披露"（progressive disclosure）行为做确定性回归评测——不调 LLM，对固定的合成轨迹（fixture）打分，保证 CI 可复现。

实测两条链路：

```bash
# ① replay：对 5 条合成问答轨迹打分（路由正确性/答案正确性/token 用量）
python3 tools/evals/replay.py evals/fixtures/replay_trajectories.json
# → aggregate: 5 questions = 1 correct + 1 irrelevant_opens_before_target
#             + 1 unknown + 1 wrong_answer + 1 wrong_routing；usage 双口径记录

# ② paper_flat：把 fixture 书按章节标题切块，生成"flat pack"评测技能包
python3 tools/evals/paper_flat.py evals/fixtures/pd03-book.txt \
        evals/fixtures/pd03-metadata.json /tmp/pd03-pack
# → 产出 SKILL.md（含 chunk 路由表）+ chunks/chunk-0N.md
```

**组成**：`replay.py`（确定性重放）→ `score.py`（打分）→ `manifest.py`（评测运行清单，自带 secret 扫描护栏）→ `paper_flat.py`（一级分块路由评测包构建）。

**对二开的启示**：二开如果改动章节检测/路由相关逻辑，应为自己的改动建立同构的 replay fixture（合成轨迹 + 确定性打分），而不是只靠 pytest 单测——这填补了"无端到端测试"的空白（见 EVALUATION.md 第四节）。

## 缺口 3：docling technical 模式实测

**方法**：自造 1 页含表格 + 代码块的 PDF（`scripts/make_test_pdf.py`，reportlab 生成），分别用 `--mode text`（pdftotext）与 `--mode technical`（docling 2.142.1，CPU）提取，对比产物。

**实测对比**：

| 维度 | text 模式（pdftotext） | technical 模式（docling） |
|---|---|---|
| 表格 | 空格对齐纯文本，无结构 | **标准 Markdown 表格**（表头+分隔线），Step 7 "Reference Tables"可直接引用 |
| 标题 | 普通文本行 | **`##` Markdown 标题**，使章节检测从 numeric(1章) 升级为 structural(2章) |
| 代码块 | 保留换行，但缩进漂移（`-layout` 空格副作用） | **换行丢失，6 行代码压成 1 行**；另引入 `\_` 转义噪声 |
| 耗时 | ~0.1s | 41.9s（首次含模型加载；稳态约 1.5s/页，与官方标称一致） |

**新发现（上游文档未明示的质量风险）**：对无字体/行距暗示的简单排版代码，docling 会丢失换行。**含义**：技术书若代码排版简单，technical 模式的代码质量可能反而低于 text 模式；二开"中文技术书优化"方向时，候选修复路径：① docling 输出后对等宽字体区段做换行重建后处理；② 在 SKILL.md Step 1.5 问询里增加"代码密集型书籍"提示，建议双模式对比选择。

**环境备注**：docling 安装含 torch/transformers，占磁盘约 2.5GB；首次运行从 HuggingFace 拉模型需外网。安装过程曾因磁盘满失败一次（`pip cache purge` 清 3GB 后恢复）。

---

## 阶段一验收

- [x] 缺口 1 闭合：SKILL.md 全文精读，10 要点提炼如上
- [x] 缺口 2 闭合：evals 两条链路（replay + paper_flat）实跑通过，框架定位明确
- [x] 缺口 3 闭合：text/technical 双模式实测对比完成，1 个新风险登记
- [x] 阶段一状态：**100%** —— 具备进入阶段二（设计）的全部事实基础
