# llm-wiki-learning

围绕 **LLM-Wiki** 模式的学习资料收集：从理念原文到生产级实现，后续会加入自己的设计实践。

## 什么是 LLM-Wiki

区别于 RAG 的"每次查询重新拼凑知识"，LLM-Wiki 让 LLM **增量地构建并维护一个持久 wiki**——结构化的、互相链接的 markdown 文件集合，位于用户与原始资料之间。知识被编译一次并持续更新，而不是每次重新推导。

## 仓库结构

| 目录 | 内容 | 角色 |
|---|---|---|
| `karpathy-llm-wiki/` | Karpathy 的 [LLM Wiki 理念原文](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) | 模式定义：三层架构（raw sources / wiki / schema）、Ingest-Query-Lint 操作循环 |
| `kimiwork-memory-prompts/` | KimiWork 记忆功能（Dream Agent）的系统提示词，提取自 Kimi 桌面端 | 同一模式的**生产级实现**：Obsidian 风格 vault、夜间巩固、dream loop |
| `tencentdb-agent-memory-wiki/` | **TDAI**（[TencentDB-Agent-Memory](https://github.com/TencentCloud/TencentDB-Agent-Memory) 开源栈）MemoryKnowledge 服务的源码级设计方案 | 同一模式的**多租户服务端实现**：增量摄取管线、FTS5 + 图检索、事务索引 |

## 三份材料的对照

Karpathy 的文章给出抽象模式，KimiWork 的 prompt 展示了它在工业产品里如何落地，TencentDB-Agent-Memory 把它做成了多租户服务端管线：

- **三层架构** ↔ KimiWork 的"模型写散文 / 工具管格式 / 脚本管装配"
- **Ingest / Query / Lint** ↔ Dream Agent 的 INGEST 阶段与 `vault_status` 体检
- **index.md / log.md** ↔ KimiWork vault 中完全同名的脚本维护文件
- **"LLM 是程序员，wiki 是代码库"** ↔ KimiWork 的反流水账机制（consolidated rewrite）与 NO_UPDATE 默认姿态
- **个人 vault + 单 Agent** ↔ TDAI 的按租户摄取队列、FTS5 + wikilink 多跳检索、事务内重建索引（见 `tencentdb-agent-memory-wiki/DESIGN.md`）

## 计划

- [x] 放入自有项目的 LLM-Wiki 设计（`tencentdb-agent-memory-wiki/`）

## 版权说明

- `karpathy-llm-wiki/` 内容来自 Andrej Karpathy 公开的 gist
- `kimiwork-memory-prompts/` 内容版权归 Moonshot AI 所有，仅用于个人学习与研究
