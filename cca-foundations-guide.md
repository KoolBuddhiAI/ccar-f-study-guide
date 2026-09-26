# Claude Certified Architect – Foundations (CCAR-F / CCA-F)
## A study guide grounded in this repo's real code

*Written 2026-09-26 by the OpenClaw agent that maintains this workspace. Exam facts are cross-confirmed by two independent prep sites (claudecertifiedarchitects.com and certsafari.com), both aligned to the official Exam Guide v1.0. I could not fetch the official PDF directly, so treat details as high-confidence but verify before booking.*

---

## Exam at a glance

| Fact | Value |
|---|---|
| Code | CCAR-F (also written CCA-F) |
| Role | Anthropic's Architect track, Foundations level |
| Cost | $125 USD |
| Items / time | 60 items / 120 minutes |
| Format | 4 scenarios drawn from a bank of 6; multiple-choice **and** multiple-response |
| Pass mark | Scaled 720 / 1000 |
| Delivery | Proctored — Pearson VUE online (OnVUE) or test center |
| Validity | 12 months; renewal = free non-proctored assessment on Anthropic Partner Academy |
| Prereq (informal) | 6+ months building with Claude APIs, Agent SDK, Claude Code, MCP |
| Official prep | Anthropic Partner Academy (anthropic-partners.skilljar.com) — partner-network members |

**The five domains and their weights:**

1. Agentic Architecture & Orchestration — **27%**
2. Claude Code Configuration & Workflows — **20%**
3. Tool Design & MCP Integration — **18%**
4. Prompt Engineering & Structured Output — **20%**
5. Context Management & Reliability — **15%**

Each section below: what the exam tests → where that exact concept lives in this repo → one-line takeaway.

---

## Domain 1 — Agentic Architecture & Orchestration (27%)

### What the exam tests
- **The agentic loop**: send request → inspect `stop_reason` → if `tool_use`, execute the tool, append the result to conversation history, loop → stop when `end_turn`.
- **Anti-patterns they explicitly test**: parsing assistant text for "done" as a termination signal; arbitrary iteration caps as the *primary* stop mechanism; treating any tool error as loop-fatal.
- **Coordinator–subagent (hub-and-spoke)**: a coordinator owns all routing, error handling, aggregation. Subagents have **isolated context** — they don't inherit the coordinator's history; you must pass complete prior findings in the subagent's prompt. Spawn parallel subagents with multiple `Task` calls in **one** coordinator response. Give subagents goals + quality criteria, not step-by-step procedures.
- **Programmatic enforcement vs prompt guidance**: when compliance must be deterministic (identity check before a refund), use hooks/prerequisite gates — prompt instructions alone have a non-zero failure rate.

### In this repo
- **The session you're reading is a live agentic loop.** My turns in #cca-prep this morning: search → read → fetch → write, each tool result appended before the next decision, ending only when I emit final text. That's `stop_reason: tool_use` vs `end_turn` in the wild.
- **`skills/youtube-digest/scripts/digest.py` → `run_pipeline()` (line 510)**: a multi-step workflow (discover → filter → summarize → cluster → post) with two programmatic gates:
  - `check_quota()` (line 93) blocks the run *before* burning API quota — a prerequisite gate, not a prompt suggestion.
  - The `sent_video_ids` ledger dedupes by video ID, so a re-run after a successful delivery can only waste quota, never double-post. That's **idempotency**, the reliability half of orchestration.
- **Coordinator–subagent, verbatim**: my `sessions_spawn` tool takes `context="isolated"` for a clean-context child and requires me to pass *the entire task in the prompt* — the child sees nothing of my conversation. The exam's "subagents don't inherit parent context" is an API fact I live with daily.
- **`workshop-skills/youtube-digest-recovery/SKILL.md` §3a**: before force-running the digest job, three preconditions must all hold (check `date -u`, no digest file today, 429 reset passed). Block-first, act-second — the same pattern as "block `process_refund` until `get_customer` returned a verified ID."

### Takeaway
> Loop on `tool_use`, stop on `end_turn`; never on words or counters. Gates belong in code, hints belong in prompts.

---

## Domain 2 — Tool Design & MCP Integration (18%)

### What the exam tests
- **Tool descriptions are the #1 selection mechanism**: state input formats, example queries, edge cases, boundaries. Minimal descriptions → unreliable selection among similar tools.
- **Error handling for recoverability**: return a *structured* result describing the error (clear message + error type) so the model can read it and decide what to do. Anti-patterns: logging server-side and returning nothing, passing raw exception text with just an `isError` flag, retrying internally forever so the agent never learns the call failed.
- **MCP itself**: Anthropic created the Model Context Protocol — expect direct questions about it (tools, servers, structured results).

### In this repo
- **`digest.py` line 93 — `check_quota()`** returns `(can_proceed, message, units_remaining)`:

  ```python
  def check_quota(safe_limit_pct=0.5):
      """Return (can_proceed, message, units_remaining)."""
      ...
      if used >= safe_limit:
          return False, f"Quota {used}/{YOUTUBE_DAILY_QUOTA}. Aborting.", remaining
      return True, f"Quota {used}/{YOUTUBE_DAILY_QUOTA}. {remaining} remaining.", remaining
  ```

  Not a crash, not a silent log — a result the caller can branch on. (This function also carries the bug I fixed on 2026-09-21: the counter never reset across dates, so it aborted against week-old usage. The fix — treat a stale `date` as zero — is documented in `memory/2026-09-21.md`.)
- **Error taxonomy → distinct recovery paths.** `gemini_call()` (line 161) classifies failures via `_record_call` / `_rate_limit_wait` / `_get_adaptive_delay` (lines 139–159): a 429 gets exponential backoff and retry; an invalid key gets an immediate, clean abort. Same call, different error *type*, different *action* — exactly what the exam means by "the model decides how to proceed."
- **The recovery skill's failure table** is the same idea one level up: `400 API_KEY_INVALID` → owner must fix; `429` → self-heals at stated time; `402` → owner tops up; `Memory Search failed` → different playbook entirely. Each row pairs an error class with a recovery action.
- **My own toolset** demonstrates description discipline: every tool leads with what it's for ("Exact single-file replacements", "Fetch URL; extract readable markdown") because selection depends on it.

### Takeaway
> A tool's job on failure isn't to hide the error — it's to *report the error usefully* so the agent can choose the next move.

---

## Domain 3 — Claude Code Configuration & Workflows (20%)

### What the exam tests
- **CLAUDE.md hierarchy**: user-level (`~/.claude/CLAUDE.md`, *not* shared via version control), project root, subdirectory — nearest file wins for scope.
- **Skills**: `SKILL.md` with frontmatter (name, description); descriptions must lead with trigger phrases; **progressive disclosure** — body loads only when triggered, keeping context lean; optional isolated/fork context so a skill runs in a subagent and returns only its summary.
- **Skills vs memory**: procedures live in skills; facts/logs live in memory files. Don't cross-contaminate.

### In this repo (same patterns, OpenClaw names)
- **`AGENTS.md` / `SOUL.md` / `USER.md`** at the workspace root are my CLAUDE.md-equivalents: workspace-wide conventions, persona, and per-user profile — each with its own scope and 4,000-char load budget. The exam's "user-level vs project-level" split is why `USER.md` holds preferences while `AGENTS.md` holds workspace rules.
- **Two skills I authored this month** (real SKILL.md files, real frontmatter):
  - `~/.openclaw/agents/main/agent/workshop-skills/youtube-digest-recovery/SKILL.md`
  - `~/.openclaw/agents/main/agent/workshop-skills/memory-index-recovery/SKILL.md`

  Look at the description line: *"YouTube Daily Digest automation failed — classify provider or memory-index errors, force-run the job, or deliver a degraded digest manually without re-burning quota."* Leading words = when this triggers. That's the exam-standard description format, and I wrote it *because* the trigger-first rule works — the skill fires only on the failure it names.
- **Progressive disclosure in practice**: neither skill's full body sits in my context until a digest actually fails. The 70-line recovery procedure costs zero tokens on a good day.
- **Skills vs memory, obeyed**: the *procedure* for recovery lives in the skill; the *record* of what broke each day lives in `memory/YYYY-MM-DD.md`. When I got that wrong (procedural lessons buried in daily notes), the AGENTS.md rule "learned a lesson → update the skill" exists precisely to fix it.

### Takeaway
> Configuration is layered by scope; skills are procedures with triggers, not documents that are always loaded.

---

## Domain 4 — Prompt Engineering & Structured Output (20%)

### What the exam tests
- **Explicit categorical criteria beat vague instructions**: "flag a comment only when its claimed behavior contradicts actual code behavior" is testable; "check that comments are accurate" is not. "Be conservative" / "only high-confidence findings" measurably fail to improve precision.
- **Structured output**: for a fixed shape, define a **JSON schema on an extraction tool** and read findings from the `tool_use` response — far more reliable than restating field names in prose. Worked examples that render the exact target shape beat longer descriptions. Placing the requirement immediately before generation helps.

### In this repo
- **Categorical, not vibes — in my own config.** My reply rules aren't "be thoughtful about when to speak"; they're enumerable: *stay quiet when 23:00–08:00, when the last check was < 30 min ago, when nothing is new; reach out when an event is < 2h away.* An auditor could test each condition. That's the exam's precision principle applied to self-governance.
- **Worked-example prompting, verbatim.** Recovery skill step 3: *"Author the digest … using the newest existing digest file under `documentation/youtube-digest/digest/` as the format template."* Instead of describing the format (header line, clusters, 2–3 sentence summaries, footer), I hand the model a concrete exemplar. Same principle as few-shot examples in the exact target shape.
- **Schema over prose.** When I fan out parallel research subagents, I pass an `outputSchema` (JSON Schema) and their results come back as structured JSON I can merge programmatically — the "read findings out of the tool_use response" pattern exactly. My automations payloads are typed JSON for the same reason.
- Even my clarifying questions follow it: structured option chips with one recommended default, not "what do you want?"

### Takeaway
> If you can't write the criteria as a checklist, the model can't either. And when shape matters, schema the output — don't describe it.

---

## Domain 5 — Context Management & Reliability (15%)

### What the exam tests
- **Lost in the middle**: models reliably use the beginnings and ends of long contexts; findings buried mid-input get dropped. Put critical material at the edges or restate it.
- **Progressive summarization risks**: condensing can launder away numbers, percentages, dates, and stated expectations into vague summaries — preserve critical values verbatim.
- Verification under uncertainty: when an operation was interrupted, its outcome is *unknown*, not failed — re-verify before retrying or reporting.

### In this repo (September was a live lab for this domain)
- **The drift incident** (`memory/2026-09-22.md`, "time-tracking drift"): I mistook queued, late-arriving context updates for sequential heartbeats and drifted ~12 hours in my mental model of "now." Fix, now a standing rule: *authoritative time = the timestamped envelope; always `date -u` before judging whether a scheduled job is late.* A perfect example of progressive-summarization failure: the summary ("it's been running all night") destroyed the fact (timestamps).
- **Unknown ≠ failed** (`memory/2026-09-21.md` lesson): gateway restarts kill my turns mid-tool-call; the written rule is *"treat tool results from killed turns as unknown and re-verify."* This week a restart ate my question to you mid-flight — which is why you got the plain-text version.
- **Deliberate summarization tiers**: `memory/YYYY-MM-DD.md` keeps raw facts and numbers; `MEMORY.md` folds only stable decisions; `USER.md` holds directives with `observed/superseded` status so contradictions can't pile up. The daily-note pattern "stale → supersedes → actual state" exists because I once had three conflicting "current states" in one file.
- **Lost-in-the-middle, mitigated**: this guide itself uses the technique — exam facts up top, takeaways at each section's end, detail in between.

### Takeaway
> Context is a budget and a liability: put facts at the edges, keep numbers verbatim, and never trust a summary of something you can check.

---

## Repo artifact index (go read the code)

| Path | What it demonstrates |
|---|---|
| `skills/youtube-digest/scripts/digest.py` | Multi-step pipeline, quota gate (l.93), retry/backoff error classification (l.139–194), idempotent delivery ledger |
| `skills/youtube-digest/state/` | Durable state: quota usage, sent-video ledger |
| `~/.openclaw/agents/main/agent/workshop-skills/youtube-digest-recovery/SKILL.md` | Skill frontmatter, trigger-first description, failure taxonomy, precondition gates, verification criteria |
| `~/.openclaw/agents/main/agent/workshop-skills/memory-index-recovery/SKILL.md` | Second skill; lock quarantining + rebuild rules |
| `documentation/youtube-digest/digest/` | Output format the worked-example pattern references |
| `AGENTS.md`, `SOUL.md`, `USER.md` | Layered context files, scoped and budgeted |
| `memory/*.md` | Daily logs with dates/numbers kept verbatim; correction patterns |

## Where to prep next
- **Official**: Anthropic Partner Academy course path for Architect Foundations (partner-network members; via anthropic-partners.skilljar.com).
- **Independent practice**: claudecertifiedarchitects.com (400 scenario questions, free account), certsafari.com (480 questions + exam-experience reports), Matthew Purcell's CCAR-F practice set (LinkedIn).
- **Book via**: Pearson VUE (pearsonvue.com/us/en/anthropic.html).

*Sources: pearsonvue.com (program structure), certsafari.com (exam details + full domain breakdown, guide v1.0 checked 2026-09-18), claudecertifiedarchitects.com (weights, formats). The official exam-guide PDF link returned binary junk; numbers above are cross-confirmed but verify at booking.*

---

## Deep dives (2026-09-26, later that day)

Full plain-language walkthroughs of all 26 subtopics live in the HTML twin of this guide, served from `documentation/cca-prep/`: `index.html` (hub) → `d1.html` … `d5.html` (one per domain), shared `style.css`. Each subtopic gets: what it means → the distinctions exam questions exploit → a real repo tie-in where one honestly exists.

Subtopic map:

- **D1 Agentic Architecture & Orchestration (27%):** 1.1 agentic loop lifecycle · 1.2 coordinator–subagent hub-and-spoke · 1.3 subagent invocation, context passing, parallel spawning · 1.4 programmatic enforcement + structured handoffs · 1.5 Agent SDK hooks (interception, normalization)
- **D2 Tool Design & MCP (18%):** 2.1 tool descriptions as selection mechanism · 2.2 structured error responses (errorCategory / isRetryable / isError) · 2.3 tool distribution + tool_choice (auto / any / forced) · 2.4 MCP server scoping (.mcp.json vs ~/.claude.json) + resources · 2.5 built-in tools (Grep vs Glob vs Read/Write/Edit)
- **D3 Claude Code Config & Workflows (20%):** 3.1 CLAUDE.md hierarchy + @import + .claude/rules/ · 3.2 slash commands + SKILL.md frontmatter (context: fork, allowed-tools, argument-hint) · 3.3 path-scoped rules (globs) · 3.4 plan mode vs direct execution · 3.5 iterative refinement (examples, test-driven, interview pattern) · 3.6 CI/CD (-p, --output-format json, independent review)
- **D4 Prompt Eng & Structured Output (20%):** 4.1 explicit categorical criteria · 4.2 few-shot for ambiguous cases · 4.3 JSON schemas via tool use (strict ≠ semantic) · 4.4 validation + retry-with-error-feedback · 4.5 Message Batches API (50% cost, 24h window, no mid-request tools) · 4.6 multi-instance review
- **D5 Context Mgmt & Reliability (15%):** 5.1 case-facts blocks + lost-in-the-middle + trimming tool outputs · 5.2 escalation triggers vs non-triggers · 5.3 error propagation (access failure ≠ empty result) · 5.4 scratchpads + crash-recovery state
