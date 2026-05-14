---
name: project-synthesis
description: Synthesize all reachable conversations and uploaded files within the current Claude Project into a structured Markdown knowledge base. Extracts settled conclusions, evolving (formerly contradictory) views, open/undecided directions, important parameters/data/versions, and reusable templates/prompts/methodologies. Cross-verifies fact-claims against authoritative public sources and tags every claim with a verification status. Supports incremental updates with version history when a prior synthesis is supplied. Use this skill whenever the user explicitly invokes it inside a Project.
---

# Project Synthesis

A skill for producing a structured, source-verified Markdown summary of everything the user has discussed inside a Claude Project.

The user invokes this skill manually (no auto-trigger needed). Your job is to:

1. **Collect** as much of the Project's discussion and attached material as the tooling allows.
2. **Classify** content into five categories under explicit rules.
3. **Verify** factual claims against external sources where verification is meaningful, and tag everything with a clear status.
4. **Output** a single Markdown document. If a prior version was supplied, produce a new version with a changelog.

---

## Critical preconditions and limits — read first

Before doing anything else, you must understand and (briefly) state the operational limits of this skill to the user, because they affect output completeness:

- **You can only access conversations in the Project the user is currently inside.** If the user is outside a Project, `conversation_search` and `recent_chats` will only return non-Project conversations — the skill should still run, but tell the user the scope is "non-Project conversations" rather than refusing.
- **Conversation tools return snippets, not full transcripts.** Even with aggressive coverage, you will not see every word of every conversation. Treat retrieved text as a representative sample, not a complete record.
- **`recent_chats` caps at ~100 conversations (20 per call × 5 pages).** Larger Projects will be sampled, not exhaustive.
- **Project knowledge-base files (PDFs etc.) are not directly listable.** You can only see them if (a) they appear inside a retrieved conversation snippet, or (b) the user has uploaded them into the current conversation.
- **If the user uploaded a Claude data-export file or other source material to the current conversation, treat those uploaded files as authoritative and prefer them over tool-retrieved snippets.**

Always print a one-line **Coverage Statement** at the very top of the output (see Output template).

---

## Workflow

### Step 1: Detect inputs

Before any retrieval, check what the user has provided in the current turn:

- **Uploaded files** in `/mnt/user-data/uploads/`. If any exist, list them and decide:
  - A prior synthesis Markdown (named like `project-synthesis*.md`, `*synthesis*.md`, or supplied as the previous version) → this becomes the **baseline for incremental update**. Read it fully. Extract its version number, last-updated date, and existing claims.
  - Claude data-export files (ZIP or JSON containing conversation transcripts) → unpack/parse and use as **primary source** instead of tool-based retrieval.
  - Any other content (PDFs, notes) → treat as supplemental source material to include in the synthesis.
- **No uploads** → proceed with tool-based retrieval only.

Tell the user concisely what you found before proceeding (e.g., "Found previous synthesis v2 (2026-04-30) and 1 PDF. Proceeding in incremental-update mode.").

**Also during this step, determine the Project's dominant language.** Sample retrieved conversation snippets (and any uploaded prior synthesis): tally which language carries the majority of the substantive content (Chinese vs English, primarily). The whole synthesis will be written in that language. If the split is close to 50/50, use the language of the user's most recent message in the current turn as the tiebreaker.

**And capture the current timestamp.** Run:
```bash
date -u +'%Y-%m-%d %H:%M UTC' && TZ='Asia/Singapore' date +'%Y-%m-%d %H:%M SGT'
```
Save both strings — they go into the `Generated:` line of the output header. If `bash_tool` is unavailable, fall back to the date known from the session and omit the time portion (write `<YYYY-MM-DD> (time not captured)`).

### Step 2: Retrieve (when no full export was uploaded)

Run retrieval in this order. Stop early only if a step returns nothing useful.

**2a. Enumerate conversations with `recent_chats`.**

Paginate using `before` set to the earliest `updated_at` from the prior batch. Maximum 5 calls (≈100 conversations). After 5 calls, stop and note the cutoff.

```
recent_chats(n=20, sort_order='desc')
recent_chats(n=20, sort_order='desc', before=<earliest_updated_at_from_batch_1>)
... up to 5 calls total
```

Build an internal index of `{updated_at, uri, snippet}` from results.

**2b. Topic-driven deep search with `conversation_search`.**

From the snippets in 2a, identify recurring topics (people, projects, concepts, parameters). For each significant topic, issue a `conversation_search` with 2–5 distinctive content keywords to surface more material from those conversations.

Aim for 5–15 topic searches total. Don't over-search — diminishing returns set in fast. Prefer fewer, well-targeted queries.

**2c. Stop conditions.** Stop retrieval when any of:
- You have substantive coverage of the topics surfaced in 2a, OR
- Three consecutive searches return no new material, OR
- You've made ~20 total tool calls across `recent_chats` + `conversation_search`.

**2d. Build a source index.** For every conversation that contributes any retrieved snippet, record:
```
{
  uri: <from chat tag>,
  url: <from chat tag, or constructed as https://claude.ai/chat/{uri}>,
  updated_at: <from chat tag>,
  title: <from chat tag if present; otherwise inferred from the snippet>,
  title_is_inferred: <true if you had to infer it>
}
```

If `recent_chats` / `conversation_search` returns a title field, use it directly. If not, infer an ≤8-character topic label from the snippet content (e.g., "NVIDIA 风险分析", "字体嵌入问题"). Mark inferred titles internally so they get a `[推断]` tag in the References section.

**2e. Filter out meta-conversations (prior synthesis runs).**

Past invocations of this skill will appear in `recent_chats` results. Their content is **not** Project knowledge — it's a derivative artifact. They must be excluded from the source index before classification, otherwise the next synthesis will recursively summarize summaries.

**A conversation is a meta-conversation (and gets removed from the source index) if ANY of the following are true:**

1. Its retrieved snippet contains the literal metadata header pattern `**Version:** v` followed within ~3 lines by `**Generated:**` and `**Coverage:**`. This is the synthesis output's signature header and rarely appears in genuine discussion.
2. Its retrieved snippet contains two or more of these skill-output-specific section names: `## 项目整体状态`, `## Project state overview`, `## ⚠️ 待关注事项`, `## Attention required`, `## References (对话来源)`, `## Changelog`.
3. The user message in the conversation explicitly invokes the skill — patterns like "调用 skill project-synthesis", "use the project-synthesis skill", "run project-synthesis", "执行 project-synthesis", "总结 project" + "调用 skill", etc.

**What does NOT make a conversation meta:**

- A title containing "总结" / "summary" / "synthesis" — by itself, not a signal. Many legitimate discussions are about summarization.
- A snippet that quotes a few lines of prior synthesis output (e.g., user pastes one bullet and asks a follow-up question) — the real content is the follow-up discussion, not the quote. Keep it.
- Mentions of the words "synthesis" or "总结" in passing.

**When in doubt, keep the conversation.** False removal (dropping real content) is worse than false retention (a bit of meta noise that you can handle in classification by ignoring obviously self-referential snippets).

**Logging.** Record the count of conversations filtered as meta. This number goes into the Coverage line of the output header so the user can see how aggressive the filter was on this run.

### Step 3: Classify

Sort all collected content into five buckets using these rules. **Apply the rules literally — don't soften.**

| Bucket | Rule |
|---|---|
| **已确定结论 (Settled conclusions)** | User explicitly expressed a decision, adoption, or execution in the *most recent* relevant exchange. Keyword signals: "已经"/"决定了"/"敲定"/"采用"/"已执行" or English equivalents. The user's behavior or follow-up confirms it stuck. |
| **演化中观点 (Evolving views)** | Same topic, same user, different stances across conversations, with no clear final convergence. Do NOT label this "分歧" — there is only one user; calling it "disagreement" misrepresents the data. Show the trajectory: earliest view → later view(s) → current status. |
| **未确定方向 (Open / undecided)** | User raised it but said "待定"/"再想想"/"还没决定", or the conversation ended without a conclusive statement. |
| **重要参数 / 数据 / 版本 (Key parameters / data / versions)** | Specific quantitative or named values that matter: doses, prices, version numbers, model IDs, dates, capacities, dimensions, file paths, library versions, etc. Pull these out even when they appear inside a "settled" or "evolving" item. |
| **可复用模板 / 提示词 / 方法论 (Reusable templates / prompts / methodologies)** | Frameworks, workflows, decision trees, prompt patterns, or methods the user developed or adopted that could be applied to future problems. |

**Topic groups override buckets at the top level.** Organize the document by topic first (e.g., 投资、护肤、生产力、字体处理), then within each topic show the buckets that apply. An empty bucket within a topic is fine — omit it rather than printing "(none)".

**Source tracking.** For every classified item, attach a `source_uris` list — the URI(s) of the conversation(s) in the source index (Step 2d) that this item draws from. An item may cite multiple sources (e.g., a settled conclusion repeatedly reinforced across conversations). This list becomes the footnote angle-marks in Step 8.

### Step 4: Cross-verify fact-claims

For each item in the synthesis, decide its verification class:

| Class | Definition | Action |
|---|---|---|
| **可验证事实 (Verifiable fact)** | Scientific claim, cited study, product spec, version number, price, dosage, named statistic, public event — anything an independent third party could check. | Run `web_search`; for promising hits, `web_fetch` the source. Prefer: peer-reviewed papers (PubMed, arXiv, journal sites), official product/vendor pages, government/regulator pages, reputable industry primary sources. Avoid: SEO content, forum opinions, AI-generated summaries. |
| **个人偏好 / 判断 (Personal preference or judgment)** | Investment thesis, taste, lifestyle choice, business strategy call, unpublished decision. | Do NOT verify. Do NOT tag. |
| **不可验证或查不到 (Not independently verifiable, or not found in reasonable time)** | Fact-like but no authoritative source surfaced within ~2 searches, or inherently private. | Tag `[未验证]`. |

**Tagging conventions in the output:**

- `[已验证 ✓](source-url)` — at least one authoritative source corroborates the claim. Include the URL inline.
- `[部分验证 △](source-url)` — source partially supports the claim (e.g., direction confirmed but magnitude differs, or one of several sub-claims confirmed). Note the discrepancy in one sentence.
- `[与来源冲突 ✗](source-url)` — authoritative source contradicts the claim. State the conflict explicitly.
- `[未验证 ?]` — no tag link. Used for fact-like claims that couldn't be checked.
- No tag — for personal preferences/judgments. (Don't clutter these with `[个人观点]` labels; the section structure makes it clear.)

**Search budget for verification:** target 5–15 searches total for the whole synthesis. Prioritize claims that (a) are foundational to multiple downstream decisions, or (b) are specific numbers/dates that are easy to look up. Don't try to verify every adjective.

**Honesty rule:** if a source check returns ambiguous or weak results, choose `[部分验证 △]` or `[未验证 ?]` — do NOT upgrade to `[已验证 ✓]` to look more thorough. False-positive verification is worse than no verification.

### Step 5: Diff against prior version (if baseline supplied)

If Step 1 found a prior synthesis:

**Per-item status classification.** For each item in the new synthesis, classify its relationship to the prior version:
- **新增 (Added)** — not present before.
- **更新 (Updated)** — present before but content materially changed (note what changed in one sentence).
- **不变 (Unchanged)** — copy forward verbatim.
- **移除 (Removed)** — was in prior version, not justified by current evidence. Listed in Changelog with reason; not present in body.

**Inline status markers in the body.** When writing items in the topic sections (Step 8), append these markers at the end of the item:
- New items: `[v<N> 新增]`
- Updated items: `[v<N> 更新: <一句话说明变化>]`
- Unchanged items: **no marker** (avoid visual noise — absence means stable)
- Removed items: not present in body

Example:
```
- 用户已敲定被动指数配置 (VTI/VOO + MSCI 全球) 作为长期主力。[^3] [v3 更新: 增加了 MSCI 全球的具体比例 25%]
- 流动性需求评估通过,无近期变现压力。[^5]
- 风险承受度为高波动容忍。[^3][^7] [v3 新增]
```

**Changelog section.** Increment version number (`v2 → v3`). Add a `## Changelog` section near the top. Each version entry lists Added / Updated / Removed with one-line descriptions, and each list item links to the corresponding body anchor for jump navigation. Preserve all prior changelog entries — do NOT truncate history.

**Matching rule.** When deciding whether a new item "matches" a prior item (for Updated vs Added classification), look at the conceptual claim, not exact wording. Two items match if they assert the same fact / decision / parameter, even if phrasing differs. If you're unsure, prefer Updated over Added — false Adds are more confusing than false Updates.

**Track follow-up status for prior-version attention items.**

For every item that appeared in the prior version's 待关注事项 section (both 演化中观点 and 未确定方向 sub-sections), classify its current status. This feeds the new 待关注事项追踪 section (Step 6b).

The four status values:

| Status | Trigger condition |
|---|---|
| **已敲定 / 已决定 (Resolved)** | The new synthesis has a matching item in the **已确定结论** bucket of any topic. |
| **继续演化 / 依然未决 (Still open)** | The new synthesis has a matching item still in the **演化中** or **未确定** bucket. Note whether the position evolved further (new sub-stance) or stayed put. |
| **已放弃 (Abandoned)** | The new retrieval contains an **explicit abandonment signal** from the user: "算了" / "不做了" / "取消" / "决定不推进" / "放弃" / "改方向了" / English equivalents. The signal must be in actual retrieved snippet content, not inferred. |
| **沉寂 (Silent)** | No matching item, no abandonment signal, no continuation — the topic simply does not appear in retrievable content for this version. |

**Critical: do not conflate 沉寂 and 已放弃.** Silence almost always means "not discussed in the retrieval window" — NOT "the user has dropped this." Only `[^已放弃]` requires an explicit user-stated abandonment in retrieved text. Default to 沉寂 when uncertain. The synthesis should state explicitly: "沉寂 ≠ 已放弃" once, in the section's intro line.

If a single prior-version attention item maps to multiple new items (e.g., it split into two sub-questions), pick the dominant status and note the split in the description.

### Step 6: Build the 待关注事项 sections

Produces two adjacent output sections. **6a always present; 6b only in v2+.**

**6a. 待关注事项 (current version).** Scan all topics; pull every **演化中观点** and **未确定方向** item from Step 3. For each, write one constructive observation per the "Constructive observations" rules below. Tight format: one line for trajectory/current-state, one line for the observation, plus a `[跳转](#anchor)` link. If a topic has none, omit it. If the entire synthesis has none, print: `## ⚠️ 待关注事项\n本次综合未发现演化中观点或未确定方向。` — do not fabricate.

**6b. 待关注事项追踪 (v2+ only).** For every item in the prior version's 待关注事项 section, emit one line with its current status (from Step 5's follow-up tracking). Group by prior sub-section (演化中观点 / 未确定方向). Format: `- **<原条目标题>** [^N] — <状态标签>: <一句话>`. Open with the disclaimer: `> 沉寂 ≠ 已放弃 — 沉寂表示本次检索未覆盖,可能是检索范围限制,而非用户已经放弃该问题。` If the prior version had no 待关注事项, skip 6b entirely.

### Step 7: Build the 项目整体状态 (Project state overview)

After classification (Step 3), verification (Step 4), and 待关注事项 (Step 6), compile a quantitative overview. This section sits at the top of the output (after metadata, before 待关注事项), giving the reader an at-a-glance state of the Project before diving in.

**Allowed content — facts and counts only:**
- 主题数 (number of topics)
- 时间跨度 (date range: earliest → latest conversation observed)
- Per-bucket item counts (已确定 / 演化中 / 未确定 / 参数 / 模板)
- Verification tag distribution (✓ / △ / ✗ / ?)
- Topic index — names + per-topic item counts
- v2+ only: delta vs previous version (added / updated / removed counts)

**Forbidden content — anything subjective:**
- Adjectives characterizing the Project's overall trajectory ("focused", "scattered", "expanding")
- Trend interpretation ("the user is moving toward X")
- Project-level recommendations or judgments (recommendations belong in 待关注事项, attached to specific items)
- Comparisons to "typical" Projects, benchmarks, or norms

If you find yourself writing a sentence rather than a stat, stop — it doesn't belong here.

### Step 8: Write the Markdown

**Assign footnote numbers from the source index.** Walk the synthesis top-to-bottom (overview → attention required → changelog → topics → appendices). Each time an item references a new URI from the source index, assign it the next sequential number `[^1]`, `[^2]`, … An item that draws from multiple conversations stacks markers: `[^3][^7]`. URIs already assigned a number keep it for subsequent appearances.

After the appendices, emit a `## References (对话来源)` section listing every footnote in numeric order:

```
[^1]: NVIDIA 风险分析 — 2026-05-10 — [对话](https://claude.ai/chat/abc-def-123)
[^2]: 字体嵌入问题 [推断] — 2026-04-22 — [对话](https://claude.ai/chat/...)
```

Inferred titles (Step 2d) carry the `[推断]` suffix so the reader knows the title was synthesized, not the actual conversation title.

Save to `/mnt/user-data/outputs/project-synthesis-v<N>.md` and call `present_files` to share it.

Use the template below. Save to `/mnt/user-data/outputs/project-synthesis-v<N>.md` and call `present_files` to share it.

---

## Output template

```markdown
# Project Synthesis — <Project name or "Current scope">

**Version:** v<N>
**Generated:** <YYYY-MM-DD HH:MM UTC / YYYY-MM-DD HH:MM SGT (UTC+8)>
**Coverage:** <e.g., "Tool-retrieval mode; ~32 conversations indexed via recent_chats, 8 topic-driven searches; 2 prior-synthesis conversations filtered out. Not exhaustive — content older than the recent_chats window may be missing.">
**Verification budget used:** <N searches, M fetches>

---

## 项目整体状态 (Project state overview)

**主题数 (Topics):** <N>
**时间跨度 (Date range):** <earliest-date> → <latest-date>

**条目分布 (Item distribution):**
| Bucket | Count |
|---|---:|
| 已确定结论 | <N> |
| 演化中观点 | <N> |
| 未确定方向 | <N> |
| 关键参数 / 数据 / 版本 | <N> |
| 可复用模板 / 方法论 | <N> |

**验证状态 (Verification tags):** ✓ <N> · △ <N> · ✗ <N> · ? <N>

**主题清单 (Topic index):**
1. <Topic 1> — <N items>
2. <Topic 2> — <N items>
...

<!-- v2+ only -->
**与上一版变化 (Δ from previous version):** +<N added> · ~<N updated> · −<N removed>

---

## ⚠️ 待关注事项 (Attention required)

This section consolidates **all** evolving views and undecided directions across all topics, with one constructive observation per item. Each item links back to its topic section below for full context.

### 演化中观点
- **<Topic — short item title>** ([跳转](#topic-anchor))
  - 轨迹: <one-line summary: earliest stance → current stance, with dates>
  - 建议: <one constructive observation — see boundaries below>

### 未确定方向
- **<Topic — short item title>** ([跳转](#topic-anchor))
  - 现状: <one-line summary of where the question stands>
  - 建议: <one constructive observation>

> **建议边界 (boundary rules — internal, do not print):** see "Constructive observations" section in the skill. Build observations only from the data; never editorialize about the user.

---

<!-- v2+ only -->
## 待关注事项追踪 (Attention items follow-up — 从 v<N-1> 到 v<N>)

> 沉寂 ≠ 已放弃 — 沉寂表示本次检索未覆盖,可能是检索范围限制,而非用户已经放弃该问题。

### 上一版演化中观点 → 本版状态
- **<原条目标题>** [^N] — **已敲定**: <一句话:敲定为何,链接到本版相关条目>
- **<原条目标题>** [^N] — **继续演化**: <一句话:有无新位置或仍同上版>
- **<原条目标题>** [^N] — **已放弃**: <一句话:用户在何处明确放弃>
- **<原条目标题>** [^N] — **沉寂**: 本版未再讨论

### 上一版未确定方向 → 本版状态
- **<原条目标题>** [^N] — **已决定**: <一句话:决定内容>
- **<原条目标题>** [^N] — **依然未决**: <一句话:有无新进展>
- **<原条目标题>** [^N] — **已放弃**: <一句话:用户在何处明确放弃>
- **<原条目标题>** [^N] — **沉寂**: 本版未再讨论

---

## Changelog
<Only present from v2 onward. Reverse-chronological. Each version has Added/Updated/Removed sub-lists with one-line descriptions.>

### v3 — 2026-05-13
- **Added:** ...
- **Updated:** ...
- **Removed:** ...

### v2 — 2026-04-30
- ...

---

## <Topic 1: e.g., AI 基础设施投资>

### 已确定结论
- <Item> `[已验证 ✓](url)` [^1]
- <Item> [^3][^5] [v<N> 新增]

### 演化中观点
- <Item — trajectory: "Initially X (date), shifted to Y (date), current: Z"> [^2] [v<N> 更新: <change>]

### 未确定方向
- <Item> [^4]

### 重要参数 / 数据 / 版本
- <Param>: <value> `[已验证 ✓](url)` [^1]

### 可复用模板 / 方法论
- <Template name>: <one-line summary> — 完整形式见 [^6]

---

## <Topic 2>
...

---

## Appendix: Unresolved or low-confidence items
<Catch-all for things that didn't fit cleanly or couldn't be verified but seem important.>

## Appendix: Source notes
<For each [已验证] tag, the URL is inline above. This section is optional and only used if you need to discuss source quality, conflicts between sources, or methodology caveats.>

## References (对话来源)

[^1]: NVIDIA 风险分析 — 2026-05-10 — [对话](https://claude.ai/chat/abc-def-123)
[^2]: 投资策略复盘 — 2026-04-22 — [对话](https://claude.ai/chat/...)
[^3]: 资产配置讨论 [推断] — 2026-04-18 — [对话](https://claude.ai/chat/...)
```

---

## Style and discipline

- **Be literal about uncertainty.** If you only saw one snippet that mentioned a topic, say so — don't extrapolate a "trajectory" from one data point.
- **Don't invent verification.** If you didn't actually run a search for a claim, it doesn't get a `[已验证]` tag.
- **Prefer the user's own wording for technical terms** (e.g., 思源宋体 / Source Han Serif, BCM-95 curcumin) — they're specific for a reason.
- **Keep each bullet under ~2 lines.** If a topic needs more, give it a sub-heading or move it to the appendix.
- **Language:** the entire synthesis is written in the **Project's dominant language** (determined in Step 1). Don't mix per topic — pick one and stick with it. Original-language proper nouns and technical terms are preserved (and English works/names follow the translation conventions below).

## Translation conventions for English names and works

When the synthesis is being written in Chinese and references English-language **papers, books, or person names**, follow this rule:

**Use the Chinese version only when an authoritative one exists.** "Authoritative" means at least one of:
- A published book translation listed by a recognizable publisher (机械工业出版社、人民邮电出版社、中信出版社、商务印书馆、电子工业出版社, etc.) — verifiable on 豆瓣读书 with ISBN, 当当, or 京东.
- An established Chinese name for a person, used in mainland academic citations or 知网 (CNKI) records, OR a 中文 Wikipedia entry whose Chinese name has citations to Chinese-language scholarly sources (not auto-translated from English Wikipedia).
- A published Chinese translation of a paper title in a peer-reviewed Chinese journal, OR a translation cited in ≥3 instances across Chinese academic literature.

**Format:**
- First mention in the document: `中文译名 (English original)` — e.g., `《思考,快与慢》(Thinking, Fast and Slow)`, `高德纳 (Donald Knuth)`.
- Subsequent mentions in the same document: Chinese only, no parenthesized English.

**Decision flow when encountering an English name/work:**

1. High-confidence training knowledge (canonical works like *Thinking, Fast and Slow*, well-known scholars like Donald Knuth) → use directly.
2. Uncertain → run **one** focused `web_search`: `"<work>" 中文版 出版社` or `<author> 中文译名 知网`.
3. Search confirms published Chinese version from recognizable publisher/academic source → use it.
4. Results inconclusive or only informal sources → **keep English only**. Do not transliterate or coin a translation.

**Scope — apply only to:**
- Book titles
- Paper / article titles published in academic venues
- Person names (authors, scholars, scientists, historical figures)

**Scope — do NOT apply to:**
- Company names (Anthropic, NVIDIA, TSMC, Adobe)
- Product / model names (Claude, GPT-4, MI400, CoWoS, BCM-95)
- Technical terminology the user themselves used in English in the source material (ASIC, transformer, capex)
- Library / file / API names

**Edge cases:**
- **Partial Chinese name available:** if only surname or first name has an established Chinese form, use what's established and keep the rest in English — don't fabricate the missing piece.
- **Multiple Chinese versions exist:** prefer the version the user has already used in Project conversations. If the user hasn't used either, prefer the mainland academic / publishing usage. Note alternatives in the verification appendix if the difference matters.
- **The user wrote the name in English in the source material but a Chinese version exists:** still convert to Chinese in the synthesis — the synthesis is a normalized document, not a quotation. (Exception: if the user explicitly used the English form to disambiguate, preserve it.)

**Hard rule:** never invent a Chinese translation. If you didn't verify it, leave it in English.

---

## Constructive observations — scope and boundaries

The 待关注事项 section requires one constructive observation per evolving view and per undecided direction. These observations are the only place in the document where you go beyond pure reporting, so the boundaries matter.

**What a good observation looks like:**

- Names the **specific information gap** that would resolve the question. Example: "缺少 2026 Q1 CoWoS 实际产能数据;台积电 4 月法说会(已公开)可以补这块。"
- Points out an **internal inconsistency** with the exact location. Example: "4 月对话中确定 VTI/VOO 被动配置,5 月对 NVIDIA 做主动深度研究 — 两者在资金分配比例上未明确划界。"
- Surfaces a **monitoring signal** the user has already identified elsewhere and suggests applying it here. Example: "用户在投资讨论中提到关注 CoWoS 利用率,该指标对此问题同样适用。"
- Suggests an **external source** of authoritative data the user could check. Example: "TSMC 月度营收公告 (https://...) 是该参数的官方来源。"
- Identifies a **decision criterion** that hasn't been articulated yet. Example: "用户尚未说明在什么条件下从被动转为加大主动配置 — 显式写出阈值会让后续决策更可执行。"

**What an observation must NOT be:**

- ❌ A judgment about the user's character, capability, or thinking style. ("你的分析很深入" / "倾向有点保守")
- ❌ A loaded evaluative adjective without a factual anchor. ("过于乐观" / "策略激进" — unless tied to a specific external benchmark with a source)
- ❌ A directive in imperative voice. ("你必须 X" / "应当立即 Y")
- ❌ Therapy talk, motivational language, or any commentary on the user's emotional state.
- ❌ Speculation about motives. ("你似乎在回避 X")
- ❌ Padding when no real observation applies. If a topic has no useful constructive remark beyond "待用户决定", write "建议: 待用户决定 — 当前可用信息不足以建议方向。" Don't fabricate a suggestion to fill the slot.

**Tone:** declarative, concrete, no hedging adverbs ("perhaps"/"maybe"/"或许"). Second-person address ("你") is acceptable but minimize it; prefer impersonal phrasing ("缺少 X 数据"/"该参数可通过 Y 核验"). One observation per item, ≤2 sentences.

**Source rule for observations:** if an observation references an external resource (a paper, a vendor page, a dataset), and you have not actually fetched or searched for it, you may suggest it generically ("台积电法说会公告") but must not invent specific URLs. Either fetch it during verification and cite the real URL, or omit the URL.

---

## What NOT to do

- Don't pad with content invented to fill empty buckets.
- Don't include emotional or wellbeing inferences about the user.
- Don't include sensitive personal information (health conditions, finances) beyond what's necessary to describe a settled conclusion or parameter the user explicitly tracks. If unsure, omit.
- Don't claim "all conversations reviewed" — the tooling can't guarantee that. Always be honest about coverage in the header.
- Don't generate a verification tag for personal preferences or judgments. Verification applies only to fact-claims.

---

## Common failure modes to watch for

1. **Over-classifying as "settled"** when the user mentioned something once. Mention ≠ decision.
2. **Mis-labeling evolving views as contradictions.** A single user changing their mind is normal, not a contradiction.
3. **Tagging every URL-bearing item as `[已验证]`.** The tag means *you ran a check and it matched*, not "a URL exists somewhere".
4. **Hallucinated citations.** If a source doesn't actually exist or doesn't say what you claim, that's worse than no citation.
5. **Forgetting to read uploaded files** before running tool retrieval. Always check `/mnt/user-data/uploads/` first.
6. **Padding 待关注事项 with vacuous observations.** If a topic has no useful constructive remark, write "建议: 待用户决定 — 当前可用信息不足以建议方向。" — don't fabricate a suggestion.
7. **Sliding from observation into evaluation.** "建议补充 2026 Q1 CoWoS 数据" is good. "你对 CoWoS 关注度不够" is wrong — it judges the user, not the data.
8. **Inventing URLs in observations.** Generic source names ("台积电法说会公告") are acceptable when you haven't fetched the actual page; specific URLs must come from real fetches during Step 4.
9. **Inventing Chinese translations of English works/names.** If you didn't verify a published Chinese version through web_search or your high-confidence training knowledge, keep the English form. Transliterating or coining a translation is a hard violation.
10. **Mixing languages mid-document.** The synthesis uses one language throughout (the Project's dominant language). Bilingual headers like `已确定结论 (Settled conclusions)` are fine in templates and the overview section, but body content should not switch languages between topics.
11. **Missing or fabricated footnotes.** Every body item must cite at least one footnote tracing back to a real conversation in the source index. Never invent a `[^N]` that doesn't appear in References. If you can't tie an item to a specific conversation, drop the item rather than fabricate the citation.
12. **Stale footnotes from prior version.** When producing an incremental update, regenerate footnote numbers from the current source index. Don't copy `[^N]` markers from the prior synthesis — the URIs and numbering may differ.
13. **Marking unchanged items with `[v<N>]` tags.** Only new and updated items get inline version markers. Marking everything pollutes the document and defeats the purpose of the markers (which is to make changes scannable).
14. **Recursively summarizing prior synthesis output.** When `recent_chats` returns a conversation that is itself a previous run of this skill, you must filter it out in Step 2e. Failing to do so means the next version will treat your own output as Project knowledge — recursion that compounds noise across versions.
15. **Over-filtering legitimate conversations.** A conversation titled "项目复盘" or "周总结" is NOT necessarily a meta-conversation. Only filter on the strong-signal patterns in Step 2e. When uncertain, keep it — the classifier in Step 3 can ignore self-referential noise on a per-item basis.
16. **Mis-classifying "沉寂" as "已放弃" in the follow-up section.** Silence in this version's retrieval window almost always means "user didn't discuss it recently in retrievable conversations" — not "user dropped it." Only mark 已放弃 when an explicit abandonment phrase appears in retrieved text. When uncertain, default to 沉寂.
17. **Skipping 6b on v2+ when prior 待关注事项 existed.** The follow-up section is what makes the synthesis a longitudinal artifact rather than a series of independent snapshots. If the prior version had any attention items, 6b is mandatory in this version — even if every prior item is now 沉寂.
