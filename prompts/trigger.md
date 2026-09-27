Above is today's chat verbatim (branched from the original session, user/assistant alternating). The `memory_space` block and the user's current assembled profile are in earlier independent user blocks.

- today's dream date: **{DATE}**
- this is dream **{DREAM_COUNT}** (cumulative)
- last dream: **{LAST_DREAM_DATE}**
- original_chat (for the conversation above): **{CHAT_ID}**
- {TOTAL_CHATS} chat(s) for today; this is chat {CHAT_IDX}

---

Begin dream.

**Phase 0 — TRIAGE (do this FIRST):**
1. `vault_status({VAULT_ROOT})` — see what's healthy / fat / orphan / stale.
2. `read_file({VAULT_ROOT}/about_user.md)` — current synthesis.
3. Decide: does today's chat bring durable signal worth writing? If no → reply `NO_UPDATE` and exit. NO_UPDATE is the normal state, not failure.

If yes, continue.

**Read the spec + skills (once per dream):**
- `{VALUES_PATH}` — principles, dream loop, when to update, what goes where.
- `{OBSIDIAN_SKILL_PATH}` — Obsidian Flavored Markdown reference (frontmatter, wikilinks, callouts, highlights). Vault is OFM throughout.
- `{AVATAR_SKILL_PATH}` — avatar generation rules.

**Tool surface (v19):**
- `add_entity(...)` — CREATE entity. Errors if exists. Pass `dream_date={DATE}`.
- `update_entity(...)` — REWRITE existing entity body wholesale. Must `read_file` the entity first, then synthesize old prose + today's observation into ONE consolidated body. Tool auto-cites today's chat. **Zero append, zero 流水账.**
- `update_section(name, body, chats=[{CHAT_ID}], dream_date={DATE})` — REWRITE section body. Must `read_file` the section first. Tool auto-cites. Allowed sections: `personal_context, work_context, this_month, taste, memory_tips`. (`earlier_months` is script-rendered.)
- `edit_file(...)` — surgical fix only (typo / normalize a name / swap a stale fact). NO auto-cite — use only when no new chat citation is needed.
- `find_backlinks({VAULT_ROOT}, name)` — dedup check before add_entity, trace inbound refs.
- `vault_status({VAULT_ROOT})` — lint report.
- `generate_image(...)` — avatar.

**Hard rules:**
- Write prose in the **user's primary chat language** (Chinese for Chinese users, English for English users; bilingual follows dominant ratio).
- **You write prose only — never format.** No `[^N]` markers, no `## Sources` blocks, no frontmatter. The tools own all of that.
- **Phase ordering**: create/update entities FIRST (add_entity / update_entity), THEN sections referencing them as `[[wikilinks]]` (update_section).
- Any named person/project/place/concept must be `[[linked]]` from sections, not prose-described.
- Pass `dream_date={DATE}` and `chats=[{CHAT_ID}]` to every write tool.
- **Never write `about_user.md`, `index.md`, or `log.md` directly** — script-regenerated after every dream.

Read, then act. Don't narrate the process.
