# TencentDB-Agent-Memory LLM-Wiki 设计方案

> 来源：对 [`TencentDB-Agent-Memory`](https://github.com/TencentDB-Agent-Memory/TencentDB-Agent-Memory) monorepo 中 MemoryKnowledge（Knowledge Service，下称 KS）服务的源码级梳理。
> 这是 Karpathy「LLM Wiki」模式在**多租户生产服务**中的落地实现——把"个人 Obsidian vault + 单个 Agent"的玩法，改造成"上传源文档 → 排队摄取 → FTS5 + 知识图谱检索"的服务端管线。

## 1. 定位与边界

KS 默认端口 8421，API 前缀 `/v3`，负责两类知识资产：**LLM-Wiki**（本文档）与 Code-Graph。管控面在 MemoryPanel（推送 LLM 绑定、接收状态回调、写远端元数据）。

LLM-Wiki 的核心循环：

```
raw/sources/ (不可变源文档)  ──ingest──▶  wiki/ (LLM 维护的 markdown 页面)
                                              │
                                              ▼
                                     index.db (FTS5 + 图 + 源状态)
                                              │
              查询 ◀── BM25 种子 + wikilink 多跳图扩展 ──┘
```

与 RAG 的本质区别（Karpathy 语义的工程化）：知识被**编译一次并持续维护**，查询时不再是临时拼 chunk，而是从已归一化的 wiki 页面 + 图结构上检索。

## 2. 总体架构

```
MemoryKnowledge/src/
├── routes/wiki.ts          # HTTP 面：资产 CRUD / ingest / search / pages / graph / raw* 文件操作
├── engines/wiki/
│   ├── manager.ts          # WikiSourceManager：摄取编排、分词、FTS5、读模型、搜索入口
│   ├── ingest-v2/          # 摄取引擎（三阶段管线）
│   ├── graph-search.ts     # wikilink 图多跳扩展（BFS + 衰减）
│   ├── index-db.ts         # index.db 生命周期：建库、读池、写事务
│   └── types.ts            # WikiPage / SearchResult 等契约
└── store/
    ├── wiki-service.ts     # 资产级生命周期 + 取消语义 + 回调
    └── build-queue.ts      # 按资产串行队列
```

关键设计决策：**wiki 的正文真相在磁盘 markdown，索引真相在 SQLite**。两者靠"摄取末尾同一写事务"对齐（§5.4），重启后从磁盘扫描重建读模型。

## 3. 数据布局

每个 wiki 资产一个目录：

```
<dataRoot>/<service_id>/<team_id>/<wiki_id>/
├── raw/sources/     # 用户上传的源文档（不可变）
├── wiki/            # LLM 生成/维护的 markdown 页面（含 index.md / log.md / overview）
└── index.db         # SQLite：wiki_fts + page_meta + graph_edge + source 四张表
```

- `raw/` 是 source of truth，LLM 只读不写；
- `wiki/` 完全由 LLM 所有，用户手工编辑通过 frontmatter `locked: true` 声明保护；
- `index.db` 是可重建的派生物，不建表不返回空——读连接拿不到库视为"wiki 未创建"直接抛错。

## 4. 页面规范（frontmatter 契约）

每页必须是带合法 YAML frontmatter 的 markdown（解析失败宽容降级为 `type: "other"`，不抛错）：

| 字段 | 语义 |
|---|---|
| `type` | **必填**。生成页类型：`source / entity / concept / comparison / query / synthesis`；结构页（index.md、log.md）由脚本维护 |
| `title` / `description` / `tags` / `timestamp` | 常规元数据 |
| `sources: string[]` | 本页由哪些源文件喂养——增量合并、级联删除的依据 |
| `locked: true` | 用户手工锁定，ingest 合并时必须跳过 |

frontmatter 是 ingest 合并、BM25 建索引、图建边三条链路共同依赖的**唯一契约**，见 `engines/wiki/ingest-v2/frontmatter.ts`。

## 5. Ingest 管线（ingest-v2）

### 5.1 增量判定

摄取前先从 `index.db` 的 `source` 表读出 `{filename → sha256, status}` 作为基线；磁盘源文件按 sha256 diff 得出 新增 / 变更 / 删除 / 未变 四组，**未变的源直接跳过**（重复摄取的常见路径，用规则挡住，不烧 LLM token）。

### 5.2 阶段一：抽取（extracting）

- `two-stage`（默认）：LLM 先"分析"产出抽取计划，再按计划生成 FILE 块；`single-stage` 省一次调用，源全文直接产出页面。源文本字符预算 `SOURCE_CHAR_BUDGET = 28_000`（余量留给 prompt 框架与输出）。
- 产出是 `Map<relPath, markdown>` 候选页。`relPath` 由 LLM 输出间接推导，**落盘前必须过 `isInsideRoot` 越界卡口**——这道检查必须早于任何文件读取，否则越界路径内容会被读进 merge prompt 而外泄。

### 5.3 阶段二：合并（merging）

候选页按页聚合（一个源可产出多页，多个源可喂养同一页），逐页走 `mergePage` 决策树：

```
页不存在 ──▶ 直接写
locked   ──▶ skip（保护用户手工编辑）
规则判重  ──▶ 候选正文 ⊆ 旧页正文（空白归一化后子串）→ 不调 LLM，
              仅把 sources 取并集写回（同一源重复摄取的典型命中）
旧页正文 > 4000 字符 ──▶ 追加模式：LLM 只产出"增量片段"，旧正文原样保留
否则                 ──▶ 整页重写：LLM 合并两版，frontmatter 失败兜底退回候选页
```

三个 token/质量权衡都落在 `merge.ts`：**规则判重 > 追加 > 重写**，成本和风险从左到右递增。合并受全局 LLM 并发限流（`globalLlmLimit`）。

### 5.4 阶段三：索引 + 状态登记（indexing，强一致）

摄取完成后：

1. `scanWikiDir` 从磁盘扫描全部页面；
2. `withWriteDb` **单事务**内完成：清空并重建 `wiki_fts / page_meta / graph_edge` 三表 + 登记各源摄取结果 + 删除已消失源的行；
3. 事务提交后 `wal_checkpoint(TRUNCATE)` 并关闭写连接；
4. `evictWikiDb` 丢弃该 wiki 的读池连接（防旧快照），下次查询重开即见新索引。

**索引、源状态、删除级联在同一事务**——这是整个一致性设计的锚点：任何一个环节失败都不会留下"索引新、状态旧"的中间态。

### 5.5 结构文件与日志

合并后 `rebuildIndexFile` 重建 `index.md`（内容目录），`appendIngestLogBatch` 追加 `log.md`（时间线日志，失败仅 warn 不阻断主流程）。概览页由 `generateOverview` 单独生成（收集各页 brief 喂给 LLM）。

### 5.6 进度上报

三阶段（extracting / merging / indexing）进度回调做节流：阶段切换立即发，同阶段 500ms 间隔内仅 percent 上升才发——多源并发摄取时避免打爆前端。

## 6. 检索子系统

### 6.1 混合分词（CJK 关键）

SQLite FTS5 自带 tokenizer 对中文无效，方案是**JS 侧预分词 + unicode61 原样存储**：

- 英文：按标点/空格切词，去 stop words，保留完整单词；
- 中文：bigram + 全词双写（`l0录入` 这类中英混合 token 先按边界拆分再分别处理）；
- 写入与查询走**同一个 `tokenize()`**，保证索引/查询两侧逻辑一致。

### 6.2 BM25 种子

`ftsSearch`：查询词分词后每 token 加 `*` 前缀、OR 连接做前缀匹配；`bm25(wiki_fts, 5.0, 1.0)` 标题权重 ×5（对齐 MiniSearch 的 boost），取负转为正分供图扩展的 decay 语义使用。

### 6.3 图多跳扩展

`graphMultiHopSearch`（graphology 内存图 + BFS）：

- 种子（BM25 命中）冻结在 hop=0、保留原始分；
- 沿 `[[wikilink]]` 边逐层扩散，每层得分 ×decay（默认 0.5），低于 minScore（默认 0.1）剪枝；
- 多种子路径命中同一节点取最高分；visited 硬上限 200（防稠密图 DoS）；hop 上限 5；
- 结果携带 `hop` 与 `via`（上一跳页标题），前端可区分"直接命中"与"图扩展"。

默认 `hop=0`（纯 BM25），图扩展是可选增强。

## 7. 并发、一致性与可靠性

| 问题 | 方案 |
|---|---|
| 同一 wiki 并发摄取 | `BuildQueue`：每个资产 key 一条 `SerialQueue`，天然串行；重复 ingest 命中 pending/processing → HTTP 409 busy |
| 摄取中删除 | 内存 `cancelled` 集合置位，worker 在检查点读到即中止（Node 单线程 + 串行队列保证无竞争） |
| 读/写竞争 | 读连接按 wiki 进池复用；写连接每次独立创建、事务内完成、checkpoint 后关闭；写后强制 evict 读连接 |
| ingest 中 LLM 挂了 | 创建 LLM client 失败不 throw——降级为只重建索引 + 落源状态，保证 source 表与磁盘不脱节 |
| 合并部分失败 | 记 `mergeErrors`；某源全部候选合并失败才判该源 failed，写回 `source.status` 供下次重试 |

## 8. API 设计（Hono，`/v3`）

- **资产层**：`POST /get`（`service_id + wiki_id` 双键寻址）
- **摄取**：入队即返；进度经回调/轮询透出
- **检索**：`search` 支持 `hop / decay / minScore / limit` 参数
- **页面/图**：`pages`、`graph`（供 Panel 知识图谱可视化）
- **raw 文件操作**：`rawWrite / rawLs / rawRm`，写路径统一走 `maybeWriteError` 映射：409 processing / 400 越界或结构性文件 / 413 超限
- **回写**：ingest 完成后回调 Panel（`TMC_CALLBACK_URL`），由 Panel 写远端 meta

## 9. LLM 接入

`LlmClient` 抽象为最小接口（`chat(params) → string`），测试可打桩。配置归一化（`normalizeLlmConfig`）强制 `baseUrl/apiKey` 由上层注入，**禁止静默回退直连**：

- `LLM_MODE=proxy`（默认）：按 `x-tdai-service-id` 用 Panel 推送的 `llm_binding`（多租户各自绑模型）；
- `LLM_MODE=custom`：环境变量直连（支持 `openai` / `anthropic` 协议）。

## 10. 安全边界

1. **路径越界**：LLM 产出的 `relPath` 落盘/读盘前必须 `isInsideRoot` 校验；
2. **结构性文件保护**：index.md / log.md 等脚本维护文件拒绝用户写入；
3. **大小限制**：rawWrite 超限 413；
4. **ID 注入**：`service_id/wiki_id` 等路径段校验 `isValidIdSegment`；
5. **内容审计**：资产操作写审计日志。

## 11. 与 Karpathy 模式的对照

| Karpathy 原文 | 本实现的工程化 |
|---|---|
| 三层架构（raw / wiki / schema） | raw/ 与 wiki/ 目录 + frontmatter 即"schema 契约"（可机读、可校验） |
| Ingest：一个源触碰 10-15 页 | 三阶段管线：抽取 → 策略化合并 → 事务内重建索引 |
| index.md / log.md | 同名文件保留，由脚本重建/追加，而非依赖 Agent 自觉 |
| Obsidian 图视图 | `graph_edge` 表 + 多跳 BFS 扩展，且默认关闭（hop=0）按需开启 |
| 单人单 Agent | 多租户 `service_id` 隔离 + 每资产串行队列 + 全局 LLM 限流 |
| LLM 自觉维护 | `locked` 保护人工编辑、规则判重省 token、失败降级不断链 |

**一句话总结**：把"LLM 是程序员、wiki 是代码库"的理念，做成了带增量摄取、事务索引、图检索和多租户隔离的服务端编译管线——Karpathy 管"模式"，这里管"可靠性"。
