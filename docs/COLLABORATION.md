# GitHub 协作规则

> 生效日期：2026-10-24 ｜ 适用仓库：[dongf9527/book-to-skill](https://github.com/dongf9527/book-to-skill)
> 本文档由协作双方共同遵守，规则的修订以 commit 形式提交（`docs:` 前缀），不接受口头例外。

---

## 一、核心规则表

| # | 规则 | 说明 |
|---|------|------|
| 1 | **唯一同步方式** | 所有代码/文档改动通过 `git commit` + `git push` 推送到 GitHub，不在对话中凭空产生代码 |
| 2 | **仓库位置** | `/home/z/my-project/book-to-skill`，不在其他目录创建 `.git` |
| 3 | **提交规范** | 小步提交，commit message 使用 conventional commits 风格（`feat:` / `fix:` / `docs:` / `refactor:` / `chore:` / `test:`） |
| 4 | **会话记录** | 分析/评估/测试记录写入 `docs/` 目录，以 Markdown 形式提交推送 |
| 5 | **禁止 force push** | 绝不使用 `git push --force`；同理禁止对已推送 commit 执行 `--amend` 后再推送（等效 force，同样触犯本条，2026-10-24 注记）；未推送 commit 的 amend 属修正笔误常规手段，不受限 |
| 6 | **Rebase 前置** | rebase 前先评估替代方案（revert/merge），必要时提前告知确认 |
| 7 | **bug 修复不重命名变量** | 修复 bug 时保持原变量名，仅在 refactor 时重命名 |

## 二、Remote 布局

```
origin   → https://github.com/dongf9527/book-to-skill.git   （写目标：所有 push 走这里）
upstream → https://github.com/virgiliojr94/book-to-skill.git（上游源：仅 fetch，永不 push）
```

- **同步上游流程**：`git fetch upstream` → 查看差异（`git log master..upstream/master --oneline`）→ 按规则 #6 评估 merge / revert / rebase 三方案的代价 → 涉及历史改写时先告知确认。
- 分支策略：当前单一 `master` 分支直推；后续若开启功能开发，使用 `feat/<topic>` 短命分支，合入后删除。

## 三、提交细则

- 一个 commit 只做一个逻辑单元；文档与代码改动分开提交（`docs:` 与 `feat:`/`fix:` 不混装）。
- commit message 格式：`<type>: <祈使句摘要，≤72字符>`，需要时加空行 + body 说明动机与影响面。
- 仓库级身份已配置：`user.name=dongf9527`、`user.email=dongf9527@users.noreply.github.com`（noreply 邮箱，不暴露真实邮箱）。
- 推送前三查：`git status` 干净（无未跟踪遗留）、`git log --oneline -5` 确认无意外提交、`git rev-parse --is-shallow-repository` 确认非浅克隆（见第六节教训）。

## 四、凭证与安全管理

- GitHub PAT **仅由助手（AI）使用**，有效期 30 天（至 2026-11-23 前后），存储于本地 git 凭证体系：
  - 主通道 `credential.helper store`（`~/.git-credentials`，权限 `600`，仅所有者可读写）；
  - 辅通道 `credential.helper cache --timeout=2592000`（内存缓存 30 天）。
- **Token 永不写入**：仓库任何文件、`docs/` 文档、commit message、分支名、issue/PR 正文。本文档本身也不含 token。
- 凭证失效（过期/revoke）时的恢复流程：用户重新提供 token → `git credential approve` 重新灌入 → 推送验证。
- 远端仓库若转为公开，需复查历史中是否误入敏感信息（当前历史干净）。

## 五、历史保护

- 规则 #5 的操作面：禁用 `push --force` / `push --force-with-lease` / 已推送 commit 的 `amend + push`。
- `git reset` 仅限**未推送**的本地 commit；任何触及 `origin/master` 已有历史的操作都先停下确认。
- rebase 前置检查单：① 是否有未推送 commit 会被重写；② merge 是否同样可行；③ 冲突规模预估。三项过完才允许执行。

## 六、浅克隆与历史完整性（2026-10-24 事故记录）

**事故**：首次推送时因克隆使用了 `--depth 50`，本地缺少完整历史对象，远端报 `remote unpack failed: did not receive expected object`，推送被拒。

**修复**：`git fetch --unshallow upstream` 补全 228+ commits 全量历史与版本 tags 后重推成功。

**固化规则**：
- 永远不做新的浅克隆（不用 `--depth`）；如必须浅克隆（如 CI 场景），禁止从其 push。
- push 前自检：`git rev-parse --is-shallow-repository` 必须返回 `false`。
- 从浅克隆恢复：`git fetch --unshallow <remote>`，完成后重验。

## 七、会话记录归档约定

- 目录：`docs/`。现有归档：
  - `docs/EVALUATION.md` —— 项目源码级评估报告（2026-10-24）
  - `docs/DEVELOPMENT.md` —— 二次开发指南（五方向改动点 + 扩展点速查表）
  - `docs/COLLABORATION.md` —— 本文档
- 后续新增记录按主题命名（如 `docs/testing/<date>-<topic>.md`），并在本清单登记。
- 每次会话结束前，若有未推送的分析/测试产出，按规则 #1/#4 提交推送，不留"只在对话里"的结论。

## 八、变更记录

| 日期 | 变更 |
|------|------|
| 2026-10-24 | 初版：核心规则表 7 条 + remote 布局 + 凭证管理 + 浅克隆事故记录 |
