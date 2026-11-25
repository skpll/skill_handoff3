---
name: handoff3
description: 跨agent会话记忆与交接，边记边交。常驻：$LIB=项目根`.handoff3/root.txt`首行绝对路径，无则项目内`.handoff3`，指错停。每轮先读`$LIB/handoff_patches.md`（有则读、无则跳过，判据句对所有轮生效），再把实质结论追加进`$LIB/memory/mem_<agent>_<会话>.md`（空格换下划线，带agent/时间/类型/逐字quote）。写入只限`$LIB`内、其余不写（正向白名单，不穷举）；"参考某agent记忆"仅指读取。优先级：用户当轮指令>项目AGENTS.md>patches判据句>技能正文，用户答"是"的patch与AGENTS.md冲突时视为最新指令可覆盖。末条回复二选一落款：碰过mem文件（写入或修复）→落款（反引号绝对路径+token，取不到省括号），完全没碰→未写入提示（默认，拿不准用它）；语言随该条回复，拿不准用英文，模板§2.2。五动作（交接/更新记忆/自检/基于记忆/帮助）触发词与边界见正文§3；仅「基于记忆」须`/handoff3`前缀。
license: MIT
compatibility: 遵循 agentskills.io 规范，跨 Agent 通用。记忆流程只用宿主自带的文件读写与搜索（grep）能力，无配置层、无外部工具。数据默认写项目内 .handoff3/，可指向外部记忆库；对用户指定的会话记录只读。
metadata:
  version: "1.1.0"
  breaks-from: "旧版 handoff（一 agent 一文件 + 水位行 → 一会话一文件 + 核对至）"
---

# 会话增量记忆与交接（一会话一文件）

把「交接」从一次性大读取，改成为日常顺手记账、交接时一键成稿。

**一个会话一个 `mem_` 文件**：每条记忆天然溯源到唯一会话记录，记录被删除或归档后自动转为只读沉淀。文件数随会话数增长是**有意取舍**——溯源清楚好过几个会话混在一个文件里。

三层加载：**常驻轻规则**在 description（每轮可见）；**动作流程**在本文件（激活时加载）；**判据与细则**在 references（按需加载）。

## 1. 数据布局

`$LIB` 解析是每个动作的第一件事：项目根 `.handoff3/root.txt` 存在且非空 → 取**首个非空行、去首尾空白（含 CRLF）**作为绝对路径；不存在 → 项目根 `.handoff3`（原位，向后兼容）；指针指向的路径不存在或不可读 → **报错停止**并写出该路径，绝不回退到项目内新建库（回退会让数据静默分裂成两份）。

```
$LIB/
├── memory/
│   ├── mem_WorkBuddy_a224e4fd-1509-42b7-9eb2-7de8f4903e72.md   一会话一个，只增不改
│   ├── mem_WorkBuddy_a5e6f3af-7107-43af-8dd1-ba29c8e7a6f0.md
│   ├── mem_WorkBuddy_AI_<会话标识>.md                           各 agent 各写各的
│   ├── mem_WorkBuddy.md                                        旧格式聚合文件，只读保留
│   └── ...
├── handoff_memory.md     全量视图：由 memory/ 重建，可随时刷新
├── handoff_patches.md    判据句规则源（每轮加载、非档案）：需用户答「是」才追加；不存在时跳过
└── handoff_report.md     产物：交接稿
```

外部库布局相同，项目内只留 `.handoff3/root.txt` 一行绝对路径。

### 1.1 视图重建（`handoff_memory.md`）

`handoff_memory.md` 是派生汇总（只由 `memory/` 生成，任何动作都不回写源文件），**只在三个时机重建**：① 交接的「重建视图」节；②「基于记忆」发现视图缺失、或末条时间落后于任一 `mem_` 末条时间；③ 用户以「重建视图 / 重建记忆汇总 / 刷新全量记忆」触发轻入口（只重建，不补写、不出报告）。**纯记忆轮不重建**——不因为「换了窗口」「发了个继续」就刷产物。

重建方式是覆盖写；构造规则（时间升序、同 quote 去重留最早、旧格式标记、不写回 `mem_`）统一见 [schema.md](references/schema.md) §2.4，本文件不复述。

**库根可共用，库目录不可共用**：允许 `<库根>/project_a`、`<库根>/project_b`；**禁止**两个项目指向同一具体库目录（同名文件被同写、视图与报告互相覆盖且不报错）。迁移前目标已有 `mem_*.md` → 停止，不覆盖不合并。

**文件数会增长，这是已知代价**：`/clear`、换窗口、新任务都换会话标识，几十个 `mem_` 文件是正常状态。处理方式三条，都不需人工干预：

- **视图统一**：读的时候只看 `handoff_memory.md`，不必翻目录；
- **沉淀不清理**：`已沉淀` 的文件**不删、不合并**；真要清理须用户明确要求，且只删已沉淀、按时间边界说明；
- **不留备份**：每次更新都是原地更新，不生成 `.bak-*` 等任何副本（副本让库膨胀，不提供额外价值）。

不要为「减少文件数」自动合并——那会抹掉溯源清晰这唯一好处，并把归属歧义重新引进来。

## 2. 每轮记忆（轻 · 常驻 · 不加载本文件也要做）

### 2.1 判定、写入

每轮对话产生实质结论后：

0. **先读判据句**：`$LIB/handoff_patches.md` 存在则读、按其判据句执行；不存在就跳过（读取时的正常路径，不报错）。判据句对所有轮次生效。**读写不对称**：读时无则跳过，用户答「是」要追加时（§3.1）无则创建。
1. **判 agent 与库根**，再写。agent 名与当前会话标识按 [adapters/](references/adapters/index.md) 的接口取得——**禁止硬编码某家产品的 id 格式、禁止扫全局进程列表挑一个、禁止参考 `$LIB` 里已有文件名**（现存文件名与「文件源」段都是历史产物，可能一开始就标错了）。
2. **写进该会话的文件**：`$LIB/memory/mem_<agent>_<会话>.md`。文件不存在就新建，带标题 + 文件源 + 正文（见 [schema.md](references/schema.md) §1–3）。
3. **写前查状态**：目标文件「状态」为 `已沉淀` → 说明该来源已冻结，本轮不该往里写。**停下、报告、不动文件**。这条同时兜住「agent 判定错把内容写进别人文件」这类事故。**唯一例外**：用户显式发「更新记忆」并指名该文件、或当轮明确要求解冻——此时按 [actions/update-memory.md](references/actions/update-memory.md) 判断来源是否真可达，可达才解冻。
4. 只追加本轮新内容，不回头重读、不重写旧条目（例外：①交接内嵌的自检、②更新记忆、③自检命中时可按需就地修本文件）。

**时间格式固定** `YYYY-MM-DD HH:MM`，不带时区后缀（详见 [schema.md](references/schema.md) §3）。从会话记录逐字重放恢复 mem 文件时照抄历史字节，不改写时间标签。

**「实质结论」判据**（能落成是/否的条件；判不清一律按「是」，宁可多记）：满足任一条即算——用户提出约束 / 偏好 / 否决 / 验收标准（哪怕一句）；做出决策（选方案、删留、改口径、定落点）；查明根因或澄清误解；落地改动且过门禁（有 commit / 测试数 / 体积等可核查证据）；推翻此前结论。不算的：纯查询答复、复述已知、确认收到、解释名词、追问澄清。拿不准就问自己一句「下个会话的人不看这轮，会不会因此做错事或重复劳动」。

### 2.2 结尾信号（硬检查项，只管一条回复）

范围仅限本轮**最后一条面向用户的回复**——工具调用之间的过程话、90–150 字的「让我先看看…」过渡句不在范围内，不必附（一轮里出现多条都带信号的，反而是没读懂这条）。只发一条短回复的轮次照样要附。

二选一，**B 是默认**：没把握到底碰过没有就用 B；A 只在本轮确实**写入或修复过** `mem_` 文件时才用——追加条目算，②更新记忆/自检就地修复了本文件也算。A 是**替换** B、不是叠加，**两样都不加等于本条规则没执行**。

**文案语言**：以**本轮最后一条回复的正文语言**为唯一依据（不看用户提问、工具输出、思考过程；三者不一致只认正文）。中文 → 中文模板；英文 → English 模板；其他语言且能稳定写出 → 按模板意译，结构与反引号路径、token 括号照旧；**拿不准、中英夹杂或写不准 → 一律用 English**（安全兜底，不算违规）。

两套模板按上述择一，绝不混用；宁可用英文，也不要附一段语言不对的落款。

**A. 落款**（本轮确实写入或修复过 `mem_` 文件）：

中文：
```
---
本轮对话要点已自动加入 handoff3 记忆：
`<$LIB/memory/mem_<agent>_<会话>.md 的绝对路径>`（Token消耗：输出 <N>）
```

English：
```
---
This turn's key points have been added to handoff3 memory:
`<absolute path to $LIB/memory/mem_<agent>_<session>.md>` (Output tokens: <N>)
```

路径运行时取绝对路径，**禁止写死项目路径**，用反引号包裹（避免 Windows 反斜杠被 Markdown 吃掉）。`<N>` 取宿主最新完整一轮的输出 token 读数，取不到就省略整个括号。

**B. 未写入提示**（本轮完全没碰 `mem_` 文件——帮助轮、纯分析轮；也是拿不准时的默认）：

中文：
```
---
本轮未写入记忆，handoff3 未改动记忆文件。
```

English：
```
---
No memory was written this turn; handoff3 did not change any memory file.
```

完全没碰 `mem_` 文件就不能用 A 的文案（会说假话）；只修了文件结构/核对至也算碰过，用 A。「记忆文件」专指 `mem_` 真源，`handoff_memory.md` 视图不算——基于记忆那轮即便刷了视图，B 依然属实。不用「只读」措辞：帮助轮、分析轮并非只读，只是本轮没写文件。

历史教训：11-20 某会话 66 条回复被初判「32 条两样都缺」，逐条复核真违规只有 4 条，其余全是同轮过渡句。范围收紧到「最后一条」后，缺的才数得清。

## 3. 五个动作（激活本技能时）

每轮记忆照常做，下列动作**命中才额外执行**。**顺序固定：先执行命中的动作，再写本轮的每轮记忆条目**——先写会把该文件的 `核对至` 推到当前，把之前真缺的轮次永久盖住（交接会据此报「无缺口」）。**本表是触发词的唯一权威**，别处不复述词表。

| 动作 | 调用形式 | 触发词（中 · 英） | 副作用（碰 `mem_` / 重建视图 / 写报告） | 详规 |
|---|---|---|---|---|
| ① 交接 | 裸 `/handoff3` | 交接 · 生成 · 报告 · 文档 · 归档 · 上下文接不上 · 重建视图 · 重建记忆汇总 · 刷新全量记忆 ／ handoff · handover · wrap up · report · archive · lost context | 自检六步 + 缺口补写 ※ / ✅ / ✅ | [actions/handoff.md](references/actions/handoff.md) |
| ② 更新记忆 | `更新记忆` 或 `/handoff3 更新记忆` | 更新记忆 · 刷新本会话记忆 · 补全记忆 · 同步记忆 · 修复落款 ／ update memory · refresh memory · sync memory | 只读写本会话那一个 / ❌ / ❌ | [actions/update-memory.md](references/actions/update-memory.md) |
| ③ 自检 | `自检` 或 `/handoff3 自检` | 自检 · 检查记忆 · 记忆体检 · 修复记忆文件 ／ self-check · check memory · memory check · memory repair | 格式与状态可跨文件，条目只改自己的 / ❌ / ❌ | [actions/selfcheck.md](references/actions/selfcheck.md) |
| ④ 基于记忆 | **须** `/handoff3 基于记忆 [想说的话]` | 基于记忆 · 按记忆 · 根据记忆 ／ from memory · based on memory | 只写本轮新结论 / 落后才重建 / ❌ | 见下段 |
| ⑤ 帮助 | `帮助` 或 `/handoff3 帮助` | 帮助 ／ help | ❌ / ❌ / ❌ | [actions/help.md](references/actions/help.md) |

> **※ ①的轻入口**：触发词为「重建视图 / 重建记忆汇总 / 刷新全量记忆」（英：rebuild / rebuild view / refresh view / rebuild memory summary）时，只按 [schema.md](references/schema.md) §2.4 覆盖写 `handoff_memory.md`——**不补写、不出报告、不写 `mem_`，也不加载 actions/handoff.md**；回报一句「视图已重建（N 文件 / M 条）」（N 含旧格式聚合文件）。其余触发词走完整交接。本轮没碰 `mem_` → 落款用 **B**。**消歧**：含「全量」→ 本条轻入口；含「本会话 / 当前」→ ②更新记忆；英文裸 `rebuild` 的归属见下条收紧规则。

**前缀规则**：仅④必须带 `/handoff3` 前缀，**裸说「基于记忆」不触发**；①②③⑤带不带都行。`/handoff3 <触发词>` 以触发词为准，不走完整交接；裸 `/handoff3`（无触发词）才是交接。宽泛词「生成 / 报告 / 文档」按字面也算交接（用户已接受误触发）。

**英文单词收紧**：裸 `help` 只在**整条消息只有这一个词**时算⑤；出现在句子里（如 “help me debug this”）不算，须写 `handoff3 help` 或 `/handoff3 help`。裸 `rebuild` 同理：整条只有这一个词 → ①轻入口（刷视图）；出现在句子里（如 “rebuild the report”）→ 按①完整交接；`rebuild view` / `refresh view` / `rebuild memory summary` 不论位置都走轻入口。

**④ 基于记忆**：先比对 `handoff_memory.md` 末条时间与各 `mem_` 末条时间，落后或不存在就按 [schema.md](references/schema.md) §2.4 重建视图，再从视图作答；「基于记忆」后面写的任何内容就是本轮任务。只写本轮新结论，不写报告。

**纯记忆轮边界**：没命中上表任何动作时，只碰自己会话的 `mem_` 文件，绝不读写 `handoff_report.md`、也不重建 `handoff_memory.md`——不因为「换了窗口」「发了个继续」「视图落后于报告」就自作主张刷产物。

**执行正文都在 `references/actions/`**：本文件只负责路由与共用规则，不复述流程。指针旁边那句链式摘要只是防漏读的骨架，**未加载对应动作文件不得凭摘要执行**。

### 3.1 反馈迭代（当场改 · 需你确认才入库）

用户指出产物问题（如「背景太笼统」）：当场重写该段，再问「以后都这样吗？」。答「是」→ 把改法写成一条**可判断**的判据句追加到 `handoff_patches.md`（不存在就新建）；答「不用」→ 只改本次。判据句要能判断（「背景须点明谁/要什么/卡在哪并引一处 quote」，不写「背景要具体」）。**与项目 AGENTS.md 冲突时**：句末注明「覆盖 AGENTS.md 的 <条目>」，视为用户最新指令（优先级高于 AGENTS.md）——这是 patches 能进化的前提，否则会被优先级压成死锁。定期问「这条删掉会不会犯错？不会就删」。

## 4. 硬规则

1. **每条结论带 quote**（原文逐字片段），内嵌显示在产物里——证据摆明面，读者自行核对；不设事后校验脚本。
2. **原始材料缩写进正文**：代码 / 配置 / 数据 / 论文 / ppt / word 等成块材料，读懂后缩写为一句结论，句末标 `路径:行` 或文档名；不贴原文、不切片、不设专块。
3. **`[出处]` 按位置分四档**（管产物）：本技能 references → 文件名；`$LIB` 内 → 文件名或相对路径（**库根绝对路径只在报告「依据」行出现一次**）；项目根内 → 相对项目根；根外（会话记录等）→ 绝对路径、运行时获取。细则见 [schema.md](references/schema.md) §5。
4. **技能本体不写绝对路径**（管技能文件）：SKILL.md / references 自身不出现盘符、不出现用户目录名。
5. **约束与期待逐字进 pin 区**（pin 区＝报告开头逐字保留的硬约束区），永不压缩；冲突时更晚的显式约束胜，旧项留在决策链不静默删。
6. **采纳归人**：SKILL.md 与 `mem_` 格式钉死，只有 `handoff_patches.md` 可进化（**判据句规则源，每轮加载、非档案**），且需用户确认。
7. **省 token**：禁止把会话记录整体读进上下文。查缺口、捞命中行、写几笔是必要开销；搬原料的大钱必须省。
8. **冻结即只读**：状态 `已沉淀` 的文件除内容错别字外一律不动；只有用户明确指名才解冻。
9. **无配置层**：本技能所有行为按规则固定执行，不读任何配置文件；用户当轮明确要求的，按硬规则 5 覆盖固定规则。
10. **写入白名单**：所有写入只限 `$LIB` 内，`$LIB` 外一律不写——这是正向规则，不按 agent 名或目录名穷举；新增 agent / 新目录自动落入"禁止"集合。"参考某 agent 记忆"仅指读取，不构成写入授权。**唯一豁免**：用户当轮明确指定具体路径（如指名要改某个文件）时才可写 `$LIB` 外，但仍须遵守 personal_files 安全规则；用户的泛指（"和 agent 记忆一起用"之类）不构成授权。规则优先级：用户当轮指令 > 项目 AGENTS.md > `handoff_patches.md` 判据句 > 本技能正文；用户答"是"确认的 patch 与项目 AGENTS.md 冲突时视为用户最新指令、可覆盖 AGENTS.md。

## 5. 交接（仅①）

执行正文在 [actions/handoff.md](references/actions/handoff.md)：`自检 → 缺口补写 → 汇总 → 重建视图 → 渲染 → 交人定夺`。

- **唯一写报告的动作**；**已含自检六步**，不必先单独发「自检」。
- 顺序约束（S4 判定必须先于缺口补写完成）、写入边界、验收与失败模式都在该文件里。
- **未加载该文件不得执行交接**——只凭这行摘要动手，必然漏步。

## 6. 自检摘要

六个子步骤**固定全执行**：S1 结构 / S2 命名 / S3 归属 / S4 可达性 / S5 覆盖 / S6 缺档。判据与修法全文见 [actions/selfcheck.md](references/actions/selfcheck.md)，本文件不复述。

**两个入口共用同一套判据**：交接内嵌（见 actions/handoff.md 的「自检」节，结果并入报告，报告外只说一句总数）；独立「自检」动作（逐条回报直接对用户）。**内嵌时不得再对对话输出第二份逐条回报**。

读写范围与越界边界见 [actions/selfcheck.md](references/actions/selfcheck.md) §0，本文件不复制。

## 7. adapter 摘要

agent 私有能力全在 `references/adapters/`，主流程只认五个接口：`agent_name` / `session_id` / `session_record_path` / `record_format` / `token_output`。已提供 `workbuddy` / `deepseek-harness` / `generic`（兜底），新增 agent 只加一个 md。见 [adapters/index.md](references/adapters/index.md)。

## 8. 明确不做

主题组与分组算法、多档预算与 token 收缩、事后校验脚本、网页编辑器、MCP/钩子、权重评分打分卡、DSPy/GEPA/DPO、复制式交叉验证、整读大文件、**自动把多会话文件合并成单文件**（碎片由视图统一，合并只会引入新的归属歧义）。

## 9. 文件参考

- `references/actions/` — 四个动作正文：[handoff](references/actions/handoff.md) / [update-memory](references/actions/update-memory.md) / [selfcheck](references/actions/selfcheck.md) / [help](references/actions/help.md)
- [schema.md](references/schema.md) 结构定义 · [types.md](references/types.md) 八类判据 · [template.md](references/template.md) 报告骨架
- `references/adapters/` — [接口与降级](references/adapters/index.md) + [workbuddy](references/adapters/workbuddy.md) / [deepseek-harness](references/adapters/deepseek-harness.md) / [generic](references/adapters/generic.md)
