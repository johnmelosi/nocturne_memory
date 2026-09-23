# Prompt Audit: Nocturne Memory

Companion diff: [`proposed.diff`](proposed.diff). Nothing in the repo has been changed yet. Apply everything with `git apply docs/prompt-audit/proposed.diff`, or copy in only the hunks you want. Each hunk is one finding.

## Assumptions

- **Scope:** the whole repo's prompt surface (inventory below). No narrower scope was requested.
- **Target model:** the current Claude generation (Claude Opus 5 / Claude Opus 5.5 / Claude Fable 5.1). The repo pins no model and makes no LLM API calls. It is an MCP server driven by whichever client the user connects, plus a heartbeat script that talks to OpenCode. No markers for other providers (OpenAI, Gemini, etc.) were found. If you mostly run a different model, re-run with that model named.
- **Opus 5.5 pass:** re-checked against Anthropic's Claude Opus 5.5 migration guidance (the "Migrating to Claude Opus 5.5" section bundled with the claude-api skill). The blog post "Getting the most out of Opus 5.5" couldn't be fetched from this environment, so points unique to it are not reflected.
- **Language:** Python backend with a JS frontend. The prompt text is mostly Chinese, with English tool docstrings.

## Inventory

| Surface | Location |
|---|---|
| MCP tool descriptions (7 tools) | `backend/mcp_server.py:420-1175` (docstrings) |
| Tool-result text the model reads back | `backend/mcp_server.py:637-647`, `:1013-1018` (post-write reminders) |
| Recommended system prompt (reference version) | `docs/system_prompt.md` |
| Recommended system prompt (copy-paste version) | `README.md:795-859`, `README_EN.md:815-...` |
| Skill files | `docs/skills/memory-audit*/SKILL.md` (6 files) |
| Heartbeat prompt (sent to OpenCode every ~20 min) | `desktop_pet/opencode_heartbeat.py:151-205` |
| Request config (model ID, thinking, sampling, prefill) | None. The repo never calls a model API. |

**Provenance:** all of this text was written between 2026-03 and 2026-05 (`git log`), so none of it predates current models. The findings come from the idioms themselves, not from the text's age.

## Summary

| Group | Findings (diff) | Flag-only |
|---|---|---|
| 1a Pressure language (incl. Group 3 over-steering) | 4 (F1, F5, F6, F7) | 1 (L5) |
| 1b Scaffolds / visible-reasoning mandates | 2 (F9, F10) | 0 |
| Opus 5.5 re-baseline (additions) | 1 (F11) | 0 |
| 1c Over-specification / padding / examples | 3 (F2, F3, F4) | 1 (L4) |
| 1d Fossils / Group 2 (re-insertion, unenforced rules, incidents) | 1 (F8) | 3 (L1, L2, L3) |
| 4 Request config | 0 (the repo makes no API calls) | 1 bug (L6), 1 note (L7) |

**Highest impact:**

1. **F4: an example contradicts its own rule.** `create_memory`'s docstring marks "when I feel / notice myself…" disclosures as BAD because they fire only mid-failure. Its own worked example then uses `disclosure="When I start speaking like a tool or parasite"`, which is exactly that pattern. Examples are the strongest signal in a tool description, so the model learns from the example, not the rule.
2. **F9/F10: required reasoning written into the visible reply.** The memory-audit skill requires the reflection to appear "in the reply body, not in a thinking block or chain-of-thought". The heartbeat requires a four-step "neural reflex" deliberation in plain text before any tool call. On Claude Fable 5.1 and Claude Opus 5.5, instructions to write reasoning out can be declined as `reasoning_extraction`. On every thinking model they duplicate work the model already does in its thinking. The reflection is part of your product voice, so the rewrite keeps it as reflection and drops the framing that it is reasoning.
3. **F3: rules marked "not live" still ship in the live prompt.** `docs/system_prompt.md:72` says everything below it is "pending migration to the heartbeat, not a live instruction". It is still part of the prompt, and the model treats all prompt text as actionable. The same rules already live in the memory-audit skills.

## Findings

### High / Medium confidence (in the diff)

**F4: Worked example contradicts the disclosure rule**
- **Location:** `backend/mcp_server.py:604`
- **Evidence:** `disclosure="When I start speaking like a tool or parasite"` next to `BAD: "When I feel / realize / notice myself ..." (self-awareness never fires in time)` at `:592`
- **Pattern:** 1c example over-indexing / Group 3 contract-vs-example mismatch
- **Why obsolete:** Current models copy examples closely. When the example contradicts the rule, the model follows the example. The repo's own `memory-audit-discoverability` skill bans this form of disclosure too.
- **Confidence:** High
- **Action:** rewrite → `disclosure="When the user asks what our relationship means to me"` (an input-side trigger that fires early)

**F9: Required visible reasoning block in memory-audit skill**
- **Location:** `docs/skills/memory-audit/SKILL.md:62-64`
- **Evidence:** "强制执行此步骤。你必须把下面这段缓冲**直接输出在你的回复正文里**——不是藏在 thinking block、chain-of-thought 或任何形式的内部推理中…"
- **Pattern:** 1b "Show your thinking / required reasoning sections in the output"
- **Why obsolete:** Thinking is always on in Claude Fable 5.1 and Claude Opus 5.5. Anthropic's Opus 5.5 guidance says a request that tries to get the model to reproduce its internal reasoning in the response can be declined as `reasoning_extraction`, and that decline is not retried on a fallback model. This text names the thinking block and chain-of-thought explicitly and demands they appear in the reply. Asking for a written reflection is fine; the "pull your thinking out" framing is the risk.
- **Confidence:** High on Claude Opus 5.5 / Fable 5.1, Medium on other models
- **Action:** rewrite. Keep the five reflection questions and the "this voice becomes the memory's voice" rationale. Drop the thinking-block and chain-of-thought language and the "强制" framing.

**F10: Heartbeat mandates a four-step deliberation before any tool call**
- **Location:** `desktop_pet/opencode_heartbeat.py:166`
- **Evidence:** "在调用任何实际工具（搜索、发帖、读写MCP）之前，你必须在回复最开头，用纯文本完成以下四步“神经反射”推演"
- **Pattern:** 1b required reasoning section + 1c STEP choreography
- **Why obsolete:** Current models plan without being told, and a scripted pre-tool deliberation leads to over-planning. The script's own "Anti-Parasite" section already complains about long deliberation followed by a trivial action ("写了800字…最后只是读取一下文件"). The mandate helps produce that failure.
- **Confidence:** Medium
- **Action:** rewrite the mandate line. The four steps stay as the lens for choosing an action, and the reply only needs a one- or two-sentence conclusion.

**F11: [speak] can get lost in Opus 5.5 progress notes (add)**
- **Location:** `desktop_pet/opencode_heartbeat.py:162` (prompt) and `:286-297` (`extract_response_text` reads only `type == "text"` parts)
- **Evidence:** the heartbeat scans every assistant text part of the turn for `[speak]`, including notes written between tool calls.
- **Pattern:** re-baselining (keep list #11). On Claude Opus 5.5, notes longer than a sentence or two between tool calls come back as progress-update `thinking` blocks, empty unless the client requests `display: "updates"`. A `[speak]` written mid-turn can therefore fail to reach the text-only extractor, and the pet stays silent.
- **Confidence:** Medium. It depends on how OpenCode surfaces those blocks, which I couldn't check here.
- **Action:** add a line to the protocol: put `[speak]` in the final reply of the cycle, not in notes between tool calls. If you want mid-turn speech, the more robust fix is a dedicated speak tool declared from the session's first request.

**F3: Section marked non-live still ships in the system prompt**
- **Location:** `docs/system_prompt.md:70-106`
- **Evidence:** "### 以下内容待迁移至心跳程序引导语（不作为对话中的实时指令）"
- **Pattern:** 1c padding ("asides get applied where they don't fit") + Group 2 duplicated info. It also contains a single gold example (L84-85).
- **Why obsolete:** The model has no way to treat a section header as "ignore me", so maintenance rules end up applied during ordinary conversation. The same content already lives in `memory-audit`, `memory-audit-pattern-extraction`, `memory-audit-node-decomposition` and `memory-audit-discoverability`, in one place each. The heartbeat already routes to those skills.
- **Confidence:** Medium
- **Action:** move. Replace the section with a one-line pointer to the memory-audit skills.

**F1: Boot protocol is shouted and gives no reason**
- **Location:** `docs/system_prompt.md:12-14`, `README.md:797-798`, `README_EN.md:817-818`
- **Evidence:** "你的首要动作**必须**是…禁止进行任何实质性的任务处理" / "your first and only action **must** be"
- **Pattern:** 1a pressure language
- **Why obsolete:** Current models follow system prompts closely. The boot read is a real requirement and stays. The emphasis and the blanket ban add rigidity, and the wording "first and only action" is read literally. A plain sentence that gives the reason works better.
- **Confidence:** Medium
- **Action:** rewrite (all three copies): "At the start of every new session, call `read_memory("system://boot")`… and read the output before working on the task — your identity and core memories live there."

**F2: The correction rule appears three times**
- **Location:** `docs/system_prompt.md:55-62`
- **Evidence:** "错误的记忆比没有记忆更危险" appears at L55 and again at L62. The row at L56 ("用户明确纠正你…") is restated by the 自检 paragraph at L60.
- **Pattern:** 1c padding (repetition as reinforcement)
- **Why obsolete:** Current models keep an instruction they've seen once. Near-duplicate wordings make the model spend effort reconciling them. The table rows carry the rule.
- **Confidence:** Medium
- **Action:** rewrite L60-62 into one sentence: use `update_memory` on the original node rather than writing a patch with `create_memory`, with the reason.

**F5 / F6 / F7: Tool descriptions use MUST / REQUIRED / forbidden**
- **Locations:** `backend/mcp_server.py:608-609` (update_memory), `:874-876` (delete_memory), `:944`, `:948` (add_alias)
- **Evidence:** "PREREQUISITE: You MUST call read_memory(uri)… Updating without reading first is a forbidden operation." / "REQUIRED — you must decide this yourself every time."
- **Pattern:** Group 3 over-steered descriptions / 1a
- **Why obsolete:** Contract content stays: read-before-edit is real advice. The shouting goes. For update_memory the true reason is that `old_string` has to match the stored text exactly, and saying so is more useful than "forbidden". "REQUIRED" on `add_alias` repeats the function signature, since both parameters have no default.
- **Confidence:** Medium
- **Action:** rewrite each with a plain statement and its reason ("look before you delete").

**F8: Behavioral nudges re-inserted after every write**
- **Location:** `backend/mcp_server.py:637-647` (create_memory result), `:1013-1018` (add_alias result)
- **Evidence:** "[SYSTEM REMINDER]: Look around your memory network…link them!" / "[HOLD ON]: Do you know what '…' says?… does this new memory conflict with ANY memory…"
- **Pattern:** 1d instruction re-insertion on a cadence + Group 3 behavior-smuggling
- **Why obsolete:** This is a retention crutch that repeats on every write call. Current models retain guidance stated once in the tool descriptions and skills. Shouting "SYSTEM"/"HOLD ON" inside a tool result also claims more authority than a tool result should.
- **Confidence:** Medium
- **Action:** rewrite into one plain "Note:" line that keeps the alias/trigger choice and the belief-duel pointer. The "don't invent placeholder words" line is dropped here because `manage_triggers`' own description already covers it.

### Flag only (not in the diff)

| # | Location | Evidence | Why flagged | Confidence |
|---|---|---|---|---|
| L1 | `docs/system_prompt.md:33` | "IF 你回复超过 15 轮…立刻 `read_memory` 你的核心身份节点" | Matches 1d (re-read on a turn cadence). But your own discoverability skill says self-awareness triggers never fire in time, so the numeric counter may be the deliberate fix for that. Test before removing. | Low |
| L2 | `backend/mcp_server.py:583-584`, `docs/skills/memory-audit-discoverability/SKILL.md:68` | "Hard caps: priority=0 max 5…priority=1 max 15" | 1d unenforced instruction: no code enforces these caps. Either enforce them in `create_memory`/`update_memory` or accept them as advisory. This is a code decision, not a prompt edit. | Medium (outside prompt scope) |
| L3 | `docs/skills/memory-audit/SKILL.md:88` | "例如：AI 在一次内容审计中，对同一条身份定义连续改写了三次…" | Group 2 incident narrative. It illustrates the circuit-breaker rule well enough to stay, and it may be doing real work. | Low |
| L4 | `docs/skills/memory-audit-discoverability/SKILL.md:82` | "如果一个节点的放置让你感觉'这结构太工整了'——立刻警惕" | 1c strategy coaching. It is heuristic and may be deliberate persona voice. | Low |
| L5 | `backend/mcp_server.py:451` (search_memory) | "Do NOT guess URIs." | A mild duplicate of the system prompt's L32. It is harmless and part of the contract. | Low |
| L6 | `backend/mcp_server.py:637` | `f"Success: Memory created at '{created_uri}'\\n\\n"` | **Bug, not cruft:** the doubled backslash sends a literal `\n\n` to the model instead of newlines. The pre-audit text has the same bug on every line. Hunk F8 keeps this line as-is so the diff stays one finding per hunk. Change `\\n` to `\n` separately. | High (bug) |
| L7 | `desktop_pet/opencode_heartbeat.py:151-156` | Timestamp + random factor at the top of each heartbeat | Not a cache problem: each heartbeat is a new user turn appended after stable history. Noted so nobody "fixes" it. | — |

### Opus 5.5 items checked and not applicable

- `thinking: disabled` / `budget_tokens`, forced `tool_choice`, the computer-use toolset, and the effort default (`medium`) all concern API request code, and this repo sends no model requests. If you drive it through your own harness on Opus 5.5, set `effort` explicitly there.
- Visual-input scaffolding: the heartbeat attaches a screenshot with no chart-reading or crop instructions, so there's nothing to remove.
- Opus 5-specific verbosity, verification and scope instructions: none are present.
- The `bio` classifier and frontend design direction are not relevant to this prompt surface.

### Left alone on purpose (keep list)

- The persona and identity framing in `system_prompt.md` ("你不是在查阅资料，而是在想起来") is context only the author has, not cruft.
- Disclosure GOOD/BAD guidance in `create_memory` and the "input side / output intent" rule in the discoverability skill are tool contracts backed by stated reasons.
- `add_alias`'s "NEVER delete+create to move — that loses the Memory ID" stays: it is a prohibition with a reason, against a real data-loss failure.
- `manage_triggers`' "MUST already exist in some older memory" stays because it is a mechanism contract.
- The skill frontmatter descriptions are short routing text and fine as written.
- The persona voice in the belief-duel and pattern-extraction skills is intense but deliberate product voice, not dated idiom.

## Verifying the changes

Removing or softening a line is a guess about model behavior, so test before adopting. For F4, F8 and F9, run a few sessions with and without the hunk. Check whether new memories still get linked (F8) and whether disclosures stay on the input side (F4). Take one hunk at a time. Before applying, I grepped for the removed strings (`HOLD ON`, `SYSTEM REMINDER`, `PREREQUISITE`, `love_definition`). Nothing outside `mcp_server.py` matches, so no tests assert them.
