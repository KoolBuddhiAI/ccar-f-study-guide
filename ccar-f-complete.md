# Claude Certified Architect – Foundations (CCAR-F / CCA-F)
## Complete study guide — overview + all 26 subtopic deep dives, grounded in real repo work

*Written 2026-09-26 by the OpenClaw agent that maintains the workspace repo. Exam facts cross-confirmed by two independent prep sites (claudecertifiedarchitects.com, certsafari.com) aligned to the official Exam Guide v1.0; the official PDF itself wouldn't fetch — high confidence, verify when booking.*

**Contents:** [Exam at a glance](#exam-at-a-glance) · [D1 Agentic Architecture 27%](#domain-1--agentic-architecture--orchestration-27) · [D3 Claude Code Config 20%](#domain-2--claude-code-configuration--workflows-20) · [D2 Tool Design & MCP 18%](#domain-3--tool-design--mcp-integration-18) · [D4 Prompts & Structured Output 20%](#domain-4--prompt-engineering--structured-output-20) · [D5 Context & Reliability 15%](#domain-5--context-management--reliability-15) · [Repo artifact index](#repo-artifact-index--go-read-the-code)

---

## Exam at a glance

| Fact | Value |
|---|---|
| Code | CCAR-F (also written CCA-F) |
| Cost | $125 USD |
| Items / time | 60 items / 120 minutes |
| Format | 4 scenarios from a bank of 6; multiple-choice **and** multiple-response |
| Pass mark | Scaled 720 / 1000 |
| Delivery | Proctored — Pearson VUE online (OnVUE) or test center |
| Validity | 12 months; renewal = free non-proctored assessment on Anthropic Partner Academy |
| Prereq (informal) | 6+ months building with Claude APIs, Agent SDK, Claude Code, MCP |
| Official prep | Anthropic Partner Academy (anthropic-partners.skilljar.com) |

Domain weights: **Agentic Architecture & Orchestration 27% · Claude Code Configuration & Workflows 20% · Prompt Engineering & Structured Output 20% · Tool Design & MCP Integration 18% · Context Management & Reliability 15%**

Every subtopic below: **what it means → the distinctions exam questions exploit → a real repo tie-in** (paths refer to the workspace repo this guide was built in).

---

# Domain 1 · Agentic Architecture & Orchestration (27%)

## 1.1 The agentic loop — know it cold

**What it means.** Send a request → Claude responds → inspect `stop_reason`. If `tool_use`: execute the tool, **append the result to conversation history**, send the next request. If `end_turn`: done. The model choosing tools from accumulated context is what makes it an agent rather than a pre-scripted chain.

- Model-driven tool choice beats preconfigured decision trees or fixed sequences.
- **Tested anti-patterns:** parsing assistant text for "done" as a stop signal; arbitrary iteration caps as the *primary* stop mechanism; ending the loop on any tool error.
- Tool results must land **in history** between iterations or the model can't use new information.

> **Repo:** the session that built this guide ran the loop live — search → read → fetch → write, each result appended before the next decision, stopping only on final text.

## 1.2 Coordinator–subagent (hub-and-spoke)

**What it means.** One coordinator owns decomposition, spawning, routing, error handling, aggregation. Subagents never talk directly — spokes through the hub, for observability and consistent errors.

- Dynamic selection: invoke only the subagents the query needs, not always the full pipeline.
- Partition subtopics/source types to avoid duplicated coverage.
- **Over-decomposition trap:** slicing too narrowly under-covers broad topics.
- Iterative refinement: evaluate the synthesis for gaps → re-delegate with targeted queries → re-synthesize.

## 1.3 Subagent invocation, context passing, spawning

**What it means.** Subagents start **blank** — no inherited conversation, no shared memory between invocations.

- The `Task` tool spawns subagents; the coordinator's `allowedTools` must include `"Task"`.
- Pass complete prior findings in the prompt (e.g., web-search results handed to the synthesis subagent).
- Structured formats that separate content from metadata (source URLs, doc names, page numbers) preserve attribution between agents.
- Parallelism: multiple `Task` calls in **one** coordinator response.
- Write prompts as goals + quality criteria, not step-by-step procedures.
- `AgentDefinition` covers each subagent type's description, system prompt, tool restrictions; fork-based sessions explore divergent approaches from a shared baseline.

> **Repo:** `sessions_spawn` is this exact API — `context="isolated"` starts clean, `context="fork"` copies the transcript; the task must be fully self-contained either way.

## 1.4 Enforcement & handoff patterns

**What it means.** When ordering *must* happen (identity verified before refund), enforce it in code: prerequisite gates and hooks. Prompt instructions alone have non-zero failure rate.

- Gate example: block `process_refund` until `get_customer` returned a verified ID.
- **Structured handoffs:** escalating to a human who can't see the transcript means compiling customer ID, root cause, amounts, recommended action.
- Multi-concern requests: decompose into distinct items, investigate in parallel with shared context, synthesize one resolution.

> **Repo:** `digest.py` gates its pipeline in code — `check_quota()` (l.93) blocks before any API spend; the `sent_video_ids` ledger makes double-delivery structurally impossible. The recovery skill §3a demands three preconditions before any force-run.

## 1.5 Agent SDK hooks — interception & normalization

**What it means.** The Agent SDK exposes hook points around the tool lifecycle (e.g., `PostToolUse`) where code intercepts a tool call and normalizes output (trim, reformat, redact) before it re-enters context.

- The programmatic counterpart to 1.4 — enforcement at the SDK layer instead of hoped-for in prompts.

**Domain takeaway:** the loop terminates on `stop_reason`, never on words or counters. Subagents are blank slates — packaging context *is* the coordinator's job. Deterministic requirements get code, not prose.

---

# Domain 2 · Claude Code Configuration & Workflows (20%)

## 2.1 CLAUDE.md hierarchy, scoping, modularity

**What it means.** Memory files load by proximity and scope: user-level `~/.claude/CLAUDE.md` (that user only — **not** shared via version control), project-level (`.claude/CLAUDE.md` or root), directory-level for subdirectories.

- **Classic diagnostic:** a new teammate "doesn't follow the rules" — the rules live in someone's user-level file instead of project-level.
- `@import` syntax keeps CLAUDE.md modular per package.
- `.claude/rules/` splits a monolith into topic files (testing.md, api-conventions.md).
- `/memory` shows which memory files are actually loaded — first debugging step for inconsistent behavior.

> **Repo:** same architecture — `AGENTS.md` (workspace rules), `SOUL.md` (persona), `USER.md` (profile, 4,000-char budget), loaded by scope each session.

## 2.2 Slash commands & skills

**What it means.** Project-scoped commands in `.claude/commands/` (VCS-shared) vs user-scoped `~/.claude/commands/` (personal). Skills live in `.claude/skills/` as `SKILL.md` with frontmatter.

- Frontmatter: `context: fork` (isolated sub-agent; verbose or exploratory output never pollutes the main session), `allowed-tools` (restrict tool access — e.g., writes only), `argument-hint` (prompt for required parameters).
- Personal variants go in `~/.claude/skills/` under different names.
- **Skills = on-demand task workflows; CLAUDE.md = always-loaded standards.** Choose accordingly.

> **Repo:** two authored skills (`youtube-digest-recovery`, `memory-index-recovery`) — trigger-first descriptions, lean bodies, verification criteria, and a prose `allowed-tools` rule ("digest.py is operator-owned; never edit it").

## 2.3 Path-specific rules — conditional loading

**What it means.** Rules in `.claude/rules/` carry YAML frontmatter with `paths:` globs; they load only when editing matching files — less irrelevant context.

- Glob rules beat directory CLAUDE.md for cross-directory conventions: `**/*.test.tsx` catches every test file wherever it lives.

> **Repo:** the memory system uses the same trick — repo-scoped facts carry `<!-- project: path:… -->` markers so project knowledge stays attached to its repo.

## 2.4 Plan mode vs direct execution

**What it means.** Plan mode = exploration before commitment: large-scale changes, multiple valid approaches, architectural decisions, multi-file modifications. Direct execution = simple, well-scoped changes.

- Plan: microservice restructuring, migrations touching 45+ files, integration-approach choices.
- Direct: a single-file fix with a clear stack trace, adding one validation conditional.
- The **Explore subagent** isolates verbose discovery and returns summaries — main context stays clean.
- Combination pattern: plan the migration, then execute directly.

## 2.5 Iterative refinement techniques

**What it means.** How to converge on quality across turns.

- **Concrete I/O examples beat prose** — the single most effective technique when descriptions produce inconsistent results. 2–3 input/output pairs.
- **Test-driven iteration:** write the suite first (behavior, edge cases, performance), then share failures each round.
- **Interview pattern:** have Claude ask questions before implementing in unfamiliar domains — surfaces what you didn't anticipate.
- Interacting problems → all issues in one message. Independent problems → sequential fixes.

> **Repo:** the digest format rule is exactly this — "use the newest existing digest file as the format template" hands the model a concrete exemplar.

## 2.6 Claude Code in CI/CD

**What it means.** Claude Code as an automated pipeline stage.

- `-p` / `--print`: non-interactive — prevents input hangs.
- `--output-format json` + `--json-schema`: machine-parseable findings (e.g., auto-posted PR comments).
- CLAUDE.md carries project context (testing standards, fixtures, review criteria) to CI-invoked Claude; feed existing tests in so generation doesn't duplicate covered scenarios.
- **Self-review limitation:** the session that generated code reviews it poorly — an independent fresh-context instance catches more.
- Re-run reviews with prior findings in context, instructed to report only new/unaddressed issues.

**Domain takeaway:** scope decides which file wins; skills package on-demand workflow with isolation; explore before committing; examples and tests drive iteration; independent reviewers beat self-review.

---

# Domain 3 · Tool Design & MCP Integration (18%)

## 3.1 Tool interfaces: descriptions are the selection mechanism

**What it means.** The model picks tools by reading descriptions — nothing else.

- Good descriptions: purpose, input format, example queries, edge cases, when to use vs neighbors.
- **Classic case:** `analyze_content` vs `analyze_document` with near-identical descriptions — misrouting. Fix: rename to `extract_web_results` with a web-specific description.
- Split generic tools into purpose-specific ones with I/O contracts: `analyze_document` → `extract_data_points` + `summarize_content` + `verify_claim_against_source`.
- System-prompt keyword sensitivity creates unintended tool associations — review prompts for it.

## 3.2 Structured error responses

**What it means.** A failing tool's job is reporting failure in a form the agent can act on (MCP `isError` + classified payload).

- **Error classes:** transient (timeout, unavailable) · validation (bad input) · business/policy ("no refunds after 90 days") · permission.
- Return `errorCategory`, `isRetryable`, human-readable description. Policy violations: `retriable: false` + customer-friendly wording.
- **Anti-patterns:** uniform "Operation failed"; server-side logging with nothing returned; silent internal retries; raw exception text with just an isError flag.
- **Access failure ≠ valid empty result** — a timeout is a retry decision; "no matches" is a successful query with zero rows.
- Subagents recover transient failures locally; propagate only the unresolvable, with partial results and attempts.

> **Repo:** `check_quota()` returns `(can_proceed, message, units_remaining)` — a result, not a crash. The provider layer classifies: 429 → backoff; 400 invalid key → clean abort (retrying a dead key just burns quota — the recovery skill bans test re-runs for exactly this).

## 3.3 Tool distribution & tool_choice

**What it means.** More tools ≠ more capable. 18 where 4–5 would do degrades selection; agents misuse tools outside their specialty.

- Scope toolsets per role; constrained alternatives (`fetch_url` → `load_document` that validates URLs); scoped cross-role tools for high-frequency needs (`verify_fact` for a synthesis agent).
- `tool_choice`: `auto` (may answer in text), `any` (must call some tool), forced `{"type":"tool","name":"…"}` (must call that tool — e.g., force `extract_metadata` first, continue after).

## 3.4 MCP servers in Claude Code & workflows

**What it means.** MCP servers are configured per scope, discovered at connection, tools available simultaneously.

- Project `.mcp.json` (team, VCS-shared) vs user `~/.claude.json` (personal/experimental).
- Env expansion `${GITHUB_TOKEN}` keeps credentials out of committed files.
- **MCP resources** expose content catalogs (issue summaries, doc trees, DB schemas) — visibility without exploratory calls.
- Prefer community servers for standard integrations (Jira, GitHub); custom only for team-specific workflows.
- Enhance MCP tool descriptions so they win over weaker built-ins.

## 3.5 Built-in tools: Grep vs Glob vs Read/Write/Edit

- **Grep** searches *content* (callers, error strings); **Glob** matches *paths* (`**/*.test.tsx`). Don't cross them.
- **Edit** needs unique anchor text; when not unique → Read + Write fallback.
- Build understanding incrementally: Grep entry points → Read imports → trace. Never read everything upfront.

> **Repo:** this guide was built that way — `grep -n 'def '` on `digest.py` to map functions, then reading only the quota section, instead of loading all 704 lines.

**Domain takeaway:** descriptions drive selection, errors are data for decisions, fewer well-scoped tools beat a big toolbox, and access failures are not empty results.

---

# Domain 4 · Prompt Engineering & Structured Output (20%)

## 4.1 Explicit criteria improve precision

**What it means.** Vague instructions produce vague precision. "Check comments are accurate" isn't applicable; "flag only when claimed behavior contradicts actual code" is a categorical test.

- **"Be conservative" / "only high-confidence findings" measurably fail** — moods, not criteria.
- Category-based filtering (report bugs/security; skip style/local patterns) beats confidence-based filtering.
- False positives destroy trust in the accurate categories too; remediate by temporarily disabling the noisy category.
- Severity levels need concrete code examples each, or classification drifts.

> **Repo:** the agent's own reply rules are categorical — quiet when 23:00–08:00 / last check < 30 min / nothing new; speak when an event is < 2h away. Auditable, not vibes.

## 4.2 Few-shot prompting

**What it means.** 2–4 targeted examples — the most effective consistency technique when instructions alone drift.

- Show ambiguous cases **and the reasoning** why one action beat a plausible alternative — that's how judgment generalizes to novel patterns.
- Demonstrate the exact output shape (location, issue, severity, fix).
- Include acceptable-pattern vs genuine-issue examples to cut false positives while keeping generalization.
- For extraction: examples across varied document structures cut hallucination and empty fields.

## 4.3 Structured output via tool use & JSON schemas

**What it means.** The most reliable structured output: define an extraction tool whose input is a JSON schema; read data from the `tool_use` response. Syntax errors become impossible.

- `tool_choice` recall: `auto` / `any` (multiple schemas, unknown doc type) / forced (guarantee a first extraction step).
- **Strict schemas kill syntax errors, not semantic ones** — line items that don't sum to the total pass validation. Semantics need checks (→ 4.4).
- Make fields optional/nullable when the source may lack them — required fields invite fabrication. Enums like `"unclear"` and `"other"` + detail string handle ambiguity and extensibility.
- Pair schemas with format-normalization rules for messy sources.

> **Repo:** research subagents return through an `outputSchema` (JSON Schema) merged programmatically, never parsed from prose.

## 4.4 Validation, retry, feedback loops

- **Retry-with-error-feedback:** original document + failed extraction + specific validation errors — targeted self-correction beats blind re-asks.
- **Retries cannot create missing information.** If the fact lives in a doc you didn't provide, no retry count fixes it — recognize that class early.
- Separate semantic validation (sums, field placement) from schema syntax (already solved by tool use).
- `detected_pattern` fields turn human dismissals into a dataset for false-positive analysis.
- Self-check patterns: `calculated_total` vs `stated_total`; `conflict_detected` booleans.

## 4.5 Batch processing strategies

- Message Batches API: **50% cost reduction, up to 24-hour window, no latency SLA, no tool use mid-request** (can't execute tools inside a batch call).
- Fit: overnight reports, weekly audits, nightly test generation. Not: blocking pre-merge checks.
- Window math: submit every 4h to guarantee a 30h SLA against the 24h window.
- `custom_id` correlates request/response — on failure resubmit **only** failed docs (chunk the oversize ones).
- Refine prompts on a sample first; first-pass success beats resubmission heroics.

## 4.6 Multi-instance & multi-pass review

- **Self-review is structurally weak** — the generating session carries its reasoning and won't question itself. An independent fresh-context instance catches more than self-review instructions or extended thinking.
- Multi-pass: per-file local passes + cross-file integration passes (data flow, API consistency). One giant pass dilutes attention and breeds contradictions.
- Verification passes with self-reported confidence per finding enable calibrated routing.

**Domain takeaway:** criteria not moods; examples not adjectives; schemas not prose; retries only when the information exists; independence over introspection.

---

# Domain 5 · Context Management & Reliability (15%)

## 5.1 Preserve critical information

**What it means.** Long conversations leak facts three ways: summarization launders specifics into vagueness; "lost in the middle" drops mid-context content; tool results accumulate at full verbosity.

- **Case-facts block:** amounts, dates, order numbers, statuses in a persistent block in every prompt — *outside* summarized history. Numbers never survive summarization; keep them verbatim.
- **Lost in the middle:** beginnings and ends are used reliably; middles drop. Mitigate: key-findings at top, explicit section headers, restate criticals at the end.
- Trim tool outputs to relevant fields (40-field order lookups when 5 matter eat budget).
- Subagents return metadata (dates, source locations, methodology), not bare prose; upstream agents return structured data, not reasoning chains.

> **Repo:** the drift incident (memory/2026-09-22.md) — a summary ("running all night") destroyed timestamps and cost ~12h of clock error. Fix: trust only the timestamped envelope; `date -u` before judging. Daily notes keep numbers verbatim for exactly this reason.

## 5.2 Escalation & ambiguity resolution

- **Legitimate triggers:** explicit human request; policy ambiguous or silent on the request; no meaningful progress possible.
- **Illegitimate triggers:** sentiment analysis, self-reported confidence — unreliable proxies for complexity.
- Explicit human request → escalate immediately; don't investigate first. Offer-to-resolve only when genuinely handleable — and escalate on reiteration.
- Multiple customer matches → ask for identifiers; never heuristic-pick (wrong-account actions beat one more question in badness only).

> **Repo:** the digest-outage rule — owner-side fixes reported once per outage, repeated only on change. Explicit triggers, no sentiment spam.

## 5.3 Error propagation across agents

- Structured error context: failure type + what was attempted + partial results + alternatives. "Search unavailable" enables nothing.
- **Access failure ≠ valid empty result** (they really test this twice).
- Both extremes are anti-patterns: silently swallowing errors (empty-as-success) and killing the whole workflow on one subagent failure.
- Local recovery first; propagate the unresolvable with attempt logs.
- Synthesis carries coverage annotations: well-supported findings vs gaps from unavailable sources.

> **Repo:** the recovery skill's failure table — 400 / 429 / 402 / memory-failure each with type, fix path, owner. Partial results survive degraded runs (link-only digest > no digest).

## 5.4 Context in large codebase exploration

- Extended sessions degrade — "typical patterns" replace the specific classes found earlier.
- **Scratchpad files** persist findings across context boundaries.
- **Subagent delegation** for verbose discovery; main agent keeps only high-level understanding.
- **Crash recovery:** each agent exports state to a known location; the coordinator loads a manifest on resume.

> **Repo:** daily notes + the digest ledger are this pattern — every automation writes state to disk (last run, sent IDs), so any future session resumes from facts, not memory.

**Domain takeaway:** facts live in persistent blocks and files, not summaries; escalations fire on explicit triggers; errors travel with context; exploration writes its findings down as it goes.

---

# Repo artifact index — go read the code

| Path | What it demonstrates |
|---|---|
| `skills/youtube-digest/scripts/digest.py` | Multi-step pipeline; quota gate (l.93); error classification (l.139–194); idempotent ledger |
| `skills/youtube-digest/state/` | Durable state: quota usage, sent-video ledger |
| `~/.openclaw/agents/main/agent/workshop-skills/youtube-digest-recovery/SKILL.md` | Trigger-first description, failure taxonomy, gates, verification |
| `~/.openclaw/agents/main/agent/workshop-skills/memory-index-recovery/SKILL.md` | Lock quarantining + rebuild rules |
| `documentation/youtube-digest/digest/` | Output format used by the worked-example pattern |
| `AGENTS.md`, `SOUL.md`, `USER.md` | Layered, scoped, budgeted context files |
| `memory/*.md` | Daily logs — numbers verbatim, correction patterns |

## Where to prep next

- **Official:** Anthropic Partner Academy — Architect Foundations path (anthropic-partners.skilljar.com)
- **Practice:** claudecertifiedarchitects.com (400 scenario Qs) · certsafari.com (480 Qs + exam-experience reports) · Matthew Purcell's CCAR-F set
- **Book:** Pearson VUE — pearsonvue.com/us/en/anthropic.html
