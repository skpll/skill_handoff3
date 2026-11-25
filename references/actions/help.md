# 帮助（actions/help.md）

⑤帮助的执行正文。**本动作不写任何文件**：不碰 `mem_`、不重建视图、不出报告，落款用 B（见 SKILL.md §2.2）。除常驻规则要求读的 `$LIB/handoff_patches.md`（存在才读）外，不再读别的文件。

按**用户该条消息的语言**输出下面一块，两套不混用；拿不准、中英夹杂或写不准 → 用 English 块。命令清单**不含「帮助」本身**，末尾不附加任何参数展示——本技能无配置层，所有行为固定。**逐字输出**，不改写、不补充。

## 中文

```text
handoff3 用法：

①  /handoff3 交接
→ 调用技能后不加任何触发词，按[交接]处理。
→ 已包含自检：交接时自动做六步体检与修复，不必先单独发「自检」。
→ 交付完整交接报告，推荐日常使用。

②  更新记忆
→ 只处理当前 agent、当前会话那一个记忆文件：缺失就建、漏记就补、有问题就修。
→ 落款没生效、或怀疑本会话记忆文件有问题/缺失时用；不动别的文件、不出报告。

③  /handoff3 自检
→ 只检查修复记忆文件，不交付报告；交接已含自检，日常不必单独发。

④  /handoff3 基于记忆 [想说的话]
→ 必须带 /handoff3 前缀；只说「基于记忆」不会触发本技能。
→ 在「基于记忆」后面，填上你这次想问的话或要做的事。
→ 先读取 handoff3 最新的全量记忆，再与 AI 协同，完成你自己的工作流。

出问题怎么办：
· 优先推荐使用[交接]，已含自检，一次完成修复、更新视图和报告。
· 只想体检不出稿时单独用[自检]；只想修当前会话这一个文件用[更新记忆]。
· 有疑问或困惑，直接告诉大模型，让 AI 协助解决。
· 把问题反馈给官方。
```

## English

```text
handoff3 usage:

1)  /handoff3 handoff
-> Invoke the skill with no trigger word; it is treated as a handoff.
-> Already includes self-check: the six checks and repairs run automatically - no need to send self-check first.
-> Produces the full handoff report. Recommended for daily use.

2)  update memory
-> Handles only this agent's current-session memory file: create it if missing, fill what is missing, repair what is broken.
-> Use it when the footer did not appear, or when you suspect this session's file is broken or missing. It touches no other file and writes no report.

3)  /handoff3 self-check
-> Checks and repairs memory files only, no report. Handoff already includes it, so you rarely need it alone.

4)  /handoff3 from memory [your request]
-> The /handoff3 prefix is required; saying "from memory" alone does not trigger the skill.
-> Put your question or task after "from memory".
-> It reads the latest full memory first, then works with the AI on your task.

When something goes wrong:
- Prefer handoff: it includes self-check and completes repair, view rebuild and report in one pass.
- Use self-check alone for an audit without a report; use update memory to fix only this session's file.
- If anything is unclear, just tell the model and let the AI help.
- Report issues to the official channel.
```
