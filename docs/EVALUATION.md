# book-to-skill 项目评估报告

> 评估日期：2026-10-24 ｜ 评估对象：[virgiliojr94/book-to-skill](https://github.com/virgiliojr94/book-to-skill) @ master (a24e62f)
> 评估方式：源码级深度评审（全量 4,500+ 行核心代码 + 65 个测试文件）+ 本地实际运行验证

---

## 一、项目定位

book-to-skill 解决一个具体痛点：**技术书读完就忘，用时找不到**。它把一本 PDF/EPUB/DOCX（或整个文档目录）转换成一个 AI Agent 可挂载的 **Skill 技能包**——由 `SKILL.md`（约 4K token 的核心心智模型 + 章节索引）和按需加载的章节文件组成。之后你在 Claude Code、GitHub Copilot CLI、Amp、Codex、OpenCode 等任何支持 Agent Skills 标准的宿主里提问，agent 只加载相关章节作答，官方实测比把全书塞进上下文**节省 24×–51× token**。

项目热度与成熟度：⭐ 35.4k stars / 3.7k forks / 228 commits，MIT 协议，自带 901 个单元测试、CI（GitHub Actions + CodeQL）、semver 变更日志和 MkDocs 文档站。在同类"文档 → agent 知识"工具中是事实上的头部开源项目。

## 二、架构评估：一个聪明的"两半"设计

整个系统分成两个耦合极低的部分，这是它最值得借鉴的设计：

```
┌─────────────────────────────────────────────────────────────┐
│  第一半：确定性 Python 提取器（book_to_skill/ 包）              │
│  输入: 任意文档 ──→ 输出: full_text.txt + metadata.json       │
│  （纯本地、零 LLM 调用、可单测、可复现）                        │
├─────────────────────────────────────────────────────────────┤
│  第二半：SKILL.md 规格书（879 行 prompt 工程）                 │
│  agent 读取提取产物，执行 Step 0-11：                          │
│  分析结构 → 问用途 → 定名 → 拆章 → 生成 glossary/patterns/     │
│  cheatsheet → 编译主 SKILL.md → 注入扫描 → 发布 GitHub        │
└─────────────────────────────────────────────────────────────┘
```

**关键洞察**：Python 层只负责"文档 → 干净文本 + 结构元数据"这一原子操作，而"怎么提炼框架、怎么写章节摘要"这些需要智能的部分全部交给宿主 agent 按照 SKILL.md 的规程执行。这带来三个好处：

1. **确定性部分可测试**——901 个测试全部覆盖 Python 层，CI 秒级跑完；
2. **智能部分免维护**——不需要 API key、不需要模型调用代码，换模型/换宿主零成本；
3. **"单点复用防漂移"贯穿全库**——章节检测正则同时被提取器和 token 计量工具 `tools/discovery_tax.py` import 同一份；不可见字符清洗集合同时被 `sanitize.py` 和生成物扫描器 `tools/scan_generated_skill.py` 共用，从机制上杜绝"提取器放行、扫描器误报"。

**工程质量亮点**（均经源码确认）：

- **鲁棒的章节检测**：三层降级策略——多语言数字编号正则（覆盖中/俄/希腊/泰/印地等 15+ 语言，含中文"第N章/回/卷"和康熙部首数字）→ Markdown/AsciiDoc 结构标题分析（按深度分组取"最浅的、拥有 ≥2 个不同标题的层"）→ Part 标头兜底。我们实测一本中文样例书，4 章"第N章"全部正确识别。
- **安全意识罕见地好**：DOCX 解析前扫描 XML 实体防 XXE/Billion Laughs；`sanitize.py` 清理 Trojan Source 双向控制符（CVE-2021-42574）、零宽字符、变体选择器等 9 类隐形码点；生成物还要过一遍 prompt 注入扫描（"ignore previous instructions" 等 7 类规则）；临时目录 chmod 0700 且拒绝 symlink。
- **工程细节**：每进程独立 workdir 防并发竞态；sha256 内容指纹支持中断恢复（`reuse_is_safe()`）；Windows GBK 控制台、CRLF 换行、cp1252 locale 等跨平台坑都显式处理；PDF 页眉页脚只删"页缘位置"防误删正文标题。

## 三、实测验证结果

在本环境（Python 3.12）实测：

| 验证项 | 结果 |
|---|---|
| 依赖检查 `--check` | ✅ 正常，缺失项给出精确安装命令 |
| Markdown 提取 + 中文章节检测 | ✅ 4 章"第N章"全部识别（numeric 方法） |
| metadata.json 结构 | ✅ 含 sha256、token 估算（CJK 感知：957 字符 → 446 tokens）、章节样本 |
| 测试套件 `pytest tests/` | ✅ **901 passed, 6 skipped, 5.68s** |
| 未装依赖 | EPUB/HTML/RTF 均有 stdlib 降级路径，PDF 有 pypdf/pdfminer 三级 fallback |

## 四、综合评分

| 维度 | 评分 | 说明 |
|---|---|---|
| 代码质量 | 9.5/10 | 注释解释"为什么"而非"是什么"，每个回归测试锚定 issue 编号 |
| 架构设计 | 9/10 | 确定性/智能性分离干净，扩展点清晰 |
| 测试完备 | 9/10 | 901 个测试，但缺"真实 PDF→完整 skill"的端到端测试（由 evals 补位） |
| 文档完备 | 9.5/10 | README + 5 篇专题文档 + 中/俄翻译 + MkDocs 站点 |
| 生产就绪 | 8.5/10 | 隐私本地处理、失败路径全部收敛 ExtractionError；但 MOBI 强依赖外部 Calibre |
| 社区活跃 | 9/10 | 35.4k stars，issue/PR 活跃，有 sponsor 支撑 |

**主要限制**：① 扫描版 PDF 无文字层时直接拒绝（需先 `ocrmypdf`，这是刻意设计但中文用户会遇到更多）；② 章节自动检测依赖显式编号，像 Pro Git 这类用节标题的书不会自动分段；③ 生成质量完全取决于宿主 agent 的模型能力（生成一本完整 skill 约需 100K+ token，官方估算约 $1/本）；④ 无原生中文分词优化，中文 token 估算是字符数/1.5 的近似。

## 五、结论与建议

**值得二开吗？值得。** 这是一个代码纪律极好的中型项目（核心 ~4.5K 行 + 900 测试），MIT 协议无法律障碍，扩展点在代码里是天然成形的（见 DEVELOPMENT.md 的改动点清单）。它当前明显偏英文书籍生态，**中文优化是最大的差异化空间**；同时"Python 只做提取、agent 做生成"的架构留了大量服务化/工程化的二开余地。建议以"克隆后二开"方式起步，保留上游 git remote 以便合入社区修复。
