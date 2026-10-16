# CCAR-F Practice Lab
## Six new practice use cases, built on the official exam scenario bank

*Companion to the CCAR-F study guide. Sources: **official Exam Guide v1.0** (Anthropic, effective July 2026) — scenario bank, preparation exercises, and sample questions are quoted or paraphrased first-hand from it. New practice cases in Part C are written by this workspace agent to extend the official material.*

---

## How the exam uses scenarios

| Fact | Value |
|---|---|
| Items / time | 60 items / 120 minutes |
| Structure | **4 scenarios drawn at random from a bank of 6** |
| Item format | Multiple-choice **and** multiple-response (each item says how many to select) |
| Pass mark | Scaled 720 on a 100–1,000 scale |
| Score report | Pass/fail + **percent-correct by domain** |

**How to practise with this page:** read a scenario, answer cold from memory, then study the explanation — in the official sample questions, the reasoning for *why the wrong options are wrong* is as instructive as the key. Every practice case below follows that format.

---

## Part A · The official scenario bank (what the exam actually shows you)

| # | Scenario | The context | Primary domains |
|---|---|---|---|
| 1 | **Customer Support Resolution Agent** | Agent SDK agent for returns, billing disputes, account issues, using custom MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`). Target: 80%+ first-contact resolution while knowing when to escalate. | D1 Agentic Architecture · D2 Tool Design & MCP · D5 Context & Reliability |
| 2 | **Code Generation with Claude Code** | Claude Code for generation, refactoring, debugging, documentation; custom slash commands, CLAUDE.md configuration, plan mode vs direct execution. | D3 Claude Code Config · D5 Context & Reliability |
| 3 | **Multi-Agent Research System** | Coordinator delegating to search / document-analysis / synthesis / report subagents, producing comprehensive cited reports. | D1 Agentic Architecture · D2 Tool Design & MCP · D5 Context & Reliability |
| 4 | **Developer Productivity with Claude** | Agent helping engineers explore unfamiliar codebases, understand legacy systems, generate boilerplate, automate repetitive tasks — built-in tools (Read, Write, Bash, Grep, Glob) plus MCP servers. | D2 Tool Design & MCP · D3 Claude Code Config · D1 Agentic Architecture |
| 5 | **Claude Code for Continuous Integration** | Claude Code inside CI/CD: automated code review, test generation, PR feedback — with prompts that minimise false positives. | D3 Claude Code Config · D4 Prompt Engineering & Structured Output |
| 6 | **Structured Data Extraction** | Extraction from unstructured documents, JSON-schema validation, high accuracy, graceful edge cases, downstream integration. | D4 Prompt Engineering & Structured Output · D5 Context & Reliability |

**Note the overlap pattern:** D1/D2/D5 cluster around the two agentic scenarios (1 and 3); D3 appears in four of six; D4 concentrates in the CI and extraction scenarios. If a domain feels weak, drill the scenarios that carry it.

---

## Part B · The four official preparation exercises

From the guide's "How to Prepare" section — build these for hands-on reps:

1. **Multi-Tool Agent with Escalation Logic** — 3–4 MCP tools with differentiating descriptions (two of them deliberately similar); agentic loop keyed on `stop_reason`; structured errors (`errorCategory`, `isRetryable`); a programmatic hook enforcing a business rule (e.g. block operations above a threshold); test multi-concern messages. *Reinforces D1, D2, D5.*
2. **Configure Claude Code for a Team Workflow** — project-level CLAUDE.md; `.claude/rules/` with glob frontmatter (`paths: ["src/api/**/*"]`, `paths: ["**/*.test.*"]`); a skill with `context: fork` + `allowed-tools`; MCP server in `.mcp.json` with env-var expansion plus a personal one in `~/.claude.json`; compare plan mode vs direct execution across simple, multi-file, and architecturally-ambiguous tasks. *Reinforces D3, D2.*
3. **Structured Data Extraction Pipeline** — JSON schema with required + optional + nullable fields and an `"other"` + detail enum pattern; validation-retry loop (document + failed extraction + specific error); few-shot examples across varied formats; batch submission with `custom_id` failure handling; field-level confidence routing to humans. *Reinforces D4, D5.*
4. **Multi-Agent Research Pipeline** — coordinator whose `allowedTools` includes `Task`; findings passed directly in each subagent's prompt; parallel Task calls in one response; structured findings (claim, evidence, source, date) preserving attribution; simulated timeout with structured error context and coverage annotations; conflicting sources preserved side by side. *Reinforces D1, D2, D5.*

---

## Part C · Six new practice use cases

Same worlds as the exam, fresh failure modes and decision points. Each has four exam-style items and a build lab.

---

### P1 · Subscription billing disputes
**Family:** support resolution · **Domains:** D1, D2, D5

**Context.** Your support agent handles subscription cancellations, failed renewals, and refund requests through MCP tools: `get_account`, `list_charges`, `issue_credit`, `escalate_to_human`. Instrumentation shows 18% of runs end with the agent retrying `issue_credit` after it returned `{"errorCategory": "business", "isRetryable": false, "reason": "beyond 90-day window"}`. Average handle time has tripled for those cases.

**Q1 (multiple choice).** What is the most effective fix for the retry behaviour?
A. Raise the retry limit for `issue_credit` so transient blips are absorbed.
B. Instruct the agent in the system prompt to stop retrying after a failed credit.
C. Make the business-rule error self-describing in the recovery path: keep `isRetryable: false` in the tool result, and have the agent surface the reason to the customer while offering the policy-compliant alternative.
D. Remove `issue_credit` from the toolset and route every refund through `escalate_to_human`.

**Correct: C.** The result already carries the right classification — the failure is that nothing in the agent's recovery policy uses it. A non-retryable business error should terminate that action and convert to a customer-facing explanation. A and B leave the loop probabilistic; D trades away the 90%-of-cases autonomy the design depends on.

**Q2 (multiple choice).** Logs show `get_account` is called for "why was I charged twice?" but `list_charges` is called for "cancel my plan". Both descriptions read "Retrieves account billing information" / "Retrieves billing records". First step?
A. Add five few-shot examples teaching the routing.
B. Rewrite both descriptions to state what each returns, the inputs it accepts, and when to use it instead of the other.
C. Merge them into one `billing_lookup` tool.
D. Add a keyword router that pre-selects the tool per request type.

**Correct: B.** Descriptions are the primary selection mechanism. B is the proportionate root-cause fix; A adds tokens without fixing the ambiguity, C is a valid architecture change but far more work than a "first step", D is over-engineering.

**Q3 (multiple response — select two).** A customer message contains three concerns: a failed charge, a wrong address on file, and a request to cancel. Which two behaviours should the agent exhibit?
A. Decompose the request and address each concern, then synthesise one unified reply.
B. Process concerns strictly in the order stated, escalating on the first that needs it.
C. Persist the transactional facts (charge IDs, amounts, dates, statuses) outside summarised history.
D. Ask the customer to resend each concern separately.

**Correct: A, C.** Multi-concern decomposition plus a persistent case-facts block is exactly the D1/D5 pairing. B abandons the remaining concerns; D shifts work to the customer.

**Q4 (multiple choice).** The agent is asked for a goodwill credit that policy doesn't cover — policy addresses only documented service outages. The correct behaviour is to:
A. Escalate immediately, because policy is silent on the request.
B. Approve it, since the customer's frustration is evident.
C. Decline and close the conversation.
D. Offer an unrelated discount to avoid escalation.

**Correct: A.** Policy gaps are an explicit escalation trigger. B and D invent policy; C is a dead end when a human could decide.

**Lab P1.** Build the loop: four tools with differentiating descriptions, structured errors carrying `errorCategory` + `isRetryable`, a programmatic gate that blocks `issue_credit` until `get_account` returns a verified ID, and a recovery policy that maps each error class to retry / explain / escalate. *Done when:* a non-retryable business error produces one customer-facing explanation and zero retries; multi-concern tickets get one synthesised reply.

---

### P2 · Onboarding a legacy Java monolith
**Family:** code generation with Claude Code · **Domains:** D3, D5

**Context.** A new engineer must understand a 12-year-old monolith: mixed conventions, tests scattered beside sources, inconsistent logging. You're setting up Claude Code to help — and the codebase-analysis runs are flooding the main session, so by the third run Claude answers about "typical service patterns" instead of this codebase's actual classes.

**Q1 (multiple choice).** Which change addresses the context degradation directly?
A. Add `context: fork` to the analysis skill so it runs isolated and returns only its summary.
B. Instruct the skill to summarise its output more tersely.
C. Run `/compact` after each analysis.
D. Move the skill to the personal `~/.claude/skills/` directory.

**Correct: A.** Isolation keeps verbose exploration out of the main conversation — the designed mechanism for exactly this. B still lands a long report; C treats symptoms after pollution; D changes scope, not isolation.

**Q2 (multiple choice).** Tests live beside sources throughout the tree and must follow one house style. How do you make Claude apply test conventions wherever it generates tests?
A. Put the conventions in the root CLAUDE.md and rely on inference.
B. Add a `.claude/rules/` file with frontmatter `paths: ["**/*.test.*"]`.
C. Create one skill per code area containing its conventions.
D. Add a CLAUDE.md in every subdirectory.

**Correct: B.** Glob-scoped rules load by path regardless of directory — the maintainable answer for cross-cutting conventions. A is unreliable inference; C needs manual invocation; D is unmaintainable across many directories.

**Q3 (multiple response — select two).** Two engineers disagree: one wants the whole codebase read upfront, the other wants incremental exploration. Which two techniques belong in the workflow?
A. Grep for entry points and trace imports outward, rather than reading everything.
B. Keep a scratchpad file recording key findings across sessions.
C. Cache the full source tree in the prompt for the whole session.
D. Disable subagents so all context stays in one conversation.

**Correct: A, B.** Incremental tracing plus durable scratch notes is the D5 pattern for large codebases. C exhausts the budget and triggers lost-in-the-middle; D removes the isolation that keeps the main session clean.

**Q4 (multiple choice).** The team must restructure one module whose service boundaries are unclear, touching dozens of files. Approach?
A. Plan mode first: explore, map dependencies, design the boundary before editing.
B. Direct execution, letting boundaries emerge as you edit.
C. Direct execution with a fully pre-written per-file specification.
D. Start direct and switch to plan mode only if problems appear.

**Correct: A.** Architectural, multi-file, multiple-valid-approaches = plan mode. B discovers dependencies too late; C presumes the answer; D ignores stated complexity.

**Lab P2.** Configure the repo: project CLAUDE.md, two rule files with glob frontmatter, one `context: fork` skill with `allowed-tools` restrictions, and a scratchpad convention. *Done when:* rules load only for matching paths, the skill's verbose output never enters the main session, and a second session can resume from the scratchpad alone.

---

### P3 · Regulatory compliance research
**Family:** multi-agent research · **Domains:** D1, D2, D5

**Context.** A coordinator delegates to search, document-analysis, and synthesis subagents to produce a regulator-facing briefing. A subagent timed out; two credible sources disagree on a key figure; the synthesis must be auditable.

**Q1 (multiple choice).** The synthesis agent needs simple fact-checks (dates, names) during its pass, and currently round-trips through the coordinator for each — adding 40% latency. Best fix?
A. Give synthesis a scoped `verify_fact` tool for simple lookups; complex checks still route via the coordinator.
B. Have synthesis batch all checks to the end of the pass.
C. Give synthesis full web-search capability.
D. Have the search agent pre-cache speculative context around every source.

**Correct: A.** Least privilege for the high-frequency case, coordination preserved for the complex 15%. B creates blocking dependencies; C breaks separation of concerns; D guesses at unknown future needs.

**Q2 (multiple choice).** The document-analysis subagent times out. What should the coordinator receive?
A. A generic "search unavailable" status.
B. An empty result marked successful.
C. Structured error context: failure type, the attempted query, partial results, and alternative approaches.
D. A propagated exception that terminates the run.

**Correct: C.** Structured context is what enables intelligent recovery — retry differently, route around, or proceed with annotated gaps. A hides the signal, B suppresses it, D aborts work that was recoverable.

**Q3 (multiple response — select two).** Two credible sources give different figures for the same statistic. The synthesis output must:
A. Preserve both values with source attribution.
B. Distinguish well-established findings from contested ones.
C. Average the two values into a single number.
D. Drop the statistic to avoid ambiguity.

**Correct: A, B.** Conflicting evidence is preserved and labelled, never silently resolved — the auditability requirement. C invents a number; D deletes information the reader needs.

**Q4 (multiple choice).** Subagents keep duplicating research. Root cause and fix?
A. The coordinator's decomposition is too broad — narrow the subtopic partitioning per subagent.
B. The search agent's queries are too short — lengthen them.
C. The synthesis agent lacks deduplication instructions.
D. The subagents share memory and overwrite each other.

**Correct: A.** Duplication is an assignment problem at the coordinator. B, C treat downstream symptoms; D is false — subagents don't share context.

**Lab P3.** Build the coordinator: `Task` in `allowedTools`, findings passed in-prompt, two parallel Task calls in one response, structured findings (`claim`, `evidence`, `source`, `date`), one simulated timeout, and a conflicting-source case. *Done when:* the report carries citations through synthesis, annotates the timed-out section as a coverage gap, and shows both conflicting figures with attribution.

---

### P4 · Internal code-search assistant
**Family:** developer productivity · **Domains:** D2, D3, D1

**Context.** Engineers ask an agent to find files, trace symbols, and explain modules across a monorepo. It currently has 19 tools, including both `search_code` and `find_files` with overlapping descriptions, and a synthesis step that keeps trying to run searches itself.

**Q1 (multiple choice).** Tool selection is erratic. Which change helps most?
A. Add more tools so every phrasing has a match.
B. Cut to the role's essential tools (~4–5) and rewrite descriptions to state inputs, outputs, and boundaries.
C. Raise the model tier.
D. Add a keyword pre-router.

**Correct: B.** Selection reliability degrades with tool count; descriptions and scoping are the levers. A makes it worse; C doesn't fix ambiguity; D bypasses the model's language understanding.

**Q2 (multiple choice).** Which built-in tool finds all call sites of `parseInvoice`?
A. Glob — it matches names across directories.
B. Grep — content search across the tree.
C. Read — load each candidate file.
D. Bash `ls -R`.

**Correct: B.** Grep searches content (symbols, strings); Glob matches paths. A confuses the two; C and D don't search content.

**Q3 (multiple response — select two).** Which two design moves respect the report-writing agent's role?
A. Restrict its toolset to retrieval and formatting tools.
B. Give it one scoped cross-role tool (`verify_fact`) for high-frequency needs.
C. Grant it the full search toolset for convenience.
D. Let it bypass the coordinator when it needs more data.

**Correct: A, B.** Scoped access plus a narrow, high-frequency exception. C and D destroy the specialisation that made it reliable.

**Q4 (multiple choice).** Exploratory questions trigger dozens of searches and the session becomes slow and vague. Best structural response?
A. Delegate verbose exploration to subagents that return summaries.
B. Increase the model's context window.
C. Ask engineers to phrase questions more precisely.
D. Disable search for complex questions.

**Correct: A.** Isolation keeps the main session lean and specific — the same mechanism as `context: fork`. B doesn't fix attention quality; C shifts burden; D removes the capability.

**Lab P4.** Reduce a bloated toolset: from 19 tools to 5 with rewritten descriptions; replace a generic `fetch_url` with a validating `load_document`; add one scoped cross-role tool. *Done when:* ambiguous requests route correctly in a 10-case spot check, and exploration runs leave the main session answering with real symbol names rather than generic patterns.

---

### P5 · Flaky-test triage in CI
**Family:** Claude Code for CI · **Domains:** D3, D4

**Context.** A CI job asks Claude to review each PR for risky changes and generate missing tests. It hangs in the pipeline, floods PRs with style nitpicks, and re-reports the same issues on every push.

**Q1 (multiple choice).** The job hangs waiting for input. Correct invocation?
A. `claude -p "..."` — print/non-interactive mode.
B. Export `CLAUDE_HEADLESS=true`.
C. Redirect stdin from `/dev/null`.
D. `claude --batch "..."`.

**Correct: A.** `-p` is the documented non-interactive mode: process, print, exit. B and D name non-existent features; C is a Unix workaround that doesn't address the CLI contract.

**Q2 (multiple choice).** Findings must post as inline comments. Which combination produces machine-parseable output?
A. `--output-format json` with `--json-schema`.
B. A firm instruction to "reply only with JSON".
C. Markdown tables parsed with regex.
D. Two passes with a self-check between them.

**Correct: A.** Schema-enforced output is the reliable mechanism. B is probabilistic syntax; C is brittle parsing of prose; D doubles cost without guaranteeing shape.

**Q3 (multiple response — select two).** Reviewers at 60% precision lose trust. Which two changes raise precision most directly?
A. Replace "report high-confidence issues" with categorical criteria for what to report and skip.
B. Temporarily disable the noisiest category while its prompt is reworked.
C. Add an instruction to "be conservative".
D. Raise the temperature.

**Correct: A, B.** Explicit categories and quarantining a noisy class restore trust and target the fix. C is a mood, not criteria — it measurably fails; D increases variance.

**Q4 (multiple choice).** A 14-file PR gets inconsistent reviews — deep on some files, contradictory across identical patterns. Restructure how?
A. Per-file local passes plus a separate cross-file integration pass.
B. Require engineers to split the PR.
C. Use a larger context window on one pass.
D. Run three passes and keep only findings that appear twice.

**Correct: A.** Focused passes fix attention dilution; the integration pass catches cross-file issues. B shifts burden; C confuses window size with attention; D suppresses real but intermittent findings.

**Lab P5.** Build the CI review job: `-p` invocation, `--output-format json` with a schema for `{file, line, severity, issue, suggested_fix}`, categorical review criteria, and prior-findings context so re-runs report only new or still-unaddressed items. *Done when:* the job exits cleanly with no input wait, output parses without post-processing, and a second run on the same commit posts no duplicates.

---

### P6 · Insurance claim document extraction
**Family:** structured data extraction · **Domains:** D4, D5

**Context.** An extraction pipeline processes thousands of claim documents into a downstream system. Fields are sometimes absent; a batch of 1,200 documents runs nightly; totals must reconcile; low-confidence extractions require human review.

**Q1 (multiple choice).** Required schema fields cause the model to invent values when the document lacks the data. Best fix?
A. Add "if unknown, leave blank" to the prompt.
B. Mark such fields optional/nullable in the schema and permit null.
C. Lower the temperature.
D. Retry until the field appears.

**Correct: B.** Nullability in the schema is the structural fix; a prompt request fights the required-field signal. C doesn't change schema pressure; D fabricates faster.

**Q2 (multiple choice).** Line items must sum to the stated total. Schema validation passes but numbers don't reconcile. Which mechanism catches this?
A. Tighten the JSON schema further.
B. Semantic validation: extract `calculated_total` alongside `stated_total` and flag mismatches.
C. Add more few-shot examples.
D. Increase max output tokens.

**Correct: B.** Strict schemas eliminate syntax errors, not semantic ones — reconciliation is a validation-layer concern. A can't express arithmetic; C and D don't check anything.

**Q3 (multiple response — select two).** Which two retry designs are sound?
A. On validation failure, resend the document + failed extraction + the specific error.
B. Retry when the required fact exists only in an external document not provided.
C. Mark fields like `"unclear"` / `"other" + detail` so ambiguity is representable.
D. Raise the batch size to absorb the failure rate.

**Correct: A, C.** Targeted error feedback plus representable ambiguity. B cannot succeed by construction; D multiplies cost without improving extraction.

**Q4 (multiple choice).** The nightly batch of 1,200 documents must finish before the morning shift, and a dozen documents fail. Best handling?
A. Correlate by `custom_id`, resubmit only the failures (chunking anything that exceeded limits).
B. Resubmit the whole batch.
C. Switch everything to synchronous calls.
D. Drop the failures and reconcile later.

**Correct: A.** `custom_id` exists for exactly this; targeted resubmission preserves the 50% batch saving. B wastes the window; C abandons batch economics; D corrupts the downstream totals.

**Lab P6.** Build the pipeline: extraction tool with a JSON schema (required, optional, nullable, `"other"` + detail, `"unclear"`), validation-retry loop with error feedback, `calculated_total` vs `stated_total` reconciliation, batch submission with `custom_id` failure handling, and field-level confidence routing. *Done when:* documents with missing fields yield null rather than invention; a deliberate total mismatch is flagged; a failed subset is resubmitted and reconciled without reprocessing the successes.

---

## Part D · Coverage matrix — where to drill

| Domain | Weight | Scenarios that carry it | Practice units |
|---|---|---|---|
| D1 Agentic Architecture & Orchestration | 27% | 1, 3, 4 | P1, P3, P4, Lab P1, Lab P3 |
| D3 Claude Code Configuration & Workflows | 20% | 2, 4, 5 | P2, P4, P5, Lab P2, Lab P5 |
| D4 Prompt Engineering & Structured Output | 20% | 5, 6 | P5, P6, Lab P5, Lab P6 |
| D2 Tool Design & MCP Integration | 18% | 1, 3, 4 | P1, P3, P4, Lab P1, Lab P4 |
| D5 Context Management & Reliability | 15% | 1, 2, 3, 6 | P2, P3, P6, Lab P2, Lab P3 |

**Routing rule:** if your score report is weak in a domain, do the practice units in that row before touching anything else — the weighted score means D1 errors cost the most.

---

## Part E · Readiness checklist

- [ ] I can state the agentic loop's termination condition and name three tested anti-patterns.
- [ ] I can explain why deterministic ordering needs programmatic gates, not prompt instructions.
- [ ] I can describe subagent context isolation and how the coordinator must compensate.
- [ ] I can classify a tool error as transient / validation / business / permission and say whether it's retryable.
- [ ] I can choose between `tool_choice: auto | any | forced` with a reason.
- [ ] I can place a convention correctly across CLAUDE.md levels, `.claude/rules/`, and skills — and say when each loads.
- [ ] I can pick plan mode vs direct execution for a described task and justify it.
- [ ] I can write a JSON schema with nullable and enum-"other" patterns, and say what it does *not* protect against.
- [ ] I can describe when retry-with-feedback works and when it cannot.
- [ ] I can list the Batch API's economics and its hard constraint on tool use.
- [ ] I can name the legitimate escalation triggers — and the two unreliable proxies.
- [ ] I can explain why an access failure isn't an empty result, and why both must be distinguishable.

Score yourself honestly; anything unchecked is your next practice unit.

---

*Official material: Claude Certified Architect – Foundations Exam Guide v1.0 (effective July 2026). Practice cases P1–P6 written 2026-10-16 by the OpenClaw workspace agent; not endorsed by Anthropic. Verify exam logistics when booking.*
