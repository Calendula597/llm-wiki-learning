# kimiwork-memory-prompts

KimiWork（Kimi 桌面版）记忆功能 **Dream Agent** 的系统提示词（System Prompt）原文提取，供学习研究。

## 来源

从 Kimi 桌面端安装目录中提取：

```
D:\Program Files (x86)\Kimi\resources\resources\daimon-bundle.tar.gz
└── app/daimon/dist/src/core/memory/vault/
```

所有文件均为字节级原样拷贝，未做任何修改。

## 这是什么

KimiWork 的记忆系统是一个"夜间做梦"的离线 Agent：白天 Agent 与用户对话，夜里 Dream Agent 醒来，把当天的对话巩固进一个 Obsidian 风格的 vault（即"记忆空间"），再由脚本组装出 `about_user.md` 注入到后续对话中。

vault 的结构：

```
vault/
├── about_user.md      # 脚本组装，注入日常对话（只读）
├── index.md           # 脚本生成的实体目录（只读）
├── log.md             # 脚本维护的梦境日志（只读）
├── sections.yaml      # 装配配置（inject_char_cap 等）
├── sections/          # 稳定轮廓：personal_context / work_context / this_month /
│                      #   earlier_months / taste / memory_tips
└── entities/          # 具体细节：people / projects / places / concepts
```

## 文件清单

| 文件 | 作用 |
|---|---|
| `prompts/system.md` | Dream Agent 的 System Prompt 本体 |
| `prompts/trigger.md` | 每次"做梦"的触发消息，定义执行流程（Phase 0 TRIAGE → INGEST → ENTITIES → SECTIONS → RETURN） |
| `model-context/values.md` | 核心规范《Profile Update Spec v19, Vault Edition》——什么该记、写到哪、怎么写 |
| `prompts/awareness.md` | 注入到日常对话的用户画像块（`<meta awareness="low">`） |
| `prompts/memory_space.md` | 注入到 dream 上下文的 vault 只读快照 |
| `model-context/config.yaml` | 路径配置单一真理源 |
| `model-context/skills/obsidian-markdown/` | Obsidian Flavored Markdown 技能 |
| `model-context/skills/about-me-avatar/` | 用户头像（1-bit 像素风）生成技能 |

## 设计要点速览

- **写"事实盒"而非"人物素描"**：只记录具体事实、原话、时间戳，禁止 "X is a Y kind of person" 这类句式
- **默认克制**：`NO_UPDATE` 是常态，"When in doubt: don't write"
- **反流水账**：更新实体必须先读后整体重写（consolidated rewrite），禁止 append
- **模型只写散文，从不排版**：frontmatter、脚注引用、文件路径全由工具自动处理
- **硬约束**：敏感话题静默丢弃；地址只到城市级；反巴纳姆效应；正文上限约 6000 字符
- **两段式存储**：sections 存稳定轮廓，entities 存具体细节，用 `[[wikilink]]` 互联

## 免责声明

本仓库内容版权归 Moonshot AI 所有，仅用于个人学习与研究。
