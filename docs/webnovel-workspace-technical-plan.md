# InkOS WebNovel 网文写作工作台改造技术方案

> 状态：Proposal / Implementation Blueprint  
> 日期：2026-10-07  
> 目标仓库：greedy-hat/inkos-web-novel  
> InkOS 基线：8fc2ae57080b9821257dee3e37cc677e2b6f389a（InkOS 2.0 creative workflows）  
> 参考项目：Vaxue/ApiSaverWriter @ b51307f09bd578e2dfde80bee0faacdae233f2a9  
> 许可证背景：InkOS 为 AGPL-3.0-only；ApiSaverWriter README 声明 AGPL-3.0-or-later。本文优先建议“行为与架构层借鉴 + 独立实现”，避免直接复制实现文件。

---

## 1. 结论与改造方向

本项目应当采用：

**InkOS Core / Harness 作为唯一执行内核和 Canonical Story State，新增 WebNovel 领域模块与 Studio 网文工作台；参考 ApiSaverWriter 的长篇网文工作流、上下文配额、人物状态、知识图谱、伏笔提醒和全书巡检设计。**

不建议反过来把 InkOS Harness 嵌进 ApiSaverWriter，也不建议在 InkOS 中并存第二套章节写作 Runtime。

核心原则：

1. InkOS Canonical Story State 是唯一事实源。
2. Memory DB、Knowledge Graph、搜索索引、Dashboard 统计都只是可重建 Projection。
3. Studio 只是交互层，不拥有独立的故事事实。
4. WebNovel 新模块通过现有 Harness Action / Result / Confirmation / Workspace 机制执行重操作。
5. 章节正文、状态变化、伏笔变化、时间线变化必须在同一个安全章节事务中校验并提交。
6. ApiSaverWriter 值得借鉴的是“作者工作流和上下文策略”，不是它的整套 Runtime。
7. V1 先解决职业网文作者每天会遇到的问题：续写、承接、吃设定、伏笔、人物状态、审稿与上下文可解释性。

---

## 2. 调研结论与代码依据

### 2.1 InkOS 当前已经具备的底座

目标仓库当前已经具备以下关键基础，不需要重写：

- packages/core/src/harness/
  - 统一 Action/Result 执行面。
  - Profile、Capability Registry、Context Compiler。
  - Artifact 校验、修订、执行证据。
  - 章节/作品工作区与恢复机制。
- packages/core/src/utils/memory-retrieval.ts
  - 章节摘要、Hook、卷摘要进入 story memory 检索。
  - SQLite FTS5/BM25 先做 lexical retrieval。
  - 再由 semantic selector 对候选进行语义选择。
  - 上一章摘要始终保留，较早记忆按相关性进入上下文。
- packages/core/src/retrieval/local-search.ts
  - SQLite FTS5/BM25 的统一本地搜索核。
  - 原始文件是权威源，索引只是可重建投影。
- packages/core/src/models/input-governance.ts
  - 已经有检索 trace / selection mode / token budget 的治理模型。
- packages/studio/src/components/ChapterWorkspacePanel.tsx
  - 已有章节 Workspace 可作为三栏式网文编辑器的演进基础。
- packages/studio/src/components/sidebar/
  - 已有 Pending Hooks 等展示，可升级为网文专用工作台。

因此，网文工作台不应该另造一套数据库和写作 Agent，而应该扩展这些现有能力。

### 2.2 ApiSaverWriter 最值得吸收的设计

参考项目中对本方案最有价值的实现集中在：

- sidecars/agent-runtime/src/context/context-optimizer.ts
  - 按“剧情 / 战斗 / 情感 / 转场”设置上下文权重。
  - 最近章节正文得到显著预算。
  - 人物卡 currentState + stateHistory 进入动态上下文。
  - Knowledge Graph 根据当前任务选点并沿强关系扩展。
  - 上下文压缩时保留头部设定与最新状态。
- sidecars/agent-runtime/src/graphs/chapter-write.graph.ts
  - 写前检查。
  - 章节计划 -> 正文 -> Review 的明确流程。
  - Hybrid retrieval 不可用时降级到 FTS5。
  - 伏笔到期可进入章节 review。
- sidecars/agent-runtime/src/main.ts
  - 章节记忆包含人物状态变化、知识变化、伏笔变化、时间线、canon facts 等结构化字段。
  - Whole-book consistency audit 使用聚合后的章节记忆，而不是每次扫描全书正文。
- desktop-app/src/platform.ts
  - 移动端调用也保持 book.audit 等明确协议。
- 产品层
  - 人物卡、知识图谱、记忆中心、文风、拆书、榜单、全书审查和伏笔提醒均围绕网文作者日常工作流组织。

### 2.3 不应照搬的部分

不建议直接引入：

- 第二套 LangGraph / agent-runtime。
- 第二套本地故事事实数据库。
- 第二套章节状态解释逻辑。
- 依赖 ApiSaver 中转体系的 Provider 绑定。
- 与 InkOS Studio 重复的前端路由、设置、模型配置。
- 把 Knowledge Graph 作为事实源。
- 把自动 Audit 结果直接写回正文或 Canon。

---

## 3. 产品定位

新增产品模块名称建议：

**InkOS WebNovel — 网文写作工作台**

定位：

> 从大纲、人物、伏笔、时间线和长期记忆，到百万字连续生产与全书一致性管理的一站式 AI 网文工作台。

目标用户：

- 番茄、起点、飞卢等长篇网文作者。
- 单书 50 万～300 万字的长期连载。
- 作者主导、AI 辅助。
- AI 批量生产、作者审稿。
- 希望随时知道“模型到底看到了什么”的高可控用户。

V1 非目标：

- 不做平台自动发布。
- 不做全套移动原生 App。
- 不做新的模型网关。
- 不做云协作。
- 不做向量数据库硬依赖。
- 不改变 InkOS 现有短篇、剧本、Play、翻译工作流的默认行为。

---

## 4. 总体架构

~~~mermaid
flowchart TB
  UI[InkOS Studio / WebNovel Workspace]
  API[Studio API]
  WV[WebNovel Domain Module]
  H[InkOS Agent Harness]
  CC[Context Compiler]
  MR[Memory Retrieval]
  GR[Graph Projection]
  AU[Audit Engine]
  WS[Safe Chapter Workspace]
  CANON[Canonical Story State]
  PROJ[Rebuildable Projections]
  LLM[Model Router / LLM]

  UI --> API
  API --> WV
  WV --> H
  H --> CC
  CC --> MR
  CC --> GR
  H --> LLM
  H --> WS
  WS --> CANON
  CANON --> PROJ
  PROJ --> MR
  PROJ --> GR
  WV --> AU
  AU --> PROJ
  AU --> CANON
~~~

必须保持的数据方向：

**Canonical State -> Projection -> Retrieval / UI**

禁止：

**UI / Graph / Search Index -> 直接成为 Canonical State**

---

## 5. 包结构建议

### 5.1 新增 packages/webnovel

建议新增内部包：

~~~text
packages/
  core/
  cli/
  studio/
  webnovel/
    package.json
    tsconfig.json
    src/
      index.ts
      contracts/
      context/
      projections/
      hooks/
      timeline/
      characters/
      graph/
      audit/
      production/
      migration/
      metrics/
      __tests__/
~~~

包名建议：

~~~json
{
  "name": "@actalk/inkos-webnovel",
  "version": "2.0.0",
  "private": true
}
~~~

第一阶段设为 private，避免尚未稳定的领域 API 被外部依赖。成熟后再决定是否发布。

### 5.2 Studio 目录

~~~text
packages/studio/src/features/webnovel/
  routes.ts
  api.ts
  store/
  components/
    WebNovelDashboard.tsx
    ChapterWorkbench.tsx
    ChapterTree.tsx
    ChapterEditor.tsx
    CopilotPanel.tsx
    ContextInspector.tsx
    CharacterCenter.tsx
    HookBoard.tsx
    TimelineView.tsx
    KnowledgeGraphView.tsx
    MemoryCenter.tsx
    AuditCenter.tsx
    ProductionCenter.tsx
  pages/
    DashboardPage.tsx
    ChapterPage.tsx
    CharactersPage.tsx
    HooksPage.tsx
    TimelinePage.tsx
    GraphPage.tsx
    MemoryPage.tsx
    AuditPage.tsx
    ProductionPage.tsx
~~~

原则：

- 领域逻辑放 packages/webnovel。
- React 组件只做展示、编辑和调用 API。
- 不在 React state 中维护“另一份人物状态”。

---

## 6. Canonical Story State 设计

### 6.1 数据分层

建议把数据分为四层。

#### A. Canonical

必须由 InkOS 持久化、校验、可回滚：

- 当前人物状态。
- 当前地点 / 时间。
- 世界规则。
- 已发生事件。
- 物品归属。
- 关系的已确认变化。
- 伏笔生命周期。
- 章节摘要。
- 时间线事件。
- 章节状态快照。

#### B. Derived Projection

可从 Canonical 重建：

- Knowledge Graph。
- Memory Search Index。
- 人物关系视图。
- 时间线 UI。
- 统计指标。
- Dashboard 风险卡。
- 章节相关性索引。

#### C. Advisory

模型分析结果，不自动成为事实：

- Audit Issue。
- AI 修订建议。
- 下一章建议。
- 节奏评分。
- 商业性评分。
- 对标作品建议。

#### D. Ephemeral

一次运行临时数据：

- Context Pack。
- Retrieval candidates。
- Semantic selected IDs。
- Token budget。
- Workspace draft。
- Review draft。

---

## 7. WebNovel 领域模型

以下 Schema 名称是建议，最终应采用 Zod 并复用 InkOS 现有 runtime-state 类型。

### 7.1 WebNovelProjectConfig

~~~ts
interface WebNovelProjectConfig {
  enabled: boolean;
  platform?: "fanqie" | "qidian" | "faloo" | "generic";
  targetDailyWords?: number;
  targetChapterWords?: number;
  chapterWordsTolerance?: number;
  recentChapterPolicy: {
    protectedCount: 1 | 2;
    maxBytesPerChapter: number;
  };
  contextProfileMode: "auto" | "manual";
  auditPolicy: {
    chapterReview: boolean;
    wholeBookAuditInterval?: number;
  };
  hookPolicy: {
    overdueAfterChapters: number;
  };
}
~~~

此配置不是故事事实，可以存到 book/profile 配置层。

### 7.2 ChapterContextProfile

ApiSaverWriter 使用四类 Profile。建议 InkOS 扩展为更符合网文生产的八类：

- plot：剧情推进。
- battle：战斗。
- emotion：情感。
- transition：转场。
- reveal：揭秘。
- climax：高潮。
- daily：日常。
- ensemble：群像。

~~~ts
type ChapterContextProfile =
  | "plot"
  | "battle"
  | "emotion"
  | "transition"
  | "reveal"
  | "climax"
  | "daily"
  | "ensemble";
~~~

Profile 可以由 Planner 自动给出，也允许作者覆盖。

### 7.3 Character Canonical State

建议不要新造完整人物数据库，而是在现有角色/状态模型上补充可追踪字段：

~~~ts
interface CharacterCurrentState {
  characterId: string;
  location?: string;
  realmOrPower?: string;
  physicalState?: string;
  emotionalState?: string;
  publicIdentity?: string;
  privateIdentity?: string;
  possessions?: string[];
  knownFacts?: string[];
  unknownFacts?: string[];
  relationshipDeltas?: Array<{
    targetCharacterId: string;
    relation: string;
    delta?: number;
    evidenceChapter: number;
  }>;
  lastConfirmedChapter: number;
}
~~~

人物中心读取“当前状态 + 状态历史”，但历史不是简单覆盖旧值，而是由章节 mutation 记录产生。

### 7.4 Timeline

~~~ts
interface WebNovelTimelineEvent {
  id: string;
  chapter: number;
  storyTime?: string;
  elapsed?: string;
  location?: string;
  participants?: string[];
  event: string;
  evidence?: string;
  confidence: "confirmed" | "explicit-relative" | "unknown";
}
~~~

注意：

- 模型没有足够证据时必须允许 unknown。
- 禁止为了“补齐时间线”自动编日期。
- Timeline mutation 与章节 Canon 提交一起进入 Workspace。

### 7.5 Hook 生命周期

建议扩展当前 HookRecord，而不是建立第二套伏笔系统：

~~~ts
interface WebNovelHookMeta {
  hookId: string;
  plantedChapter: number;
  lastProgressChapter?: number;
  expectedPayoffStart?: number;
  expectedPayoffEnd?: number;
  overdueAfterChapters?: number;
  importance: "main" | "major" | "minor";
  relatedCharacterIds?: string[];
  relatedEntityIds?: string[];
  seedEvidence?: string;
  progressEvidence?: Array<{
    chapter: number;
    evidence: string;
  }>;
  payoffEvidence?: string;
}
~~~

状态继续沿用 InkOS 的 unresolved / resolved / superseded 等既有语义。

“到期”是计算属性，不应直接替代 status：

~~~text
isOverdue =
  unresolved
  AND currentChapter - lastProgressChapter >= threshold
~~~

### 7.6 Knowledge Graph Projection

Graph 是 Projection。

~~~ts
interface WebNovelGraphNode {
  id: string;
  type: "character" | "location" | "organization" | "item" | "ability" | "event" | "secret";
  label: string;
  canonicalRef: string;
  lastConfirmedChapter?: number;
}

interface WebNovelGraphEdge {
  id: string;
  source: string;
  target: string;
  relation: string;
  weight: number;
  canonicalRefs: string[];
  lastConfirmedChapter?: number;
}
~~~

Graph 重建入口：

~~~text
Canonical Story State
+ Character Matrix
+ Hooks
+ Timeline
+ Chapter Summaries
-> Graph Projector
-> graph.json / sqlite projection
~~~

禁止：

- 用户拖拽一条 Graph Edge 后立刻改 Canon。
- LLM 从 Graph 推断“未明确发生的关系”并写成事实。

如用户在图谱上手工修改，应转化为“Proposed Canon Mutation”，走确认与校验。

### 7.7 Audit Issue

~~~ts
interface WebNovelAuditIssue {
  id: string;
  severity: "high" | "medium" | "low";
  category:
    | "character"
    | "timeline"
    | "world-rule"
    | "naming"
    | "hook"
    | "causality"
    | "duplication"
    | "continuity";
  summary: string;
  chapters: number[];
  evidence: Array<{
    chapter: number;
    source: string;
    excerpt: string;
  }>;
  suggestion?: string;
  status: "open" | "ignored" | "fixed" | "invalid";
  generatedAt: string;
}
~~~

Audit Issue 属于 Advisory 层。

只有用户点击“应用修复”后，才生成 Workspace revision。

### 7.8 Context Trace

这是网文工作台必须新增的核心产品能力。

~~~ts
interface WebNovelContextTrace {
  runId: string;
  chapter: number;
  profile: ChapterContextProfile;
  totalBudgetTokens: number;
  protected: ContextTraceItem[];
  retrieved: ContextTraceItem[];
  graphExpanded: ContextTraceItem[];
  omitted: ContextTraceItem[];
  budgetBySection: Record<string, number>;
  retrievalQuery: string;
  semanticSelectedIds: string[];
}
~~~

Context Inspector 必须能回答：

- AI 看到了哪些最近正文？
- 命中了哪些旧章节？
- 哪些人物卡进入本章？
- 哪些伏笔进入本章？
- 哪些资料因为预算被裁掉？
- 检索 query 是什么？
- 为什么第 108 章被选中？
- 当前 Profile 是什么？

---

## 8. 上下文编排：最重要的核心改造

### 8.1 现状问题

InkOS 已经有很好的长期 Memory Retrieval，但网文写作还需要更明确的“近期正文保护”。

只依赖摘要，即使设定不出错，也容易发生：

- 对话语气断裂。
- 上一章动作接不上。
- 场景空间连续性丢失。
- 章末悬念被摘要压平。
- 角色当前情绪失真。

ApiSaverWriter 的做法是直接给 previousChapters 高预算，这一点应吸收。

### 8.2 Protected Context

新增“不可与普通检索竞争”的 Protected Context：

优先级 P0：

1. 当前章纲 / Author Instruction。
2. Canon 当前状态。
3. 上一章正文。
4. 上一章结尾关键窗口。
5. 本章明确点名的人物 / 物品 / 地点。
6. 明确到期的主线 Hook。

优先级 P1：

- 上上章正文。
- 直接相关人物状态。
- 当前场景地点。
- 最近时间线。
- 当前卷目标。

优先级 P2：

- BM25 + semantic selector 历史记忆。
- Graph 邻接扩展。
- 参考资料。
- Skills。

P0 不能被普通 BM25 结果挤掉。

### 8.3 Dynamic Context Profile

初始权重建议如下，实际数值应由 benchmark 调优：

| Section | 剧情 | 战斗 | 情感 | 转场 | 揭秘 | 高潮 | 日常 | 群像 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| recent prose | 28% | 30% | 24% | 36% | 20% | 26% | 32% | 20% |
| character state | 14% | 20% | 24% | 10% | 12% | 18% | 18% | 24% |
| memory | 24% | 14% | 24% | 16% | 30% | 20% | 18% | 22% |
| hooks | 10% | 8% | 12% | 8% | 18% | 16% | 8% | 10% |
| world / timeline | 12% | 18% | 8% | 16% | 12% | 12% | 10% | 12% |
| skills / references | 12% | 10% | 8% | 14% | 8% | 8% | 14% | 12% |

注意：这些只是 dynamic pack 的分配，固定 System Prompt 与硬规则不算在表内。

### 8.4 Profile 自动判定

Planner 返回：

~~~json
{
  "profile": "reveal",
  "profileConfidence": 0.82,
  "secondaryProfile": "emotion"
}
~~~

当 confidence < 0.6：

- 默认 plot。
- 或使用上一个章节 Profile。
- 不应额外调用一个昂贵模型仅为分类，除非当前 Planner 本来就需要调用。

### 8.5 Retrieval Pipeline

建议上下文生成链：

~~~text
Author Instruction
+ Chapter Outline
+ Current Canon
        |
        v
Build Retrieval Query
        |
        v
FTS5/BM25 candidate retrieval
        |
        v
Semantic selector / rerank
        |
        +------> Historical summaries
        +------> Unresolved hooks
        +------> Volume summaries
        +------> Book-bound references
        |
        v
Graph seed selection
        |
        v
1-hop strong-edge expansion
        |
        v
Deduplicate by canonical source
        |
        v
Profile budget allocation
        |
        v
Protected + Dynamic context pack
        |
        v
Context Trace
~~~

### 8.6 Graph 扩展规则

借鉴 ApiSaverWriter，但严格限制：

- 只扩展 1 hop。
- 强关系优先。
- 最多 16～20 个节点。
- Graph 不能覆盖 Canon。
- Graph 中没有 canonicalRefs 的边不进入 Writer context。
- 避免因为“角色认识某人”自动把整个人物网络全部塞进去。

### 8.7 长期记忆与最近正文的职责分离

明确：

- Recent Prose 负责“语言、动作、场景、情绪连续性”。
- Chapter Summary 负责“事件历史”。
- Character State 负责“当前事实”。
- Hook State 负责“未完成承诺”。
- Timeline 负责“时间约束”。
- Graph 负责“实体关系导航”。
- Volume Summary 负责“大跨度阶段目标”。
- Reference Materials 负责“外部资料”。

禁止让单一 summary 承担全部职责。

---

## 9. WebNovel Chapter Pipeline

建议将现有 InkOS 写章能力组合成 WebNovel 专用高层 Action，而不是重写 Writer。

### 9.1 Pipeline

~~~mermaid
flowchart LR
  A[Preflight] --> B[Context Build]
  B --> C[Chapter Plan]
  C --> D[Draft]
  D --> E[Chapter Review]
  E --> F{Pass?}
  F -- no --> G[Revise]
  G --> E
  F -- yes --> H[Settle State]
  H --> I[Workspace Validation]
  I --> J[Atomic Commit]
  J --> K[Projection Rebuild]
~~~

### 9.2 Preflight

检查：

- 当前章纲是否存在。
- 上一章是否完整。
- Canon 是否可解析。
- 是否存在 stale workspace。
- 是否存在未解决的高风险 Audit Issue。
- Profile 是否可判定。
- 当前章节是否有到期 Hook。
- 目标字数是否合法。

Preflight 默认不阻断低风险问题。

阻断条件只包括：

- Canon 无法加载。
- 章节编号异常。
- Workspace 未恢复。
- 关键状态 schema 校验失败。
- 用户要求“严格章纲”但章纲为空。

### 9.3 Plan

Plan 输出应包含：

- 承接锚点。
- 人物位置。
- 人物当前目标。
- 当前冲突。
- 本章事件链。
- 本章信息增量。
- Hook 推进。
- 时间变化。
- 章末钩子。
- Context Profile。

### 9.4 Draft

Writer 只能依据 Context Pack。

不允许 Writer 自己偷偷调用新的搜索流程，避免：

- Planner 和 Writer 使用不同事实。
- 运行不可复现。
- Context Trace 无法解释。

如果确实需要工具调用，必须通过 Harness 工具并记入 trace。

### 9.5 Review

单章 Review 重点：

- 章纲一致性。
- 人物状态。
- 角色已知/未知信息。
- 时间线。
- 物品归属。
- Hook 到期与推进。
- 因果。
- 重复段落。
- 章末钩子。

Review 不做“全书大扫描”。

### 9.6 Settle

Settler 从最终正文抽取：

- chapter summary。
- character mutations。
- timeline mutations。
- hook mutations。
- entity / relation evidence。
- new canon facts。
- location / item ownership changes。

所有 Mutation 带 evidence。

### 9.7 Atomic Commit

必须复用并强化 InkOS 的安全工作区：

~~~text
draft.md
chapter-summary mutation
character-state mutation
timeline mutation
hook mutation
graph evidence
run snapshot
review result
        |
        v
validate as one unit
        |
        v
atomic commit
~~~

任何一个关键 canonical mutation 校验失败：

- 不推进 Canon。
- 不标记 Hook resolved。
- 不更新 Timeline。
- 正文保持在 Workspace。
- UI 提供“修复状态 / 放弃草稿 / 手工确认”入口。

---

## 10. Harness Action 设计

建议新增能力，不建立新的 Agent Loop。

### 10.1 只读 Action

~~~text
webnovel.dashboard.get
webnovel.context.preview
webnovel.memory.search
webnovel.character.get
webnovel.character.history
webnovel.hook.list
webnovel.timeline.list
webnovel.graph.get
webnovel.audit.list
webnovel.chapter.health
~~~

### 10.2 生成 Action

~~~text
webnovel.chapter.plan
webnovel.chapter.write
webnovel.chapter.review
webnovel.chapter.revise
webnovel.audit.run
webnovel.hook.suggest
webnovel.timeline.rebuild
webnovel.graph.rebuild
~~~

### 10.3 需要确认的修改 Action

~~~text
webnovel.chapter.commit
webnovel.audit.apply-fix
webnovel.character.apply-mutation
webnovel.hook.apply-mutation
webnovel.timeline.apply-mutation
~~~

### 10.4 批量生产 Action

V3：

~~~text
webnovel.production.start
webnovel.production.pause
webnovel.production.resume
webnovel.production.cancel
~~~

批量生产仍然逐章进入现有 Harness，不允许绕过章节事务。

---

## 11. Studio 网文写作工作台

### 11.1 一级入口

Studio 新增“网文工作台”。

进入某本长篇作品后，主导航：

~~~text
总览
章节
大纲
人物
伏笔
时间线
关系图
记忆
审稿
生产
~~~

### 11.2 Dashboard

首页展示：

- 当前总字数。
- 章节数。
- 当前卷。
- 今日字数目标。
- 最近 7 天产量。
- 当前主线。
- 下一章建议。
- 高风险一致性问题。
- 逾期伏笔。
- 长期未出现的重要人物。
- 连续多章无主线推进提醒。

风险提醒必须来自可解释规则或 Audit Issue，不要使用模糊“AI 感觉”。

### 11.3 章节三栏工作台

布局：

~~~text
┌─────────────┬──────────────────────────────┬─────────────────┐
│ 章节 / 卷树  │ 正文编辑器                   │ AI Copilot      │
│             │                              │                 │
│ 第一卷      │ 第 428 章                    │ 本章计划        │
│  426        │                              │ 当前状态        │
│  427        │ 正文……                       │ 相关记忆        │
│  428 <-     │                              │ 待回收伏笔      │
│  429        │                              │ 时间线          │
│             │                              │ Context Trace   │
└─────────────┴──────────────────────────────┴─────────────────┘
~~~

AI Copilot 面板不是普通聊天窗口，而是“结构化上下文 + 可调用操作”。

### 11.4 Context Inspector

这是 V1 必须交付。

展示：

~~~text
本章 Profile：揭秘 + 情感

Protected
✓ 第427章全文
✓ 当前人物状态：林凡 / 苏清雪
✓ H-071 苏清雪身世

Retrieved
✓ 第118章 玉佩凤凰纹
✓ 第203章 陈长老看到玉佩
✓ 第377章 北域使者称其“殿下”

Graph expanded
✓ 北域皇族 -> 凤凰纹
✓ 陈长老 -> 北域

Omitted
○ 第54章支线：预算不足
○ 北境战争资料：相关性低
~~~

每项支持“查看来源”。

### 11.5 人物中心

页面结构：

- 基本设定。
- Canon 当前状态。
- 已知信息。
- 未知信息。
- 所持物。
- 当前地点。
- 关系。
- 状态历史。
- 最近出场章节。
- 相关 Hook。
- 相关 Audit Issues。

“编辑当前状态”必须生成 mutation proposal，而不是直接改 projection。

### 11.6 伏笔中心

列表支持：

- 主线 / 角色 / 情感 / 世界观分类。
- 埋设章节。
- 最近推进章节。
- 预计回收区间。
- 逾期状态。
- 关联人物。
- Evidence。
- 回收建议。

支持看板：

~~~text
新埋设 | 推进中 | 待回收 | 逾期 | 已回收
~~~

### 11.7 Timeline

双视图：

- 按故事时间。
- 按章节。

时间信息不足时显示“未明确”，禁止 UI 自动伪造日期。

### 11.8 Knowledge Graph

技术上可继续使用 Studio 已有的 @xyflow/react。

交互规则：

- 默认只读。
- 节点点击打开 Canon 来源。
- Edge 点击显示 evidence。
- 手工变更必须转 Proposed Mutation。
- 支持“只看本章相关图谱”。

### 11.9 Memory Center

统一查询：

- Chapter Summary。
- Volume Summary。
- Hook。
- Timeline。
- Character State History。
- Canon Fact。
- Reference Materials。

搜索结果必须显示来源与章节。

### 11.10 Audit Center

页面：

~~~text
全书健康度
人物一致性
时间线
设定
伏笔
因果
重复剧情
~~~

Issue 操作：

- 查看证据。
- 忽略。
- 标记误报。
- 请求修复方案。
- 在 Workspace 中应用修订。
- 修复后复审。

---

## 12. Whole-book Audit

### 12.1 输入原则

借鉴 ApiSaverWriter：默认不把所有正文一次塞给模型。

输入：

- 世界规则。
- 总纲 / 卷纲。
- 人物当前状态与关键历史。
- 逐章结构化 summary。
- timeline events。
- unresolved hooks。
- important canon facts。

### 12.2 分层 Audit

建议两层。

#### Fast deterministic audit

不调用模型即可发现：

- 同一人物多个 canonical id。
- Hook 超期。
- Timeline 排序异常。
- resolved Hook 仍被标记 active。
- 物品同时存在多个 owner。
- 角色在同一 storyTime 出现在互斥地点。
- chapter summary 缺失。

#### LLM semantic audit

负责：

- 人物认知冲突。
- 因果断裂。
- 设定规则软冲突。
- 名称混用。
- 重复剧情。
- 伏笔回收与 seed 不匹配。

### 12.3 Evidence Contract

每条 high / medium issue 必须至少给：

- 2 个 source。
- chapter id。
- 短 evidence。
- 可执行建议。

没有足够 evidence：

- 降为 low，或
- 不输出。

### 12.4 不自动修改

Audit 绝不直接修改正文。

流程：

~~~text
Audit
-> Issue
-> Fix Proposal
-> User/Agent Confirm
-> Chapter Workspace
-> Review
-> Atomic Commit
~~~

---

## 13. API 设计

建议在 Studio Server 增加 webnovel route group。

~~~text
GET  /api/webnovel/:bookId/dashboard
GET  /api/webnovel/:bookId/context/:chapter
GET  /api/webnovel/:bookId/characters
GET  /api/webnovel/:bookId/characters/:id
GET  /api/webnovel/:bookId/hooks
GET  /api/webnovel/:bookId/timeline
GET  /api/webnovel/:bookId/graph
GET  /api/webnovel/:bookId/memory/search?q=
GET  /api/webnovel/:bookId/audits

POST /api/webnovel/:bookId/chapter/:chapter/plan
POST /api/webnovel/:bookId/chapter/:chapter/write
POST /api/webnovel/:bookId/chapter/:chapter/review
POST /api/webnovel/:bookId/audit
POST /api/webnovel/:bookId/projections/rebuild
~~~

写操作应继续通过现有 interaction / confirmation 机制，不应该让 REST route 绕过 Harness。

---

## 14. CLI 入口

V1 至少支持：

~~~text
inkos webnovel dashboard [book]
inkos webnovel context <chapter> [book] --json
inkos webnovel hooks [book]
inkos webnovel audit [book]
inkos webnovel graph rebuild [book]
inkos webnovel timeline rebuild [book]
~~~

目的：

- UI 不是唯一入口。
- 方便测试和 CI。
- 方便未来 OpenClaw / Agent Skill 调用。

---

## 15. Projection 设计

建议统一放到：

~~~text
story/projections/
  webnovel/
    graph.json
    timeline-view.json
    character-current.json
    dashboard.json
    audit-index.json
~~~

这些文件全部可以删除后重建。

Canonical 原始状态仍放现有 story/state 或现有权威位置。

### 15.1 Rebuild Command

~~~text
inkos webnovel rebuild --all
~~~

执行：

1. 读取 Canon。
2. 校验 schema。
3. 重建 timeline projection。
4. 重建 character current view。
5. 重建 graph。
6. 刷新 memory search scope。
7. 重建 dashboard metrics。

Projection rebuild 不调用 LLM，除非显式指定 semantic enrichment。

---

## 16. 与现有 Memory Retrieval 的改造点

重点修改：

### packages/core/src/utils/memory-retrieval.ts

保留：

- chapter summaries。
- hooks。
- volume summaries。
- FTS5/BM25。
- semantic selector。

新增：

- 支持 timeline summary documents。
- 支持 character-state-history documents。
- 支持 canon-fact documents。
- 可返回 reason / selectedBy。
- recent prose 不放进普通 memory ranking，而走 protected section。

### packages/core/src/retrieval/local-search.ts

原则上不需要大改。

可新增：

- kind weights。
- query debug tokens。
- source grouping。
- optional reciprocal rank fusion 接口，为未来 vector retrieval 留位置。

V1 不建议引入外部向量库。

---

## 17. Context Compiler 改造

重点文件：

- packages/core/src/harness/context-compiler.ts
- packages/core/src/harness/agent-context.ts
- packages/core/src/models/input-governance.ts

新增 WebNovelContextPolicy：

~~~ts
interface WebNovelContextPolicy {
  profile: ChapterContextProfile;
  recentProse: {
    protectedCount: number;
    budgetTokens: number;
  };
  characterStateBudget: number;
  memoryBudget: number;
  hookBudget: number;
  timelineBudget: number;
  graphBudget: number;
  referenceBudget: number;
  skillBudget: number;
}
~~~

Composer 最终产出：

~~~ts
interface CompiledWebNovelContext {
  protectedContext: string;
  dynamicContext: string;
  trace: WebNovelContextTrace;
}
~~~

必须避免重复注入：

例如同一事实已经存在 Canon current state，则历史 memory 中的旧状态应降低优先级或标注 historical。

---

## 18. Story State Mutation Contract

所有模型生成的状态变化都应改成“提议”，统一 schema：

~~~ts
interface CanonMutationProposal {
  type:
    | "character"
    | "timeline"
    | "hook"
    | "item"
    | "relationship"
    | "canon-fact";
  targetId: string;
  operation: "create" | "update" | "resolve" | "supersede";
  before?: unknown;
  after: unknown;
  evidence: {
    chapter: number;
    excerpt: string;
  };
}
~~~

校验规则：

- evidence 必须来自当前最终正文。
- 不能从 chapter plan 直接 settle 成事实。
- 不能从 AI suggestion settle 成事实。
- unknown 不得自动变 confirmed。
- 已 resolved Hook 不应无证据 reopening。

---

## 19. 自动生产中心

V3 再做。

### 19.1 Production Job

~~~ts
interface WebNovelProductionJob {
  bookId: string;
  startChapter: number;
  count: number;
  targetWords: number;
  maxRevisionRounds: number;
  autoCommitThreshold?: number;
  stopOnHighSeverityIssue: boolean;
}
~~~

### 19.2 每章执行

~~~text
plan
-> write
-> review
-> revise <= N
-> settle
-> validate
-> commit
-> audit lightweight
-> next
~~~

### 19.3 自动停止条件

- Canon validation failed。
- high severity continuity issue。
- LLM 连续失败。
- 章节字数严重不足。
- Workspace 无法提交。
- 用户取消。
- Token / Cost Budget 达上限。

### 19.4 自动提交

V1/V2 禁止。

V3 可增加：

~~~text
score >= threshold
AND no high issue
AND workspace valid
AND canon mutations valid
=> auto commit
~~~

默认关闭。

---

## 20. ApiSaverWriter 功能映射

| ApiSaverWriter 能力 | InkOS WebNovel 处理方式 |
|---|---|
| previousChapters 高预算 | Protected Recent Prose |
| Context Profile | WebNovelContextPolicy |
| currentState + stateHistory | Canon current state + mutation history |
| Knowledge Graph | Rebuildable projection |
| hybrid retrieval | 先沿用 InkOS FTS5/BM25 + semantic selector，预留 RRF |
| chapter memory | 扩展 InkOS structured chapter summary |
| foreshadow overdue | Hook aging computed policy |
| whole-book audit | WebNovel Audit Engine |
| review center | Studio Audit Center |
| mobile narrow layout | 后续响应式 Studio，不作为 V1 独立 App |
| book scraping / rankings | 后续独立 market-research capability |
| style management | 复用 InkOS Skills / Profile / references |

---

## 21. 数据迁移与兼容

### 21.1 现有 InkOS 项目

首次开启 WebNovel：

~~~text
inkos webnovel enable
~~~

执行：

- 不修改正文。
- 不修改现有 chapter summary。
- 不修改现有 Hook 语义。
- 创建 webnovel config。
- 生成 projections。
- 对已有状态做 schema-compatible enrichment。
- 缺少 timeline 时标记 unknown，不推断日期。

### 21.2 ApiSaverWriter 导入

可作为 V2 单独能力：

~~~text
inkos webnovel import apisaverwriter <path>
~~~

建议只迁移：

- 小说元数据。
- 世界观。
- 大纲。
- 章节正文。
- 人物卡。
- 章节记忆。
- 伏笔。
- 文风描述。

不直接迁移：

- ApiSaver 账号配置。
- API Key。
- Provider 路由。
- 云端数据。
- Runtime cache。
- 内部 id 直接作为 InkOS canonical id。

导入后必须执行 Canon rebuild + audit。

---

## 22. 测试方案

### 22.1 Unit Tests

packages/webnovel：

- profile resolver。
- budget allocator。
- hook overdue calculator。
- timeline parser。
- graph projector。
- canon mutation validator。
- context deduper。
- audit evidence validator。

### 22.2 Core Integration Tests

场景：

1. 第 1～20 章正常写作。
2. 第 5 章埋伏笔，第 40 章检索回忆。
3. 角色第 10 章受伤，第 12 章恢复。
4. 物品 ownership 转移。
5. 时间跳跃。
6. rejected chapter rollback。
7. stale workspace 恢复。
8. chapter rewrite 后 projection 重建。

### 22.3 Retrieval Regression

建立固定小说 fixture，验证：

- 当前目标提到“凤凰玉佩”时必须召回第 118 / 203 / 377 章相关 memory。
- 无关战争章节不能占满 budget。
- 上一章正文始终在 protected context。
- resolved hook 默认不召回，除非 Audit / history view。

### 22.4 Long-run Simulation

至少增加：

- 200 章快速 CI fixture。
- 1000 章 nightly / manual benchmark。

指标：

- Canon mutation schema failure rate。
- Retrieval recall@K。
- Context token usage。
- Hook retrieval success。
- stale state incident。
- rollback correctness。
- audit false positive sample rate。

### 22.5 Studio E2E

Playwright：

- 打开 WebNovel Dashboard。
- 进入章节。
- 查看 Context Inspector。
- 写下一章。
- Review。
- Workspace diff。
- Commit。
- Hook 状态变化。
- Timeline 更新。
- Audit Issue -> Fix Proposal。

---

## 23. 可观测性

每次章节 run 保存：

- runId。
- profile。
- token budget。
- selected memories。
- graph expansions。
- protected sources。
- omitted sources。
- LLM model。
- review result。
- mutations。
- validation result。
- commit result。

Studio 提供“运行详情”。

目标：

用户能判断一次错误到底是：

- Retrieval 没找回来。
- Context budget 裁掉了。
- Canon 本来就错。
- Writer 忽略了明确事实。
- Settler 抽取错。
- Audit 误报。

---

## 24. 性能策略

### 24.1 不扫描全书正文

日常写章：

- 最近 1～2 章正文。
- 结构化长期记忆。
- FTS 检索。
- Canon current state。

Whole-book audit：

- 默认结构化聚合数据。
- 只在 evidence 不足时按需读取具体章节。

### 24.2 Cache

允许缓存：

- projection hash。
- compiled stable context prefix。
- graph projection。
- dashboard metrics。
- retrieval index。

不能缓存成 Canon：

- LLM 推断。
- audit suggestion。
- writer plan。

### 24.3 Token Budget

Context Trace 必须记录：

- estimated input tokens。
- 每 section tokens。
- 被裁剪数量。

之后可以用真实数据调权重，不靠感觉。

---

## 25. 安全与隐私

延续 InkOS 本地优先原则：

- Canon、本地正文、Memory Index 不上传第三方，除非模型调用所需。
- API Key 继续走现有 secrets 机制。
- Context Inspector 不显示 API Key。
- 日志不得输出 secret。
- Audit export 默认不包含密钥和服务配置。
- 导入 ApiSaverWriter 时显式忽略任何 token / key / cookie。

---

## 26. 许可证与参考项目边界

ApiSaverWriter 可作为功能和架构参考，但建议：

1. 优先重新实现行为，不直接复制文件。
2. 如果确实复制或改编代码：
   - 记录来源文件与 commit。
   - 保留版权与许可证要求。
   - 在 NOTICE / docs 中注明来源。
3. 本项目整体仍按 InkOS 当前 AGPL-3.0-only 管理。
4. 在正式合并大段第三方实现前再做一次许可证核查。

这不是法律意见；这里只给出工程上的低风险做法。

---

## 27. 分阶段实施

## Phase 0 — 领域骨架与可观测性

目标：不改变写作效果，先建立 WebNovel Domain。

交付：

- packages/webnovel。
- WebNovelProjectConfig。
- Context Trace schema。
- Projection 目录约定。
- webnovel enable / rebuild。
- 基础 Studio route。
- 基础 tests。

验收：

- 老项目不开启 WebNovel 时行为完全不变。
- 开启后可重建 projections。
- 无 LLM 调用也可完成基础 rebuild。

## Phase 1 — V1 可用网文工作台

目标：作者能真正每天使用。

交付：

- Dashboard。
- 三栏章节工作台。
- Protected Recent Prose。
- Context Profile。
- Context Inspector。
- 人物当前状态。
- Hook Board + overdue。
- Chapter Review。
- Workspace commit。

这是最优先阶段。

验收：

- 上一章正文始终可见于 Context Trace。
- 旧伏笔可按目标召回。
- 用户能知道模型使用了哪些上下文。
- 状态失败时正文不污染 Canon。

## Phase 2 — Story Intelligence

交付：

- Timeline。
- Knowledge Graph projection。
- Memory Center。
- Whole-book Audit。
- Audit Center。
- 修复提案工作流。
- ApiSaverWriter import 可选。

验收：

- Graph 可完全重建。
- Timeline unknown 不被编造。
- Audit Issue 必须带 evidence。
- Apply Fix 走 Workspace，不直接改 Canon。

## Phase 3 — Production Automation

交付：

- 多章生产队列。
- 自动 Review / revise。
- 停止策略。
- 成本与 token dashboard。
- 可选 auto commit threshold。
- 市场研究 / 榜单作为独立 capability。

---

## 28. 建议的 PR 拆分

### PR 1 — webnovel domain foundation

文件：

~~~text
packages/webnovel/**
pnpm-workspace.yaml
package.json scripts（如需要）
packages/core/src/index.ts（只导出必要 contract）
~~~

内容：

- schemas。
- config。
- projections。
- context trace。
- hook aging。
- tests。

### PR 2 — context policy + recent prose

文件：

~~~text
packages/core/src/harness/context-compiler.ts
packages/core/src/harness/agent-context.ts
packages/core/src/utils/memory-retrieval.ts
packages/core/src/models/input-governance.ts
packages/webnovel/src/context/**
~~~

内容：

- Protected Recent Prose。
- Profile budget。
- Context Trace。
- retrieval reason。

### PR 3 — WebNovel Studio MVP

文件：

~~~text
packages/studio/src/features/webnovel/**
packages/studio/src/App.tsx
packages/studio/src/components/Sidebar.tsx
packages/studio/src/api/**
~~~

内容：

- Dashboard。
- Chapter Workbench。
- Context Inspector。
- Character + Hook panel。

### PR 4 — timeline + graph

内容：

- Timeline schema / projector。
- Graph projector。
- @xyflow/react UI。
- canonical evidence links。

### PR 5 — whole-book audit

内容：

- deterministic audit。
- LLM audit。
- evidence contract。
- Audit Center。
- fix proposal。

### PR 6 — automation

内容：

- production job。
- queue。
- stop condition。
- optional auto commit。

---

## 29. 具体文件级改造清单

### 现有文件：优先修改

- packages/core/src/utils/memory-retrieval.ts
  - 新 memory kinds。
  - retrieval reason。
  - 与 webnovel context policy 对接。

- packages/core/src/retrieval/local-search.ts
  - 保持现有统一检索核。
  - 可增加 kind weight 与 trace metadata。

- packages/core/src/harness/context-compiler.ts
  - 增加 WebNovel Protected / Dynamic 两层 context。

- packages/core/src/harness/agent-context.ts
  - 将 WebNovel context trace 进入 run observation。
  - 不改变非 WebNovel Profile 行为。

- packages/core/src/harness/contracts.ts
  - 增加 WebNovel capability contracts。

- packages/core/src/harness/capability-registry.ts
  - 注册 WebNovel actions。

- packages/studio/src/components/ChapterWorkspacePanel.tsx
  - 逐步抽取通用部分；WebNovel 使用新的 ChapterWorkbench。

- packages/studio/src/components/Sidebar.tsx
  - 增加 WebNovel 入口。

### 新文件：建议

~~~text
packages/webnovel/src/contracts/config.ts
packages/webnovel/src/contracts/context.ts
packages/webnovel/src/contracts/audit.ts
packages/webnovel/src/context/profile.ts
packages/webnovel/src/context/budget.ts
packages/webnovel/src/context/compile.ts
packages/webnovel/src/hooks/aging.ts
packages/webnovel/src/timeline/projector.ts
packages/webnovel/src/graph/projector.ts
packages/webnovel/src/audit/deterministic.ts
packages/webnovel/src/audit/semantic.ts
packages/webnovel/src/projections/rebuild.ts
packages/webnovel/src/production/job.ts
~~~

---

## 30. V1 验收标准

V1 完成不是“页面能打开”，而是满足以下条件：

### 写作连续性

- 下一章始终包含上一章受保护正文。
- 可以配置是否再带上上章。
- 最近正文不会被普通 RAG 结果挤出。

### 长期记忆

- 100 章以前的相关事件能通过 memory retrieval 召回。
- Context Inspector 显示其来源。

### 人物状态

- 当前人物状态与历史状态明确分离。
- Writer 优先使用 current state。
- 旧 memory 不能覆盖新 Canon。

### 伏笔

- Hook 显示 planted / last progress / expected payoff。
- overdue 是计算结果。
- 到期 Hook 可以进入 Plan / Review。

### 安全提交

- 正文与 Canon mutation 一起校验。
- 任一关键 mutation 失败，不推进正式状态。

### 可解释性

- 每次写作均能查看 Context Trace。
- 能看到 Protected / Retrieved / Graph / Omitted 四类来源。

### 兼容

- 非 WebNovel 用户不受影响。
- 现有 Studio、CLI、TUI 继续工作。
- 现有项目可 opt-in。

---

## 31. 明确禁止的实现方式

以下方式应在 code review 中直接阻止：

1. 新建一套独立 webnovel SQLite 作为故事事实库。
2. 让 React Zustand 成为人物 / 伏笔 truth。
3. Graph edge 没 evidence 却写回 Canon。
4. Audit 自动修改正文。
5. Writer 自己绕过 Context Compiler 任意读取全书。
6. 把最近正文也丢进 BM25 竞争，导致它可能被裁掉。
7. 把“模型计划中的事件”提前写成已发生事实。
8. Timeline 信息不足时自动补日期。
9. resolved Hook 被无证据重新打开。
10. 为 WebNovel 复制一套 LLM Provider / Model Router。
11. 第一阶段就引入外部向量数据库。
12. 第一阶段就做自动发布和移动原生客户端。

---

## 32. 第一批实际开发任务

建议立即开始的顺序：

### Task 1

建立 packages/webnovel：

- config schema。
- context profile。
- context trace。
- hook aging。
- projection contracts。

### Task 2

给 Context Compiler 增加：

- WebNovel mode。
- protected recent prose。
- profile budgets。
- trace output。

### Task 3

扩展 memory retrieval：

- timeline / character state history / canon facts。
- retrieval reason。
- selected source trace。

### Task 4

Studio 增加：

- WebNovel Dashboard。
- Chapter Workbench。
- Context Inspector。
- Hook Board。

### Task 5

增加 Settle mutation contract：

- character。
- timeline。
- hook。
- item。
- relationship。

### Task 6

增加 Whole-book Audit。

这六个任务完成后，已经能形成一个明显区别于普通 AI 小说工具的 V1。

---

## 33. 最终目标架构

最终产品不应该只是“一个更好看的 InkOS 页面”，而应该形成以下能力闭环：

~~~text
作者意图
  |
  v
网文工作台
  |
  v
Planner 判定章节类型
  |
  v
Context Compiler
  |-- Protected Recent Prose
  |-- Canon Current State
  |-- FTS5/BM25 Memory
  |-- Semantic Selection
  |-- Hook State
  |-- Timeline
  |-- Graph Projection
  |-- Skills / References
  |
  v
可解释 Context Trace
  |
  v
Writer
  |
  v
Reviewer
  |
  v
Settler -> Canon Mutation Proposals
  |
  v
Safe Workspace Validation
  |
  v
Atomic Commit
  |
  v
Projection Rebuild
  |
  +--> Character Center
  +--> Hook Board
  +--> Timeline
  +--> Knowledge Graph
  +--> Memory Center
  +--> Dashboard
  +--> Whole-book Audit
~~~

这套架构的核心竞争力不是“能写一章”，而是：

**能够在数百到数千章之后，仍然知道当前故事的真实状态、为什么取用了某段历史、哪些伏笔尚未兑现，并且在生成失败时不会让正文与状态分叉。**

---

## 34. 最终建议

本项目后续开发应坚持：

**InkOS 的脑 + ApiSaverWriter 的作者工作台经验。**

具体落地就是：

- 底层继续使用 InkOS Harness。
- Canonical State 继续由 InkOS 管。
- Retrieval 继续以 InkOS 的可重建 FTS5/BM25 投影为核心。
- 吸收 ApiSaverWriter 的 Recent Prose、Context Profile、人物状态历史、Knowledge Graph、伏笔到期、Whole-book Audit。
- 新增 Context Inspector，把“AI 为什么记住/忘记某件事”变成产品能力。
- 任何 AI 结论都先是 Proposal，只有经过 Workspace Validation 才能成为 Canon。

优先级上，不要先做榜单、发布和移动端。

**第一目标应是：让一个作者真的敢用它连续写 1000 章。**
