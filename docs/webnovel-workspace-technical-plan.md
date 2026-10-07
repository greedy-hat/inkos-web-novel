# InkOS WebNovel 网文写作工作台终态改造技术方案

> 状态：Target Architecture / Final Implementation Blueprint  
> 日期：2026-10-07  
> 目标仓库：greedy-hat/inkos-web-novel  
> InkOS 基线：8fc2ae57080b9821257dee3e37cc677e2b6f389a  
> 参考项目：Vaxue/ApiSaverWriter @ b51307f09bd578e2dfde80bee0faacdae233f2a9  
> 核心要求：**不做 MVP、不引入临时架构、不以“先跑起来以后再重构”为实施策略。所有模块按最终边界、最终 Schema、最终 Action/API 和最终持久化模型设计；实施可以分批合并，但每一批都必须落在终态架构上。**

---

## 0. 决策摘要

本项目采用：

**InkOS 作为唯一 Agent Harness、唯一模型调度层、唯一 Canonical Story State 和唯一安全提交机制；新增完整的 WebNovel Domain，并在 Studio 中实现职业网文作者工作台。**

ApiSaverWriter 只作为以下能力的产品和实现参考：

- 最近章节正文保护。
- 动态 Context Profile。
- 结构化章节记忆。
- 人物 current state + state history。
- Knowledge Graph。
- 伏笔老化 / 到期提醒。
- Whole-book consistency audit。
- 文风、拆书、榜单、知识卡等作者工作流。

明确不引入：

- 第二套 Agent Runtime。
- 第二套故事事实数据库。
- 第二套 Provider / Model Router。
- 第二套章节状态解释器。
- 为了“先上线”而存在的临时字段、临时 API、临时表或临时页面。

最终目标不是“能生成下一章”，而是：

> **一个作者可以从开书、总纲、人物、世界观、拆书、文风、章节生产、长期记忆、伏笔、时间线、审稿、全书巡检，到数百万字连载，都在同一套可解释、可回滚、可重建的事实系统里完成。**

---

# 1. 为什么必须以 InkOS 为底座

InkOS 现有架构已经具备最难替代的基础：

- 统一 Agent Harness。
- Context Compiler。
- Capability Registry。
- Artifact Validation / Revision。
- Story State。
- Memory Retrieval。
- SQLite FTS5/BM25 本地检索。
- Safe Chapter Workspace。
- 运行快照、恢复与回滚。
- 多模型 Provider。
- Studio / TUI / CLI 多入口。
- Skill 系统。
- 原始文件为权威、索引为 Projection 的数据哲学。

ApiSaverWriter 的优势主要位于作者工作流和网文专用策略层。

因此正确融合方向是：

~~~text
InkOS Core / Harness
        |
        +---- WebNovel Domain
        |       |
        |       +---- Context Policy
        |       +---- Character State
        |       +---- Hook Lifecycle
        |       +---- Timeline
        |       +---- Graph Projection
        |       +---- Memory Center
        |       +---- Audit
        |       +---- Production Orchestration
        |       +---- Market / Analysis
        |
        +---- Studio WebNovel Workbench
~~~

而不是：

~~~text
ApiSaverWriter Runtime
        +
InkOS Runtime
        =
两套状态 / 两套上下文 / 两套真相
~~~

---

# 2. “不返工”的工程定义

这里的“不返工”不是“不允许修 Bug”，而是禁止架构性推翻。

## 2.1 第一行生产代码前必须冻结的内容

在实现任何 UI 或 Writer 改造前，必须先完成并评审：

1. WebNovel Domain 边界。
2. Canonical / Projection / Advisory / Ephemeral 数据分层。
3. 所有核心 Zod Schema。
4. CanonMutationProposal 协议。
5. Context Section 与优先级模型。
6. Context Trace Schema。
7. Harness Action 命名与输入输出。
8. Studio Server API 路由。
9. Projection 文件布局。
10. Migration Version。
11. Audit Issue 协议。
12. Production Job 协议。
13. Import / Export 契约。
14. Feature Capability / Permission 模型。
15. 测试 Fixture 和基准指标。

冻结后允许“向后兼容地增加字段”，不允许：

- 改 ID 语义。
- 把 Projection 提升为 Canon。
- 换另一套章节流水线。
- UI 直接写文件后再补 Domain。
- 先存一套 SQLite，之后再迁回 Story State。

## 2.2 所有分批开发都必须是终态切片

实施可以分多个 PR，但每个 PR 都必须满足：

- 使用最终 package path。
- 使用最终 schema。
- 使用最终 action name。
- 使用最终 projection path。
- 使用最终 API version。
- 不出现 temporary / legacy-v-next / mvp-only 数据结构。
- 后续 PR 只能增加实现，不需要搬家或改核心协议。

---

# 3. 最终产品范围

最终 WebNovel Workbench 包含：

| 模块 | 最终能力 |
|---|---|
| 小说总览 | 字数、章节、卷、日更目标、主线、风险、Hook、Audit、生产状态 |
| 开书向导 | 题材、平台、目标读者、卖点、核心冲突、总纲、世界观、主角、金手指 |
| 大纲中心 | 总纲 / 卷纲 / 章纲、拖拽排序、AI 规划、版本历史 |
| 章节工作台 | 章节树、正文编辑、AI Copilot、Context Inspector、Review、Diff |
| 人物中心 | 设定、当前状态、状态历史、已知/未知信息、关系、出场历史 |
| 世界观中心 | 势力、地点、规则、等级体系、物品、能力、禁则 |
| 伏笔中心 | 埋设、推进、回收、到期、Evidence、关联人物与章节 |
| 时间线 | 故事时间、章节时间、相对时间、冲突检测 |
| 关系图谱 | 人物 / 地点 / 势力 / 物品 / 能力 / 事件关系，可回溯证据 |
| 记忆中心 | Summary、Hook、Timeline、Character State、Canon Fact、Reference |
| 审稿中心 | 单章审稿、跨章一致性、全书巡检、问题修复流程 |
| 文风中心 | 文风档案、样本、风格约束、去模板化、绑定到作品 |
| 拆书中心 | 导入作品、章节结构、节奏、技法、风格、可复用 Skill |
| 对标 / 市场 | 榜单、题材、标签、样本、市场研究；作为可插拔数据源 |
| 自动生产中心 | 多章队列、Plan/Write/Review/Revise/Settle/Commit、停止策略 |
| Context Inspector | 展示 AI 真正看到了什么、为什么召回、为什么被裁剪 |
| 运行中心 | Token、模型、检索、Review、Mutation、Validation、Commit Trace |
| 导入导出 | InkOS / ApiSaverWriter / Markdown / EPUB 等项目迁移与归档 |

不把平台发布绑定到 WebNovel Domain；发布属于独立 adapter，可在最终架构中接入，但不能污染 Canon。

---

# 4. 最终包结构

新增正式 package：

~~~text
packages/webnovel/
  package.json
  tsconfig.json
  src/
    index.ts

    contracts/
      ids.ts
      config.ts
      chapter.ts
      character.ts
      world.ts
      hook.ts
      timeline.ts
      graph.ts
      memory.ts
      context.ts
      audit.ts
      production.ts
      import-export.ts
      events.ts
      migrations.ts

    canon/
      reader.ts
      writer.ts
      mutations.ts
      validation.ts
      history.ts
      evidence.ts

    context/
      profiles.ts
      classifier.ts
      protected-context.ts
      retrieval-query.ts
      retrieval.ts
      graph-expansion.ts
      dedupe.ts
      budget.ts
      compiler.ts
      trace.ts

    characters/
      projector.ts
      history.ts
      relevance.ts

    hooks/
      lifecycle.ts
      aging.ts
      relevance.ts
      projector.ts

    timeline/
      parser.ts
      validator.ts
      projector.ts
      conflicts.ts

    graph/
      projector.ts
      relations.ts
      traversal.ts
      evidence.ts

    memory/
      documents.ts
      projection.ts
      query.ts
      ranking.ts

    audit/
      deterministic/
      semantic/
      evidence-validator.ts
      scoring.ts
      fix-proposal.ts
      service.ts

    writing/
      preflight.ts
      planner.ts
      context-build.ts
      writer.ts
      reviewer.ts
      reviser.ts
      settler.ts
      commit.ts
      workflow.ts

    outline/
      contracts.ts
      planning.ts
      validation.ts
      projection.ts

    style/
      profile.ts
      samples.ts
      binding.ts
      analysis.ts

    analysis/
      book-import.ts
      chapter-analysis.ts
      skill-extraction.ts
      market-research.ts

    production/
      contracts.ts
      queue.ts
      runner.ts
      policies.ts
      checkpoints.ts
      recovery.ts

    projections/
      layout.ts
      rebuild.ts
      hashing.ts
      version.ts

    migrations/
      registry.ts
      v1.ts

    metrics/
      dashboard.ts
      health.ts
      token-usage.ts
      production.ts

    __tests__/
~~~

包名：

~~~json
{
  "name": "@actalk/inkos-webnovel",
  "version": "2.0.0",
  "private": true
}
~~~

本 package 初期保持 private，不是因为 API 是临时的，而是为了避免外部生态锁死内部接口；内部 Contract 仍按稳定终态设计。

---

# 5. Studio 最终目录

~~~text
packages/studio/src/features/webnovel/
  routes.ts
  api/
    client.ts
    contracts.ts
  store/
    navigation.ts
    editor-session.ts
    ui-preferences.ts

  components/
    shell/
      WebNovelShell.tsx
      WebNovelNav.tsx

    dashboard/
      WebNovelDashboard.tsx
      HealthCard.tsx
      ProductionCard.tsx
      HookRiskCard.tsx

    chapter/
      ChapterWorkbench.tsx
      ChapterTree.tsx
      ChapterEditor.tsx
      ChapterToolbar.tsx
      ChapterPlanPanel.tsx
      ChapterReviewPanel.tsx
      WorkspaceDiff.tsx

    context/
      ContextInspector.tsx
      ContextSection.tsx
      RetrievalTrace.tsx
      BudgetChart.tsx

    outline/
      OutlineCenter.tsx
      OutlineTree.tsx
      OutlineEditor.tsx

    character/
      CharacterCenter.tsx
      CharacterState.tsx
      CharacterHistory.tsx
      CharacterRelations.tsx

    world/
      WorldCenter.tsx
      WorldEntityPanel.tsx

    hook/
      HookBoard.tsx
      HookDetail.tsx
      HookTimeline.tsx

    timeline/
      TimelineView.tsx
      TimelineConflictPanel.tsx

    graph/
      KnowledgeGraphView.tsx
      GraphEvidencePanel.tsx

    memory/
      MemoryCenter.tsx
      MemorySearch.tsx
      MemorySourcePanel.tsx

    audit/
      AuditCenter.tsx
      AuditIssueList.tsx
      AuditIssueDetail.tsx
      FixProposalPanel.tsx

    style/
      StyleCenter.tsx
      StyleProfileEditor.tsx

    analysis/
      BookAnalysisCenter.tsx
      MarketResearchCenter.tsx

    production/
      ProductionCenter.tsx
      ProductionQueue.tsx
      ProductionRunDetail.tsx

    runs/
      RunCenter.tsx
      RunTrace.tsx

  pages/
    DashboardPage.tsx
    ChapterPage.tsx
    OutlinePage.tsx
    CharactersPage.tsx
    WorldPage.tsx
    HooksPage.tsx
    TimelinePage.tsx
    GraphPage.tsx
    MemoryPage.tsx
    AuditPage.tsx
    StylePage.tsx
    AnalysisPage.tsx
    ProductionPage.tsx
    RunsPage.tsx
~~~

Studio store 只允许持有：

- 当前选中项。
- 面板开关。
- 编辑器未保存 buffer。
- UI preferences。

禁止持有 Canonical Character State、Canonical Hook State 或 Timeline Truth。

---

# 6. 数据分层：永久规则

## 6.1 Canonical Layer

Canonical 是唯一故事事实。

包含：

- Book metadata。
- 世界规则。
- 总纲 / 卷纲 / 已确认章纲。
- 角色定义。
- 角色当前状态。
- 角色知识状态。
- 已确认人物关系。
- 地点状态。
- 势力状态。
- 物品 / 能力归属。
- 章节正文。
- 章节结构化摘要。
- 时间线事件。
- Hook 生命周期。
- Canon Facts。
- 已确认 Story State。
- Canon Mutation History。

要求：

- Zod 校验。
- Revision。
- Evidence。
- 可回滚。
- Atomic Commit。
- 不依赖 UI。
- 不依赖搜索索引。

## 6.2 Projection Layer

可删除后完全重建：

- memory.db。
- graph projection。
- character current view。
- character history view。
- timeline view。
- dashboard metrics。
- audit index。
- search index。
- relation cache。
- chapter relevance index。

Projection 不能拥有 Canon 中不存在的事实。

## 6.3 Advisory Layer

不自动进入故事事实：

- Audit Issue。
- AI Fix Proposal。
- 下一章建议。
- 市场建议。
- 风格建议。
- 对标建议。
- 商业评分。
- 自动生产建议。

## 6.4 Ephemeral Layer

运行期：

- Retrieval candidates。
- Context Pack。
- Context Trace。
- Draft。
- Plan。
- Review draft。
- Settler raw output。
- Workspace pending mutation。
- Tool intermediate result。

---

# 7. 稳定 ID 体系

为避免后期迁移返工，所有领域对象使用稳定 ID，而不是标题或数组下标。

~~~ts
type BookId = string;
type ChapterId = string;
type CharacterId = string;
type EntityId = string;
type HookId = string;
type TimelineEventId = string;
type CanonFactId = string;
type AuditIssueId = string;
type ProductionJobId = string;
type RunId = string;
~~~

推荐 ID：

~~~text
book_<nanoid>
chapter_<nanoid>
char_<nanoid>
entity_<nanoid>
hook_<nanoid>
timeline_<nanoid>
fact_<nanoid>
audit_<nanoid>
job_<nanoid>
run_<nanoid>
~~~

章节 number 是排序属性，不作为永久 ID。

原因：

- 中途插章。
- 拆章 / 合章。
- 番外。
- 重排卷。
- 导入外部作品。

都不能导致所有引用失效。

---

# 8. Canon Mutation：所有写操作的唯一入口

任何模型或 UI 想改变 Story Truth，都必须形成 Mutation Proposal。

~~~ts
interface CanonMutationProposal {
  mutationId: string;
  bookId: BookId;
  chapterId?: ChapterId;

  domain:
    | "character"
    | "relationship"
    | "timeline"
    | "hook"
    | "item"
    | "world"
    | "outline"
    | "canon-fact";

  targetId: string;

  operation:
    | "create"
    | "update"
    | "resolve"
    | "supersede"
    | "delete";

  before?: unknown;
  after?: unknown;

  evidence: CanonEvidence[];

  source:
    | "chapter-settler"
    | "author"
    | "import"
    | "audit-fix";

  confidence?: number;
}
~~~

Evidence：

~~~ts
interface CanonEvidence {
  sourceType: "chapter" | "author-confirmation" | "import";
  sourceId: string;
  chapterId?: ChapterId;
  excerpt?: string;
  charStart?: number;
  charEnd?: number;
}
~~~

严格规则：

- Chapter Plan 不能成为 Evidence。
- Audit Suggestion 不能成为 Evidence。
- Graph 不能成为 Evidence。
- Memory Search Hit 不能成为 Evidence。
- 当前正文或作者明确确认可以成为 Evidence。
- Imported data 必须标 source=import。
- unknown 不能自动升为 confirmed。

---

# 9. WebNovelProjectConfig：最终配置

~~~ts
interface WebNovelProjectConfig {
  schemaVersion: 1;
  enabled: boolean;

  platform: {
    kind: "fanqie" | "qidian" | "faloo" | "generic";
    genre?: string;
    audience?: string;
  };

  writing: {
    targetDailyWords?: number;
    targetChapterWords: number;
    chapterWordsTolerance: number;
    defaultPointOfView?: string;
  };

  context: {
    profileMode: "auto" | "manual";
    defaultProfile: ChapterContextProfile;
    recentProse: {
      protectedChapterCount: 1 | 2;
      maxTokensPerChapter: number;
      protectLastParagraphTokens: number;
    };
    totalDynamicBudgetRatio: number;
  };

  hooks: {
    defaultOverdueAfterChapters: number;
    mainHookOverdueAfterChapters: number;
  };

  audit: {
    chapterReview: boolean;
    deterministicBeforeWrite: boolean;
    semanticWholeBook: boolean;
    wholeBookIntervalChapters?: number;
  };

  production: {
    maxRevisionRounds: number;
    stopOnHighSeverityIssue: boolean;
    stopOnCanonValidationError: boolean;
    autoCommit: {
      enabled: boolean;
      minReviewScore: number;
    };
  };

  market: {
    providers: string[];
  };
}
~~~

Config 与 Canon 分开：

~~~text
book/config/webnovel.json
~~~

---

# 10. 角色模型：定义、当前状态、历史分开

## 10.1 Character Definition

长期稳定设定：

~~~ts
interface WebNovelCharacterDefinition {
  id: CharacterId;
  name: string;
  aliases: string[];
  role: string;
  immutableTraits?: string[];
  background?: string;
  appearance?: string;
  personality?: string;
}
~~~

## 10.2 Current State

~~~ts
interface WebNovelCharacterState {
  characterId: CharacterId;
  asOfChapterId: ChapterId;

  location?: EntityId;
  realmOrPower?: string;
  physicalState?: string;
  emotionalState?: string;
  publicIdentity?: string;
  privateIdentity?: string;

  possessions: EntityId[];
  abilities: EntityId[];

  knownFactIds: CanonFactId[];
  explicitlyUnknownFactIds: CanonFactId[];

  status: "active" | "missing" | "dead" | "retired" | "unknown";

  lastConfirmedChapter: number;
}
~~~

## 10.3 State History

不单独让模型维护历史表。

历史由 Canon Mutation Log 派生：

~~~text
Character Definition
+ Mutation Log
-> Current State
-> Historical View
~~~

这样避免 currentState 与 stateHistory 两份数据长期漂移。

---

# 11. World / Entity 模型

统一 Entity Registry：

~~~ts
type WebNovelEntityType =
  | "location"
  | "organization"
  | "item"
  | "ability"
  | "realm"
  | "race"
  | "rule"
  | "secret"
  | "event";
~~~

~~~ts
interface WebNovelEntity {
  id: EntityId;
  type: WebNovelEntityType;
  name: string;
  aliases: string[];
  description?: string;
  status?: string;
  attributes: Record<string, string | number | boolean | null>;
  createdAtChapter?: number;
  lastConfirmedChapter?: number;
}
~~~

人物不塞进通用 Entity，人物有专门 schema；Graph 可以将两者统一投影。

---

# 12. Canon Facts 与人物认知

必须显式区分：

- 客观事实。
- 某个角色知道的事实。
- 某个角色明确不知道的事实。

~~~ts
interface CanonFact {
  id: CanonFactId;
  statement: string;
  subjectIds: string[];
  validFromChapter: number;
  validUntilChapter?: number;
  status: "active" | "superseded";
  evidence: CanonEvidence[];
}
~~~

这解决长篇最常见错误之一：

> “作者知道”不等于“角色知道”。

Writer Context 必须从 character knownFactIds 判断角色认知。

---

# 13. Hook：完整生命周期

~~~ts
interface WebNovelHook {
  id: HookId;
  title: string;
  type:
    | "main"
    | "character"
    | "emotion"
    | "world"
    | "mystery"
    | "item"
    | "promise";

  status:
    | "planted"
    | "progressing"
    | "ready"
    | "resolved"
    | "superseded";

  importance: "critical" | "major" | "minor";

  plantedChapter: number;
  lastProgressChapter?: number;

  expectedPayoff: {
    startChapter?: number;
    endChapter?: number;
    description?: string;
  };

  relatedIds: string[];

  seedEvidence: CanonEvidence[];
  progressEvidence: Array<{
    chapter: number;
    evidence: CanonEvidence[];
  }>;
  payoffEvidence?: CanonEvidence[];
}
~~~

“逾期”是计算属性：

~~~text
unresolved
AND
(
  currentChapter > expectedPayoff.endChapter
  OR
  currentChapter - lastProgressChapter >= overdueThreshold
)
~~~

不写入 status，避免后来规则调整时需要数据迁移。

---

# 14. Timeline：终态模型

~~~ts
interface WebNovelTimelineEvent {
  id: TimelineEventId;
  chapterId: ChapterId;
  chapterNumber: number;

  orderKey: string;

  absoluteTime?: {
    calendar?: string;
    value: string;
  };

  relativeTime?: {
    fromEventId?: TimelineEventId;
    elapsed: string;
  };

  locationIds: EntityId[];
  participantIds: CharacterId[];

  event: string;

  certainty:
    | "explicit"
    | "relative-explicit"
    | "derived-order-only"
    | "unknown";

  evidence: CanonEvidence[];
}
~~~

规则：

- 没写日期就不造日期。
- 可以只有 orderKey。
- “三天后”可以保留 relativeTime。
- 只有明确文本证据时才生成 absoluteTime。
- Timeline Conflict Detector 优先用确定信息，不对 unknown 做强冲突判断。

---

# 15. Knowledge Graph：永远是 Projection

Graph Node：

~~~ts
interface WebNovelGraphNode {
  id: string;
  sourceId: CharacterId | EntityId | HookId | TimelineEventId | CanonFactId;
  kind:
    | "character"
    | "location"
    | "organization"
    | "item"
    | "ability"
    | "event"
    | "hook"
    | "fact";
  label: string;
  lastConfirmedChapter?: number;
}
~~~

Graph Edge：

~~~ts
interface WebNovelGraphEdge {
  id: string;
  source: string;
  target: string;
  relation: string;
  weight: number;
  evidenceRefs: string[];
  lastConfirmedChapter?: number;
}
~~~

Graph 来源只能是：

~~~text
Character State
World Entities
Canon Facts
Hook relations
Timeline
Chapter Summary relations
        |
        v
Graph Projector
~~~

手工编辑 Graph 时：

~~~text
UI edit
-> Proposed Canon Mutation
-> Confirmation
-> Canon
-> Rebuild Graph
~~~

Graph 本身从不直接保存“作者编辑后的真相”。

---

# 16. 章节结构化记忆：最终 Schema

ApiSaverWriter 的结构化 chapter memory 很值得吸收，但必须与 InkOS Canon 对齐。

~~~ts
interface WebNovelChapterMemory {
  chapterId: ChapterId;
  chapterNumber: number;
  title: string;

  summary: string;
  keywords: string[];

  events: string[];

  characterStateChanges: Array<{
    characterId: CharacterId;
    summary: string;
    mutationIds: string[];
  }>;

  relationshipChanges: Array<{
    sourceId: CharacterId;
    targetId: CharacterId;
    summary: string;
    mutationIds: string[];
  }>;

  timelineEventIds: TimelineEventId[];

  hookChanges: Array<{
    hookId: HookId;
    action: "plant" | "progress" | "resolve" | "supersede";
  }>;

  canonFactIds: CanonFactId[];

  endingHook?: string;
  mood?: string;
  chapterType?: ChapterContextProfile;
}
~~~

关键：

**Chapter Memory 是对已提交 Canon 的索引化摘要，不是独立事实源。**

---

# 17. Context Profile：终态类型

最终支持：

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

每章允许：

~~~ts
interface ChapterProfileDecision {
  primary: ChapterContextProfile;
  secondary?: ChapterContextProfile;
  confidence: number;
  reason: string;
}
~~~

secondary 不引入另一套 profile；预算按线性混合计算。

---

# 18. Context Section：固定终态枚举

~~~ts
type WebNovelContextSection =
  | "author-instruction"
  | "chapter-outline"
  | "current-canon"
  | "recent-prose"
  | "character-state"
  | "timeline"
  | "hooks"
  | "memory"
  | "graph"
  | "world"
  | "references"
  | "style"
  | "skills"
  | "session";
~~~

未来调权重只改 policy，不改结构。

---

# 19. Context 优先级

## P0 — Hard Protected

无论检索得分如何都优先保留：

- Author Instruction。
- 当前章纲。
- Canon current state 中与本章实体相关的强约束。
- 上一章正文。
- 上一章末尾重点窗口。
- 明确点名的人物 / 地点 / 物品状态。
- 本章必须处理的 critical Hook。
- 已确认时间约束。

## P1 — Soft Protected

- 上上章正文。
- 当前卷目标。
- 主角状态。
- 当前场景参与人物。
- 最近 Timeline。
- major 到期 Hook。

## P2 — Retrieved

- 历史 Chapter Memory。
- Volume Summary。
- Canon Fact。
- Character History。
- Reference Material。
- Graph expansion。

## P3 — Optional

- Skills。
- Market reference。
- 非核心风格样本。
- 辅助资料。

裁剪顺序永远：

~~~text
P3 -> P2 -> P1
~~~

P0 不因普通预算不足被静默裁剪；如果上下文窗口真的无法容纳，必须显式报 ContextOverflow，而不是偷偷删 Canon。

---

# 20. Dynamic Context Budget

下表是默认策略，数值最终通过 benchmark 校准，但 Section 结构永久不变。

| Section | 剧情 | 战斗 | 情感 | 转场 | 揭秘 | 高潮 | 日常 | 群像 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| recent prose | 28 | 30 | 24 | 36 | 20 | 26 | 32 | 20 |
| character | 14 | 20 | 24 | 10 | 12 | 18 | 18 | 24 |
| memory | 24 | 14 | 24 | 16 | 30 | 20 | 18 | 22 |
| hooks | 10 | 8 | 12 | 8 | 18 | 16 | 8 | 10 |
| world/timeline | 12 | 18 | 8 | 16 | 12 | 12 | 10 | 12 |
| skills/reference/style | 12 | 10 | 8 | 14 | 8 | 8 | 14 | 12 |

固定 System Prompt、Author Instruction、硬 Canon 不参与上述百分比分配。

---

# 21. Retrieval：最终流程

~~~text
Author Instruction
+ Chapter Outline
+ Current Canon
+ Profile
        |
        v
Query Builder
        |
        v
Lexical Retrieval: SQLite FTS5/BM25
        |
        v
Candidate Pool
        |
        v
Semantic Selector
        |
        v
Optional Graph Expansion
        |
        v
Canonical Deduplication
        |
        v
Temporal / State Validity Filter
        |
        v
Context Budget Allocator
        |
        v
Protected + Dynamic Pack
        |
        v
Context Trace
~~~

重要规则：

1. 上一章正文不通过 BM25 竞争。
2. Canon current state 不通过 BM25 竞争。
3. 搜到旧人物状态时，必须标 historical，不得覆盖 current state。
4. resolved Hook 默认不参与写作检索。
5. Audit / Memory Center 可以查询 resolved Hook。
6. Reference Material 不能覆盖 Canon。
7. semantic selector 只能从候选中选，不能凭空生成 ID。

---

# 22. Local Search 的最终演进方向

保留：

- SQLite。
- FTS5。
- BM25。
- source file authority。
- rebuildable index。

允许扩展：

~~~ts
interface RetrievalHit {
  id: string;
  kind: string;
  source: string;
  score: number;

  lexicalScore?: number;
  semanticScore?: number;
  recencyScore?: number;
  importanceScore?: number;

  selectedBy: Array<
    | "protected"
    | "bm25"
    | "semantic"
    | "graph"
    | "rule"
  >;
}
~~~

为未来 embedding / vector search 预留：

~~~ts
interface RetrievalProvider {
  search(request: RetrievalRequest): Promise<RetrievalHit[]>;
}
~~~

默认仍使用 LocalSearchIndex。

未来加 VectorProvider 时只替换 provider，不改 Context Compiler。

这才是真正“不返工”。

---

# 23. Context Trace：正式产品能力

~~~ts
interface WebNovelContextTrace {
  schemaVersion: 1;
  runId: RunId;
  bookId: BookId;
  chapterId: ChapterId;

  profile: ChapterProfileDecision;

  model: {
    provider: string;
    model: string;
    contextWindowTokens?: number;
  };

  budget: {
    totalAvailable: number;
    fixed: number;
    dynamic: number;
    bySection: Record<WebNovelContextSection, number>;
  };

  retrieval: {
    query: string;
    candidates: ContextTraceItem[];
    semanticSelectedIds: string[];
  };

  protected: ContextTraceItem[];
  selected: ContextTraceItem[];
  graphExpanded: ContextTraceItem[];
  omitted: ContextTraceItem[];

  estimatedInputTokens: number;
}
~~~

ContextTraceItem：

~~~ts
interface ContextTraceItem {
  id: string;
  section: WebNovelContextSection;
  source: string;
  sourceId?: string;
  chapterNumber?: number;
  reason: string;
  score?: number;
  estimatedTokens: number;
  historical?: boolean;
}
~~~

Studio 必须展示：

- 看到了什么。
- 为什么看到。
- 为什么没看到。
- 占多少 token。
- 来源在哪里。
- 是否 historical。
- 是否 protected。

---

# 24. Writer 不允许自主形成第二套检索逻辑

章节 Writer 的输入必须是 Compiler 产出的 Context Pack。

Writer 可以调用 Harness Tool，但 Tool 调用必须：

- 记录到 Run Trace。
- 产生 Observation。
- 不能静默覆盖 Canon。
- 新取得的资料需要进入当前 run 的 supplemental context。

禁止在 Writer Prompt 里写“如果需要你自己回忆之前章节”，因为那会破坏可解释性。

---

# 25. 最终章节流水线

~~~mermaid
flowchart LR
  A[Preflight] --> B[Profile]
  B --> C[Context Build]
  C --> D[Plan]
  D --> E[Draft]
  E --> F[Review]
  F --> G{Pass}
  G -- no --> H[Revise]
  H --> F
  G -- yes --> I[Settle]
  I --> J[Mutation Validation]
  J --> K[Workspace Validation]
  K --> L[Commit]
  L --> M[Projection Rebuild]
  M --> N[Post-Commit Audit]
~~~

所有入口，包括 Studio / CLI / TUI / automation，都调用同一 workflow。

---

# 26. Preflight：完整检查

Preflight 输出：

~~~ts
interface WebNovelPreflightResult {
  blockers: PreflightIssue[];
  warnings: PreflightIssue[];
  profileHint?: ChapterProfileDecision;
  activeHookIds: HookId[];
  requiredCharacterIds: CharacterId[];
  staleWorkspace?: boolean;
}
~~~

Blocker：

- Canon Schema 无法加载。
- 章节 ID / number 冲突。
- 上一个 Workspace 未决且无法恢复。
- 当前章纲声明 strict 但不存在。
- 关键 Projection version 与 Canon schema 不兼容。
- 迁移未完成。

Warning：

- 上一章正文缺失。
- Timeline 信息不足。
- Hook 大量逾期。
- 角色状态长时间未确认。
- Whole-book Audit 存在 high issue。

---

# 27. Planner：最终 Contract

~~~ts
interface WebNovelChapterPlan {
  chapterId: ChapterId;
  profile: ChapterProfileDecision;

  continuityAnchors: string[];
  activeCharacters: CharacterId[];
  activeLocations: EntityId[];

  characterGoals: Array<{
    characterId: CharacterId;
    goal: string;
  }>;

  eventChain: string[];
  conflictEscalation: string[];
  newInformation: string[];

  hookActions: Array<{
    hookId: HookId;
    intent: "mention" | "progress" | "payoff";
  }>;

  timelineIntent?: string;
  endingHook: string;

  mustKeep: string[];
  mustAvoid: string[];
}
~~~

Planner 只是“写作意图”，不修改 Canon。

---

# 28. Draft：终态规则

Writer 输出：

~~~ts
interface WebNovelDraftResult {
  content: string;
  selfSummary?: string;
}
~~~

selfSummary 只是辅助，不直接写 chapter memory。

真正 Chapter Memory 由 Commit 之后的 Canon / Settler 结果生成。

---

# 29. Review：单章审稿

Review Schema：

~~~ts
interface WebNovelChapterReview {
  score: number;
  consistent: boolean;

  issues: Array<{
    severity: "high" | "medium" | "low";
    category:
      | "outline"
      | "character"
      | "knowledge"
      | "timeline"
      | "world-rule"
      | "ownership"
      | "hook"
      | "causality"
      | "duplication"
      | "language";
    evidence: string;
    suggestion: string;
  }>;

  suggestions: string[];
}
~~~

单章 Review 不替代 Whole-book Audit。

---

# 30. Revision

Revision 必须带：

- 原正文。
- Review Issue。
- Canon constraints。
- 不可修改项。

Revision 之后重新 Review。

maxRevisionRounds 来自 config。

超过上限：

- 进入 needs-author-review。
- 不静默继续生产。

---

# 31. Settler：事实抽取器

Settler 只从最终正文提取：

- Chapter Summary。
- Character Mutation。
- Relationship Mutation。
- Timeline Mutation。
- Hook Mutation。
- Item / Ability Mutation。
- Canon Fact。
- Entity mention / evidence。

Settler 输出：

~~~ts
interface WebNovelSettleResult {
  chapterMemoryDraft: WebNovelChapterMemory;
  mutations: CanonMutationProposal[];
  graphEvidence: GraphEvidence[];
}
~~~

注意：

- GraphEvidence 不是 Graph Edge 本身。
- Graph Projector 以后用 evidence + Canon 重建关系。

---

# 32. Workspace：最终事务边界

单章 Workspace 必须包含：

~~~text
workspace/
  chapter/
    draft.md
    final.md
  plan.json
  review.json
  settle.json
  mutations.json
  context-trace.json
  validation.json
  commit-manifest.json
~~~

Commit Manifest：

~~~ts
interface WebNovelCommitManifest {
  runId: RunId;
  bookId: BookId;
  chapterId: ChapterId;

  files: Array<{
    path: string;
    beforeHash?: string;
    afterHash: string;
  }>;

  mutationIds: string[];
  projectionVersion: number;

  validatedAt: string;
}
~~~

Commit 成功条件：

- 正文可解析。
- 所有 Canon Mutation 合法。
- Evidence 合法。
- 无 ID 冲突。
- Hook lifecycle 合法。
- Timeline 硬冲突通过。
- Character current state 可计算。
- Manifest hash 完整。

失败：

**任何 Canon 都不推进。**

---

# 33. Projection 重建

固定目录：

~~~text
story/projections/webnovel/
  version.json
  graph.json
  timeline.json
  characters.json
  hooks.json
  dashboard.json
  audit-index.json
~~~

Memory DB 仍按 InkOS 现有 story/memory.db 策略，可使用 scope 区分 WebNovel document kind。

Projection Version：

~~~ts
interface ProjectionVersion {
  schemaVersion: 1;
  canonRevision: string;
  projectionRevision: string;
  generatedAt: string;
}
~~~

如果 Projection revision 落后：

- Studio 显示 rebuilding。
- 自动重建。
- 不把旧 Projection 当 Canon。

---

# 34. Memory Documents：统一索引类型

新增 document kind：

~~~ts
type WebNovelMemoryKind =
  | "chapter-summary"
  | "volume-summary"
  | "hook"
  | "timeline-event"
  | "character-state-history"
  | "canon-fact"
  | "world-entity"
  | "reference";
~~~

document metadata 必须有：

- source ID。
- source revision。
- chapter number（如适用）。
- validity。
- historical flag。

---

# 35. Knowledge Graph 检索

Graph 检索不是单独的事实召回器，而是用于：

- 从显式命中实体扩展相关实体。
- 找 1-hop 强关系。
- 找与 Hook 关联实体。
- 找物品归属。
- 找角色关系。

默认：

- seed <= 10。
- expanded nodes <= 20。
- edge <= 32。
- 1-hop。
- strong relation 优先。

未来需要 2-hop 时通过 policy 配置，不改 Graph Schema。

---

# 36. Whole-book Audit：终态设计

完整 Audit 分三层。

## 36.1 Deterministic Audit

完全不调用模型：

- duplicate ID。
- alias collision。
- Chapter Memory 缺失。
- resolved Hook 仍 active。
- critical Hook 逾期。
- item multiple owner。
- dead character active。
- Timeline 明确硬冲突。
- projection stale。
- chapter number duplication。
- broken Canon reference。
- orphan relation。
- mutation before/after 不一致。

## 36.2 Semantic Aggregate Audit

输入：

- 世界规则。
- 总纲 / 卷纲。
- Character Definition。
- Current State。
- Character History digest。
- Chapter Memory。
- Timeline。
- Hook。
- Canon Facts。

检查：

- 人物前后矛盾。
- 认知穿帮。
- 力量体系越界。
- 称谓 / 专名混乱。
- 因果断裂。
- 伏笔遗忘。
- 回收与 seed 不一致。
- 重复剧情。
- 主线长期停滞。
- 角色长期无意义出场。

## 36.3 Evidence Drill-down

如果 aggregate audit 怀疑冲突但证据不足：

~~~text
Audit candidate
-> retrieve exact chapter excerpts
-> evidence verification
-> create issue or drop
~~~

不是直接把猜测变 Issue。

---

# 37. Audit Issue：正式 Schema

~~~ts
interface WebNovelAuditIssue {
  id: AuditIssueId;
  bookId: BookId;

  severity: "high" | "medium" | "low";

  category:
    | "character"
    | "knowledge"
    | "timeline"
    | "world-rule"
    | "naming"
    | "ownership"
    | "hook"
    | "causality"
    | "duplication"
    | "continuity"
    | "projection";

  summary: string;

  entityIds: string[];
  chapterIds: ChapterId[];

  evidence: CanonEvidence[];

  suggestion?: string;

  status:
    | "open"
    | "ignored"
    | "fixed"
    | "invalid";

  createdAt: string;
  resolvedAt?: string;
}
~~~

High / Medium Issue 必须有 >= 2 个可定位 evidence，除非问题本身是单点 schema violation。

---

# 38. Audit Fix：绝不直接改 Canon

~~~text
Audit Issue
-> Fix Proposal
-> Author Preview
-> Chapter / Canon Workspace
-> Validation
-> Commit
-> Re-run relevant audit
-> Mark fixed
~~~

---

# 39. 文风中心：终态

文风不能只是 Prompt 文本。

~~~ts
interface WebNovelStyleProfile {
  id: string;
  name: string;
  description: string;

  constraints: {
    pov?: string;
    tense?: string;
    sentenceRhythm?: string;
    dialogueDensity?: string;
    expositionDensity?: string;
    forbiddenPatterns?: string[];
    preferredPatterns?: string[];
  };

  sampleRefs: Array<{
    source: string;
    charStart?: number;
    charEnd?: number;
  }>;

  skillRefs: string[];
}
~~~

Context 中 Style 独立 section，不混入 Canon Facts。

---

# 40. 拆书中心

参考 ApiSaverWriter 的拆书理念，但实现为独立分析领域：

输入：

- 用户拥有或合法导入的文本。
- Markdown / TXT / EPUB。

产出：

- 章节结构。
- Opening 类型。
- Conflict。
- Pacing。
- Cliffhanger。
- Dialogue ratio。
- Scene density。
- Character introduction。
- Hook patterns。
- Style features。
- 可复用 Skill Proposal。

拆书结果属于 Advisory / Reference，不进入当前小说 Canon。

---

# 41. 对标 / 市场中心

市场数据作为 Adapter：

~~~ts
interface WebNovelMarketProvider {
  id: string;
  getRankings(request: RankingRequest): Promise<RankingResult>;
  searchBooks?(request: BookSearchRequest): Promise<BookSearchResult>;
}
~~~

这样未来：

- 番茄。
- 起点。
- 飞卢。
- 用户自己的数据源。

都不需要改变 WebNovel Domain。

市场结果不进入 Canon。

---

# 42. Production Center：最终设计，不后补

自动生产能力从一开始就进入 Domain Contract，即使 UI 最后实现。

~~~ts
interface WebNovelProductionJob {
  id: ProductionJobId;
  bookId: BookId;

  range: {
    startChapter: number;
    chapterCount: number;
  };

  writing: {
    targetWords: number;
    tolerance: number;
  };

  policy: {
    maxRevisionRounds: number;
    stopOnHighSeverityIssue: boolean;
    stopOnCanonValidationError: boolean;
    stopOnContextOverflow: boolean;
    stopOnProviderErrorCount: number;

    autoCommit: {
      enabled: boolean;
      minReviewScore: number;
      requireNoHighIssue: boolean;
    };
  };

  budget?: {
    maxInputTokens?: number;
    maxOutputTokens?: number;
    maxCost?: number;
  };

  status:
    | "queued"
    | "running"
    | "paused"
    | "blocked"
    | "completed"
    | "cancelled"
    | "failed";
}
~~~

即使第一批 UI 不暴露自动生产，也不能以后再重新设计章节 workflow。

---

# 43. Production Runner

每章统一走：

~~~text
Preflight
-> Context
-> Plan
-> Draft
-> Review
-> Revise
-> Review
-> Settle
-> Mutation Validation
-> Workspace Validation
-> Commit
-> Projection Rebuild
-> Lightweight Audit
-> Check Stop Policy
-> Next Chapter
~~~

Production Runner 只是 orchestration，不能拥有 Writer 专用实现。

---

# 44. Checkpoint / Recovery

Job 每个关键阶段存 checkpoint：

~~~text
job_<id>/
  job.json
  chapter_<id>/
    context-built
    plan-ready
    draft-ready
    review-ready
    settled
    committed
~~~

恢复时根据 checkpoint 继续，不重新偷偷生成已完成步骤。

---

# 45. Studio 总览：作者驾驶舱

首页最终展示：

- 作品名 / 状态。
- 总字数。
- 总章节。
- 当前卷。
- 今日目标与完成量。
- 最近 7 / 30 天产量。
- 当前主线。
- 当前 Profile 分布。
- 下一章。
- Critical / Major Hook。
- Audit High / Medium。
- Character Risk。
- Timeline Risk。
- Production Job。
- 最近 Run。
- Token / Cost。
- Projection 状态。

所有风险卡可跳到证据。

---

# 46. 章节工作台：最终三栏

~~~text
┌────────────────┬──────────────────────────────────┬────────────────────┐
│ 卷 / 章节树      │ 正文编辑器                       │ AI Copilot          │
│                │                                  │                    │
│ 卷一            │ 第428章                          │ 本章计划            │
│ 426            │                                  │ Context Inspector   │
│ 427            │ 正文                             │ 人物状态            │
│ 428 <-         │                                  │ 伏笔                │
│ 429            │                                  │ 时间线              │
│                │                                  │ Review              │
│                │                                  │ Workspace Diff      │
└────────────────┴──────────────────────────────────┴────────────────────┘
~~~

右侧不是聊天框替代品，而是结构化工作区。

聊天仍可存在，但不是主交互模型。

---

# 47. Context Inspector：完整交互

分类：

- Protected。
- Retrieved。
- Graph Expanded。
- References。
- Skills。
- Omitted。

每项：

- 标题。
- kind。
- 来源。
- 章节。
- selection reason。
- token。
- score。
- historical。
- 点击查看原文。

支持：

- “下一次强制包含”。
- “下一次排除”。

这两个操作只改变 Run Override，不改变 Canon。

---

# 48. 人物中心

人物页面：

- Definition。
- Current State。
- Known Facts。
- Unknown Facts。
- Possessions。
- Abilities。
- Current Location。
- Relations。
- State History。
- Appearance History。
- Hook。
- Timeline。
- Audit Issue。

作者手工编辑 Current State：

~~~text
Edit
-> Proposed Mutation
-> Diff
-> Confirm
-> Canon Commit
-> Projection Rebuild
~~~

---

# 49. 世界观中心

管理：

- 地点。
- 势力。
- 规则。
- 等级。
- 能力。
- 物品。
- 秘密。
- 禁则。

世界规则分：

~~~ts
type WorldRuleStrength =
  | "hard"
  | "soft"
  | "rumor";
~~~

只有 hard rule 作为 Writer 强约束。

soft / rumor 需要在 Context 中明确标识，避免模型把传闻当真。

---

# 50. 伏笔中心

视图：

~~~text
全部
关键
主线
人物
情感
世界
谜团
物品
承诺

新埋设
推进中
待回收
逾期
已回收
已废弃
~~~

每个 Hook：

- Seed evidence。
- Progress timeline。
- Expected payoff window。
- Last progress。
- Related entities。
- Retrieval hits。
- Payoff evidence。
- Audit warnings。

---

# 51. Timeline UI

视图：

1. Story Time。
2. Chapter Order。
3. Character Track。
4. Location Track。

Timeline 只能展示已有 certainty。

unknown 必须显式显示“未明确”。

---

# 52. Graph UI

使用 @xyflow/react。

支持：

- Filter by entity type。
- Filter by chapter range。
- Only current chapter context。
- Edge evidence。
- Node canon detail。
- Hook overlay。
- Timeline overlay。

编辑行为转 Mutation Proposal。

---

# 53. Memory Center

统一搜索所有可检索知识。

过滤：

- Chapter。
- Character。
- Hook。
- Timeline。
- Canon Fact。
- World。
- Reference。
- Historical / Current。

结果显示：

- source。
- chapter。
- rank。
- selectedBy。
- Canon current / historical badge。

---

# 54. Audit Center

Dashboard：

- Overall Health。
- Character。
- Knowledge。
- Timeline。
- World Rules。
- Hook。
- Causality。
- Repetition。
- Projection Health。

Issue 流程：

~~~text
Open
-> Inspect
-> Ignore / Invalid / Fix Proposal
-> Workspace
-> Commit
-> Re-audit
-> Fixed
~~~

---

# 55. Run Center

每次 Agent 运行有永久可查记录：

- Run ID。
- Trigger。
- User instruction。
- Profile。
- Model。
- Context Trace。
- Tool calls。
- Plan。
- Draft。
- Review。
- Revision。
- Settler。
- Mutations。
- Validation。
- Commit。
- Token usage。
- Cost（如 Provider 提供）。

---

# 56. Studio Server API：最终版本

统一前缀：

~~~text
/api/v1/webnovel
~~~

只读：

~~~text
GET /books/:bookId/dashboard
GET /books/:bookId/config
GET /books/:bookId/outline
GET /books/:bookId/chapters
GET /books/:bookId/chapters/:chapterId
GET /books/:bookId/chapters/:chapterId/context
GET /books/:bookId/characters
GET /books/:bookId/characters/:characterId
GET /books/:bookId/world/entities
GET /books/:bookId/hooks
GET /books/:bookId/timeline
GET /books/:bookId/graph
GET /books/:bookId/memory/search
GET /books/:bookId/audits
GET /books/:bookId/audits/:issueId
GET /books/:bookId/runs
GET /books/:bookId/runs/:runId
GET /books/:bookId/production
~~~

操作：

~~~text
POST /books/:bookId/chapters/:chapterId/plan
POST /books/:bookId/chapters/:chapterId/write
POST /books/:bookId/chapters/:chapterId/review
POST /books/:bookId/chapters/:chapterId/revise
POST /books/:bookId/chapters/:chapterId/commit

POST /books/:bookId/audits/run
POST /books/:bookId/audits/:issueId/fix-proposal
POST /books/:bookId/audits/:issueId/apply

POST /books/:bookId/projections/rebuild

POST /books/:bookId/production/jobs
POST /books/:bookId/production/jobs/:jobId/pause
POST /books/:bookId/production/jobs/:jobId/resume
POST /books/:bookId/production/jobs/:jobId/cancel

POST /books/:bookId/import
POST /books/:bookId/export
~~~

所有写路由必须落到 Harness / Domain Action，不允许 route 自己改文件。

---

# 57. Harness Action：最终命名

只读：

~~~text
webnovel.dashboard.get
webnovel.config.get
webnovel.context.preview
webnovel.memory.search
webnovel.character.list
webnovel.character.get
webnovel.character.history
webnovel.world.list
webnovel.hook.list
webnovel.timeline.list
webnovel.graph.get
webnovel.audit.list
webnovel.run.get
webnovel.production.list
~~~

生成 / 分析：

~~~text
webnovel.chapter.plan
webnovel.chapter.write
webnovel.chapter.review
webnovel.chapter.revise
webnovel.audit.run
webnovel.audit.fix-propose
webnovel.analysis.book
webnovel.analysis.style
webnovel.analysis.market
~~~

变更：

~~~text
webnovel.chapter.commit
webnovel.canon.mutate
webnovel.audit.fix-apply
webnovel.projection.rebuild
webnovel.import
~~~

生产：

~~~text
webnovel.production.start
webnovel.production.pause
webnovel.production.resume
webnovel.production.cancel
~~~

---

# 58. CLI：最终入口

~~~text
inkos webnovel enable
inkos webnovel status
inkos webnovel dashboard
inkos webnovel context <chapter>
inkos webnovel character list
inkos webnovel character show <id>
inkos webnovel hook list
inkos webnovel timeline
inkos webnovel graph
inkos webnovel memory search <query>
inkos webnovel audit
inkos webnovel audit show <id>
inkos webnovel rebuild
inkos webnovel produce --from <n> --count <n>
inkos webnovel production status
inkos webnovel import <source>
inkos webnovel export <format>
~~~

CLI 与 Studio 使用同一 Domain，不复制业务逻辑。

---

# 59. 迁移系统：从第一天就版本化

~~~ts
interface WebNovelMigration {
  fromVersion: number;
  toVersion: number;
  migrate(ctx: MigrationContext): Promise<void>;
  validate(ctx: MigrationContext): Promise<void>;
}
~~~

~~~text
book/config/webnovel.json
schemaVersion: 1

story/projections/webnovel/version.json
schemaVersion: 1
~~~

Migration Registry：

~~~text
0 -> 1
1 -> 2
...
~~~

以后 schema 演进必须 migration，不允许“启动时猜字段”。

---

# 60. 现有 InkOS 项目启用流程

~~~text
inkos webnovel enable
~~~

执行：

1. 验证项目。
2. 创建 config。
3. 扫描现有 chapters。
4. 读取现有 structured state。
5. 生成稳定 ID mapping。
6. 生成 Canon-compatible enrichment。
7. 生成 projections。
8. 建 memory index。
9. 运行 deterministic audit。
10. 输出 migration report。

不自动：

- 猜绝对时间。
- 猜角色关系。
- 猜未知人物身份。
- 改原正文。

---

# 61. ApiSaverWriter 导入：正式 Adapter

~~~text
packages/webnovel/src/import-export/apisaverwriter/
  reader.ts
  mapping.ts
  validation.ts
  report.ts
~~~

迁移：

- 项目元数据。
- 世界观。
- 总纲 / 章纲。
- 人物卡。
- 正文。
- Chapter Memory。
- Hook。
- 文风。
- 可用的关系数据。

不迁移：

- API Key。
- Cookie。
- Provider secret。
- Runtime cache。
- 账号信息。
- 云端计费信息。

导入后：

~~~text
External data
-> staging
-> schema validation
-> stable ID mapping
-> Canon import transaction
-> projection rebuild
-> full audit
-> import report
~~~

---

# 62. Provider / Model Router

WebNovel 永远不实现第二套 Provider。

直接复用 InkOS：

- OpenAI compatible。
- Responses。
- Anthropic。
- Gemini。
- Kimi。
- MiniMax。
- DeepSeek。
- OpenRouter。
- Ollama。
- 其他现有 provider。

WebNovel 只声明角色：

~~~ts
type WebNovelModelRole =
  | "planner"
  | "writer"
  | "reviewer"
  | "settler"
  | "auditor"
  | "semantic-selector"
  | "analyzer";
~~~

然后交给现有 model routing 映射。

---

# 63. 多模型路由

允许：

~~~text
planner -> fast model
writer -> strong prose model
reviewer -> reasoning model
settler -> structured-output model
semantic-selector -> cheap model
auditor -> strong reasoning model
~~~

Role 是稳定抽象，以后换模型不用改 workflow。

---

# 64. 错误模型

定义领域错误：

~~~ts
type WebNovelErrorCode =
  | "CANON_INVALID"
  | "CANON_CONFLICT"
  | "WORKSPACE_STALE"
  | "CONTEXT_OVERFLOW"
  | "RETRIEVAL_FAILED"
  | "PROFILE_INVALID"
  | "MUTATION_INVALID"
  | "EVIDENCE_INVALID"
  | "TIMELINE_CONFLICT"
  | "HOOK_TRANSITION_INVALID"
  | "PROJECTION_STALE"
  | "AUDIT_EVIDENCE_INSUFFICIENT"
  | "PRODUCTION_BLOCKED";
~~~

不要 UI 解析英文 error message。

---

# 65. Event Contract

新增领域事件，供 Studio / logging / production 使用：

~~~ts
type WebNovelEvent =
  | { type: "context.compiled"; runId: RunId }
  | { type: "chapter.planned"; runId: RunId }
  | { type: "chapter.drafted"; runId: RunId }
  | { type: "chapter.reviewed"; runId: RunId }
  | { type: "chapter.revised"; runId: RunId }
  | { type: "chapter.settled"; runId: RunId }
  | { type: "canon.validated"; runId: RunId }
  | { type: "chapter.committed"; runId: RunId }
  | { type: "projection.rebuilt"; bookId: BookId }
  | { type: "audit.completed"; bookId: BookId }
  | { type: "production.blocked"; jobId: ProductionJobId };
~~~

Event 是观察信号，不是状态源。

---

# 66. Feature Capability

WebNovel 能力注册到现有 Capability Registry。

建议：

~~~text
webnovel.read
webnovel.write
webnovel.audit
webnovel.production
webnovel.import
webnovel.market
~~~

便于以后：

- CLI。
- Studio。
- OpenClaw。
- Agent。
- 组织权限。

统一复用。

---

# 67. 性能设计

日常写章不能扫描全书正文。

固定策略：

- Recent Prose：最近 1～2 章。
- Current Canon：结构化读取。
- Memory：FTS5/BM25。
- Graph：projection。
- Timeline：projection。
- Hook：structured state。

Whole-book Audit：

- 聚合结构化数据。
- 按需 evidence drill-down。

---

# 68. Cache 设计

允许缓存：

- projection hash。
- stable context prefix。
- graph。
- dashboard metrics。
- retrieval query result。
- reference parsing result。

缓存 Key：

~~~text
bookId
+ canonRevision
+ projectionRevision
+ chapterId
+ profile
+ instructionHash
~~~

不能把模型建议缓存成 Story Truth。

---

# 69. 1000 章规模要求

终态必须按以下量级设计：

- 1000～3000 章。
- 200～500 个角色。
- 500～2000 个世界实体。
- 500～3000 个 Hook / Story Promise。
- 数万 Canon Mutation。
- 数千 Timeline Event。
- 数十万到数百万字 Reference。

任何设计如果要求每次写章 JSON.parse 整部小说全部历史，应否决。

---

# 70. Benchmark：正式进入 CI / Manual Gate

固定 benchmark 项目：

## Small

- 20 章。
- 功能正确性。

## Medium

- 200 章。
- CI regression。

## Large

- 1000 章。
- nightly / manual release gate。

测量：

- Context build latency。
- Retrieval recall@K。
- Retrieval precision sample。
- Protected context presence。
- Canon mutation failure。
- Projection rebuild time。
- Whole-book audit time。
- Peak memory。
- Token input。
- Hook recall。
- Timeline conflict recall。
- Recovery correctness。

---

# 71. Retrieval Benchmark

固定问题：

~~~text
第118章埋“凤凰玉佩”
第203章推进
第377章推进
第430章目标要求揭示北域身世
~~~

必须：

- 召回相关三条中的关键证据。
- 不让无关战争支线占满预算。
- Context Trace 显示 reason。
- Character Current State 保持最新。

---

# 72. Consistency Benchmark

固定场景：

- 第 3 章左臂受伤。
- 第 7 章恢复。
- 第 17 章左手持剑。

系统不能误报第 17 章冲突，因为 Canon current state 已恢复。

这用于验证“旧 memory 不覆盖 current state”。

---

# 73. Character Knowledge Benchmark

设定：

- 作者在第 10 章知道秘密。
- 主角到第 50 章才得知。

第 30 章生成时：

- Canon Fact 可以存在。
- 主角 knownFactIds 不能包含。
- Writer 不得让主角说出秘密。

---

# 74. Hook Benchmark

- 第 10 章埋。
- 第 30 章推进。
- expected payoff 60～70。
- 第 75 章未回收。

必须：

- Hook Center 标 overdue。
- Planner 可收到。
- Review 可提醒。
- 不自动标 resolved。

---

# 75. Timeline Benchmark

- 第 10 章明确 3 月 1 日。
- 第 11 章“三天后”。
- 第 12 章明确 3 月 3 日。

必须发现 hard conflict。

如果第 12 章只写“清晨”，不得编造日期。

---

# 76. Workspace / Rollback Benchmark

模拟：

1. Draft 成功。
2. Settler 生成 character mutation。
3. Hook mutation 非法。
4. Validation fail。

必须：

- 正式正文不提交。
- character state 不推进。
- Hook 不推进。
- Workspace 保留。
- 可修复重试。

---

# 77. Test 层次

### Unit

- Schema。
- Profile。
- Budget。
- Dedup。
- Hook lifecycle。
- Timeline。
- Graph projector。
- Mutation validation。
- Audit evidence。

### Integration

- Context build。
- Retrieval。
- Chapter workflow。
- Commit。
- Rebuild。
- Import。
- Production checkpoint。

### Flow

- 从开书到写章。
- 从 Audit 到 Fix。
- 从旧项目到 WebNovel。
- Production pause/resume。

### Studio E2E

Playwright 覆盖所有主工作台。

---

# 78. Observability

每次 run 永久记录：

~~~text
runId
trigger
user instruction
profile
model roles
context trace
tool calls
plan
draft revision hashes
review
settle
mutations
validation
commit
projection rebuild
token usage
cost
errors
~~~

用户可以从错误章节回溯：

> 到底是没检索到、被裁了、模型没遵守、Settler 抽错，还是 Canon 本身有问题。

---

# 79. 安全与隐私

- Secrets 不进入 Context Trace。
- Log 自动 redact。
- Import 忽略 key / token / cookie。
- Projection 不存 provider secret。
- Audit export 默认不导出服务配置。
- Local-first。
- 网络模型只收到当前 task 所需 context。

---

# 80. 许可证策略

ApiSaverWriter 用于参考。

实现策略：

1. 优先独立重写行为。
2. 不直接复制整文件。
3. 如果确实改编具体实现：
   - 标记源仓库。
   - 标记源 commit。
   - 保留要求的版权信息。
   - 进行 AGPL 兼容性核查。
4. 当前 InkOS 继续 AGPL-3.0-only。

---

# 81. 具体修改现有 InkOS 文件

重点改：

~~~text
packages/core/src/harness/context-compiler.ts
packages/core/src/harness/agent-context.ts
packages/core/src/harness/contracts.ts
packages/core/src/harness/capability-registry.ts
packages/core/src/harness/action-observation.ts
packages/core/src/harness/artifact-validation.ts
packages/core/src/utils/memory-retrieval.ts
packages/core/src/retrieval/local-search.ts
packages/core/src/models/input-governance.ts
packages/core/src/index.ts
~~~

Studio：

~~~text
packages/studio/src/App.tsx
packages/studio/src/components/Sidebar.tsx
packages/studio/src/api/**
packages/studio/src/features/webnovel/**
~~~

CLI：

~~~text
packages/cli/src/**
~~~

新增：

~~~text
packages/webnovel/**
~~~

---

# 82. Context Compiler 接口

建议通过 Adapter 方式接入，不把网文逻辑硬编码到通用 Compiler。

~~~ts
interface DomainContextContributor {
  id: string;

  supports(input: ContextCompileInput): boolean;

  collectProtected(
    input: ContextCompileInput
  ): Promise<ContextContribution[]>;

  collectDynamic(
    input: ContextCompileInput
  ): Promise<ContextContribution[]>;

  finalizeTrace?(
    trace: InputGovernanceTrace
  ): Promise<void>;
}
~~~

WebNovel 实现：

~~~text
WebNovelContextContributor
~~~

这样未来其它媒介仍可有自己的策略。

---

# 83. Core 与 WebNovel 的依赖方向

为了避免循环依赖：

~~~text
core
  ^
  |
webnovel
  ^
  |
studio / cli
~~~

但如果 Core Harness 需要加载 Domain Contributor：

使用 interface / registration，不允许：

~~~text
core -> webnovel -> core
~~~

推荐：

~~~text
core exposes DomainContextContributor
webnovel implements it
studio/cli registers it during bootstrap
~~~

这条非常重要，否则后期 package 会出现循环依赖返工。

---

# 84. Domain Bootstrapping

~~~ts
registerCapability(webNovelCapabilities);
registerDomainContextContributor(webNovelContextContributor);
registerArtifactValidators(webNovelValidators);
registerProjectionBuilder(webNovelProjectionBuilder);
~~~

WebNovel 是插件式领域模块，但属于官方内置 package。

---

# 85. Storage Port

WebNovel 不直接散落 fs 调用。

~~~ts
interface WebNovelStorage {
  readConfig(bookId: BookId): Promise<WebNovelProjectConfig>;
  readCanon<T>(ref: CanonRef<T>): Promise<T>;
  writeWorkspaceFile(...): Promise<void>;
  readProjection<T>(...): Promise<T>;
  writeProjection<T>(...): Promise<void>;
}
~~~

默认 adapter 使用 InkOS 本地项目目录。

未来如果 Studio Desktop / Cloud 改存储方式，Domain 不重写。

---

# 86. Market Provider Port

~~~ts
interface MarketProvider {
  searchRankings(request: RankingRequest): Promise<RankingResult[]>;
}
~~~

平台爬虫不放进核心 Domain。

---

# 87. Import Adapter Port

~~~ts
interface ProjectImportAdapter {
  id: string;
  inspect(source: string): Promise<ImportInspection>;
  import(source: string, staging: ImportStaging): Promise<ImportResult>;
}
~~~

Adapter：

- InkOS Legacy。
- ApiSaverWriter。
- Markdown。
- EPUB。

---

# 88. Export Adapter

~~~ts
interface ProjectExportAdapter {
  id: string;
  export(project: WebNovelExportSource): Promise<ExportResult>;
}
~~~

支持：

- Markdown。
- TXT。
- EPUB。
- JSON archive。

---

# 89. UI 不直接依赖文件结构

Studio 只调用：

~~~text
/api/v1/webnovel
~~~

禁止组件自己知道：

~~~text
story/state/hooks.json
story/memory.db
story/projections/...
~~~

这样以后存储变化不会重写 UI。

---

# 90. Schema Forward Compatibility

所有持久化 JSON：

~~~ts
{
  schemaVersion: number;
  ...
}
~~~

读取：

- 当前版本 -> 正常。
- 旧版本 -> migration。
- 新版本 -> 拒绝写入，提示升级。
- 未知字段 -> 默认保留或由 schema policy 决定。

禁止 silently drop unknown persisted fields。

---

# 91. Import / Mutation / Commit 的统一事务哲学

三个入口最终都落到：

~~~text
Staging
-> Validate
-> Diff
-> Confirm when required
-> Transaction
-> Canon
-> Projection rebuild
~~~

不要每种功能各写一套保存方式。

---

# 92. 开发合并顺序

注意：以下只是 **实现与合并顺序，不是 MVP / V1 / V2**。所有项目都属于同一个终态 Release Scope。

## Merge Set A — Contracts & Architecture

完成全部：

- packages/webnovel 目录。
- 全部核心 Schema。
- stable IDs。
- Migration registry。
- Ports。
- Action contracts。
- API contracts。
- Event contracts。
- Projection layout。
- Test fixtures。

**A 未完成，不允许开始大规模 UI。**

## Merge Set B — Canon / Mutation / Projection

完成：

- Canon adapters。
- Mutation validation。
- Hook lifecycle。
- Timeline。
- Character history。
- Graph projector。
- Projection rebuild。
- Migration。

## Merge Set C — Context System

完成：

- Protected Recent Prose。
- Profiles。
- Retrieval pipeline。
- Graph expansion。
- Dedup。
- Budget。
- Context Trace。
- Core contributor registration。

## Merge Set D — Chapter Workflow

完整实现：

- Preflight。
- Plan。
- Write。
- Review。
- Revise。
- Settle。
- Validate。
- Commit。
- Recovery。

不是只实现 Write。

## Merge Set E — Intelligence

完整实现：

- deterministic audit。
- semantic audit。
- evidence drill-down。
- fix proposal。
- style。
- analysis。
- memory search。

## Merge Set F — Studio Workbench

一次按最终导航实现：

- Dashboard。
- Chapter。
- Outline。
- Characters。
- World。
- Hooks。
- Timeline。
- Graph。
- Memory。
- Audit。
- Style。
- Analysis。
- Production。
- Runs。

可以分 PR，但不建立临时路由或临时页面结构。

## Merge Set G — Production / Import / Export / CLI

完成：

- Production Queue。
- Recovery。
- Budget。
- ApiSaverWriter import。
- Export。
- CLI 全入口。

## Merge Set H — Hardening

- 200 / 1000 章 benchmark。
- 性能。
- E2E。
- Migration regression。
- Failure injection。
- Docs。

---

# 93. Release Gate：完成定义

只有全部满足以下条件，才称为“WebNovel Workbench 完成”。

## Architecture

- 无第二套 Runtime。
- 无第二套 Canon。
- 无 UI 直写 Story State。
- 所有 persisted schema versioned。
- 所有 projection 可重建。

## Authoring

- Dashboard。
- Outline。
- Chapter Workbench。
- Character。
- World。
- Hook。
- Timeline。
- Graph。
- Memory。
- Audit。
- Style。
- Analysis。
- Production。
- Runs 全部完成。

## Context

- Protected Recent Prose。
- Dynamic Profile。
- Long-term retrieval。
- Context Trace。
- Omitted reason。
- Current vs historical separation。

## Consistency

- Character current state。
- Character knowledge。
- Timeline。
- Hook lifecycle。
- Canon facts。
- Whole-book audit。

## Reliability

- Atomic commit。
- Rollback。
- Stale workspace recovery。
- Production checkpoint。
- Projection rebuild。
- Migration。

## Testing

- Unit。
- Integration。
- Flow。
- Studio E2E。
- 200 chapter CI。
- 1000 chapter benchmark。

没有“先上线再把 Timeline / Graph / Audit 补回来”的完成定义。

---

# 94. 禁止项

Code Review 必须阻止：

1. 新增第二套 WebNovel SQLite 作为 Canon。
2. Zustand / React state 成为 Truth。
3. Graph 成为 Truth。
4. Search Index 成为 Truth。
5. Audit 自动直接改正文。
6. Plan 直接写 Canon。
7. Writer 绕过 Context Compiler。
8. Recent Prose 进入普通 BM25 竞争导致可能消失。
9. 模型补造 Timeline 日期。
10. 无 Evidence 写 Mutation。
11. Current State 与 State History 独立维护两份事实。
12. WebNovel 自己实现 Provider。
13. WebNovel 自己实现第二套 Agent Loop。
14. 临时 API 后续再换。
15. 临时 DB 后续再迁。
16. 临时 ID 使用 chapter title / array index。
17. 第一版 Graph 无 canonicalRefs。
18. API route 直接写本地文件。
19. UI 知道内部 story/state 文件路径。
20. Production Runner 绕过单章 workflow。
21. 为追求功能速度牺牲 rollback。
22. 把老项目迁移逻辑散落在业务代码里。
23. Schema 没版本号。
24. Projection 无 revision。
25. Context Trace 只记录 selected，不记录 omitted。

---

# 95. 对 ApiSaverWriter 的最终吸收清单

直接吸收“思想并独立实现”：

| ApiSaverWriter | InkOS WebNovel 终态 |
|---|---|
| Context Profile | 8 类 Profile + primary/secondary 混合 |
| previous chapters | Protected Recent Prose |
| currentState | Canon Character Current State |
| stateHistory | Canon Mutation-derived History |
| Knowledge Graph | Evidence-backed Projection |
| memory fields | Canon-linked Chapter Memory |
| Hybrid Retrieval | RetrievalProvider Port，默认 FTS5/BM25 + semantic selector |
| foreshadow aging | Hook computed overdue |
| book.audit | Three-layer Whole-book Audit |
| context trace | 完整 Context Inspector |
| review center | Chapter Review + Audit Center |
| style binding | Versioned Style Profile |
| book analysis | Analysis Domain |
| rankings | MarketProvider Adapter |
| mobile workflows | Studio responsive；不改变 Domain |
| ApiSaver runtime | 不引入 |
| ApiSaver provider coupling | 不引入 |

---

# 96. 最终设计的核心优势

完成后，InkOS WebNovel 和普通“AI 小说编辑器”的区别不是 UI，而是数据与执行模型：

~~~text
普通工具：
Prompt + 最近章节 + 模型
        |
        v
正文

InkOS WebNovel：
Author Intent
+ Chapter Outline
+ Current Canon
+ Protected Recent Prose
+ Structured Memory
+ Hooks
+ Timeline
+ Character Knowledge
+ Graph Projection
+ References
+ Style
+ Skills
        |
        v
Governed Context
        |
        v
Traceable Agent Workflow
        |
        v
Draft
        |
        v
Review
        |
        v
Evidence-backed Mutation
        |
        v
Atomic Canon Commit
        |
        v
Rebuildable Intelligence Views
~~~

---

# 97. 最终原则

整个改造必须始终坚持五句话：

**一、Canon 只有一个。**

**二、Projection 随时可以删掉重建。**

**三、AI 的分析和建议不是事实。**

**四、每次写章都必须可解释、可验证、可回滚。**

**五、所有分批实现都服务于同一个终态架构，不存在 MVP 临时结构。**

最终目标：

> **让作者敢把一本 1000～3000 章、数百万字的长期连载交给这套系统管理，而不是写到一半再因为状态、伏笔、时间线或架构问题推翻重做。**
