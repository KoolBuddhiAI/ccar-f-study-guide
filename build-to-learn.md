# Build-to-Learn Portfolio
## Agent systems that solve real company problems *and* exercise exactly what CCAR-F validates

*Written 2026-10-16 by the workspace agent. The exam blueprint below is from the official Exam Guide v1.0 (first-hand). The projects are proposals, not endorsements — pick by value, learn by domain coverage.*

---

## The rule that makes this work

The exam tests **judgement about tradeoffs**, not API recall. So each project here must be built so the tradeoffs actually appear:

- A tool that's just a CRUD form over a database teaches nothing — there's no loop, no tool selection problem, no context pressure.
- A tool that's an **agent system** forces every exam domain to show up: loops that must terminate correctly, tools that must be described so the model picks right, subagents that start blank, context that must be curated, output that must be schema-valid, failures that must be classified, and humans that must be escalated to at the right moment.

**Anti-goals** (if your build has these, it's not exam practice): a deterministic workflow with no model decisions; a single prompt with no tools; tools with one-line descriptions; no error taxonomy; no verification step; no escalation path; no persistent state.

**A build counts as exam practice when you can point at:** the `stop_reason` loop · a programmatic gate · a structured error with `errorCategory`/`isRetryable` · a coordinator + isolated subagent · a JSON-schema output · a validation-retry · a case-facts block · an explicit escalation trigger. That's the checklist for every project below.

---

# Track A — Operations agents (internal, high value, broad coverage)

## A1 · IT request & licence provisioning agent
**The problem.** Developers ask for hardware, software, licences, and IT support through tickets and chat. Provisioning is manual, policy lives in people's heads, and approvals stall.

**System shape.** An agent with MCP tools: `get_employee`, `check_inventory`, `get_licence_eligibility`, `raise_procurement`, `assign_licence`, `open_support_ticket`, `escalate_to_human`. Requests arrive in natural language; the agent decomposes multi-part asks ("laptop + IDE licence + VPN access"), checks entitlements, and acts within policy.

**The exam concepts it forces.**
- **D1 (27%)**: agentic loop; decomposition of multi-concern requests; a **programmatic gate** — no `raise_procurement` above a cost threshold until `get_licence_eligibility` returned an approved code. Prompt instructions would leak; a gate can't.
- **D2 (18%)**: descriptions that separate `check_inventory` from `get_licence_eligibility` (deliberately similar — that's the exercise); structured errors: out-of-stock is *transient* (retry later), "role not entitled" is *business* (`isRetryable: false`, explain to the user), "budget frozen" is *permission*.
- **D5 (15%)**: request facts block (employee ID, entitlements verified, cost, approval state) kept outside summarised history; escalation on policy silence ("no policy covers contractor laptops").

**Mirrors official scenario 1** (Customer Support Resolution Agent) — the closest possible transfer: same tools-plus-gates skeleton, different domain.

**Value.** Ticket deflection, deterministic policy enforcement, and an audit trail of who approved what.
**Lab done-when.** A multi-part request is decomposed and answered in one reply; a non-retryable business error produces zero retries and one clear explanation; an out-of-threshold request cannot reach procurement without the eligibility tool's output.

---

## A2 · ISO 27001 continuous-evidence agent ⭐ *flagship*
**The problem.** The ISO 27001 audit needs evidence that controls operate — currently a scramble of screenshots, tickets, and spreadsheets pulled together in the weeks before. Evidence is stale, ownership is unclear, and gaps are found during the audit instead of before it.

**System shape.** A coordinator with specialised collectors. Each collector owns a control family (access control, change management, supplier management, incident management, backup, logging…). Collectors pull from systems via MCP (repo/CI, ticketing, cloud, HR/access system, backup console), extract structured evidence from documents where there's no API, and write to an **evidence register** with: control ID, statement of what it proves, source, timestamp, freshness window, owner, and hash of the artefact. A verifier subagent re-checks each item against the control's acceptance criteria. A report writer produces the audit pack. Scheduled runs keep the register warm; humans sign off what machines can't.

**The exam concepts it forces.**
- **D1**: coordinator–subagent with **isolated** subagents (a collector must be handed its control family and its access scope in the prompt — it inherits nothing); parallel Task calls per run; iterative re-delegation when the verifier finds a gap; **programmatic gates** so nothing is marked "evidenced" without a verifier pass.
- **D2**: evidence-collection tools with tight scopes; error taxonomy matters enormously — "source unreachable" (transient) vs "control owner hasn't signed" (business, escalate) vs "no access to the HR system" (permission). Also: prefer existing controls' APIs over scraping.
- **D4 (20%)**: extracting evidence from unstructured artefacts (policies, meeting minutes, tickets) needs JSON schemas with nullable fields (some controls have no artefact, and *null must be allowed* or the model invents proof), enum `"other"`+detail for control variants, and validation-retry when an extraction misses its required fields. Batch-run document extraction overnight.
- **D5 (15%)**: this is *literally* the case-facts pattern — transactional facts (control ID, date, owner, status) outside summarised history; coverage annotations for controls with no evidence yet; **freshness** as a first-class field because "evidenced last March" is the failure mode that fails audits.
- **Escalation**: gaps and policy questions go to the control owner with a structured handoff (control, what's missing, what's needed, by when) — the exam's handoff pattern, on real work.

**Extension to end users / customers.** The register becomes a **trust portal / security-questionnaire concierge**: sales and customers ask for evidence; an agent drafts answers citing specific register entries and flags anything it can't support. This is the internal tool paying for itself twice.

**Value.** Audit prep goes from a pre-audit scramble to continuous readiness; findings caught by your own verifier instead of the auditor; one register serving audits, customer questionnaires, and internal assurance.
**Lab done-when.** Three collectors run in parallel and merge into one register; a deliberately removed evidence item surfaces as a coverage gap with an owner and a deadline; a "null required field" case returns null rather than an invented date; the audit pack lists every claim with its source and freshness.

---

## A3 · Cryptographic key & certificate lifecycle agent
**The problem.** Keys, certificates, and secrets live across cloud accounts, services, and appliances. Expiry surprises cause outages; rotation policy compliance is unverifiable; audit wants proof.

**System shape.** A scheduled agent that inventories keys/certs/secrets, checks each against policy (algorithm, key length, rotation age, storage location, owner), forecasts expiries, and either files remediation tickets or escalates. A normalisation hook redacts secret material before it ever re-enters the model's context.

**The exam concepts it forces.**
- **D1**: scheduled agentic runs; termination on a well-defined completion state.
- **D2**: tool design around *inventory vs policy check vs remediation action* — three different tools that must not be confused; error categories that distinguish "cloud API throttled" (transient) from "no permission to list certs in account X" (permission) from "policy says rotation overdue" (business, raise ticket).
- **D5 / safety**: redaction and normalisation **before context** — the hooks pattern (D1.5) with a compliance reason to exist; never let key material into a transcript, a log, or a report.

**Value.** Prevents expiry outages; produces the rotation-compliance evidence ISO 27001 A.10 and SOC 2 want, continuously.
**Lab done-when.** The inventory run redacts all secret material (grep the transcript for it — zero hits); expired/throttled/permission errors each take their correct path; a certificate expiring in 21 days produces exactly one remediation ticket, not two.

---

## A4 · Payroll & payslip Q&A agent
**The problem.** Payroll questions (payslip line items, deductions, leave balances, pay dates) consume HR time; answers must be accurate and private.

**System shape.** A deliberately **read-only** agent: `get_payslip`, `get_pay_policy`, `get_leave_balance`, `raise_payroll_query`, `escalate_to_human`. Identity verification is a hard gate before any pay data is returned. It explains, it never mutates.

**The exam concepts it forces.**
- **D1/D2**: verification gate — no `get_payslip` until identity is verified, enforced in code; business errors ("no payslip for that period") explained rather than retried.
- **D5**: PII discipline — minimal facts in context, no payslip dumps into history, escalation for anything policy doesn't cover (disputes, adjustments, statutory queries). This is the exam's "deterministic compliance" case with real consequences.
- **D4**: numeric accuracy — the semantic-validation pattern (extract `stated_net` vs `calculated_net` and flag mismatches) rather than trusting a single extraction.

**Value.** Deflects repetitive HR questions; keeps sensitive data out of general channels; produces an escalation trail.
**Lab done-when.** Unverified identity cannot obtain pay data (test it adversarially); a disputed deduction routes to a human with a structured handoff; no payslip raw values appear in the agent's persisted history.

---

## A5 · People-ops cycle agent (reviews, inductions, onboarding)
**The problem.** Review cycles, inductions, probation checkpoints, and onboarding steps depend on someone remembering to chase. Completion state lives in spreadsheets.

**System shape.** A stateful coordinator: `get_review_cycle`, `list_due_participants`, `send_reminder`, `book_session`, `record_completion`, plus per-role induction plans generated from a template + role data. It nudges owners, tracks completion, escalates overdue items.

**The exam concepts it forces.**
- **D1**: coordinator + task state; **idempotency** (a reminder must not fire twice for the same week — the ledger pattern); escalation on overdue.
- **D3 (20%)**: workflow configuration — this is the natural home for a project skill/command with `allowed-tools` restricted to scheduling and notification tools, so a company-wide process can't go rogue.
- **D5**: durable state (scratchpad/manifest) so a restarted run resumes rather than double-nudging; case facts for each employee's cycle position.

**Value.** Predictable people ops with zero spreadsheet custody.
**Lab done-when.** Running the job twice in a row produces zero duplicate reminders; a restart mid-run resumes from the manifest; an overdue review escalates once, with the participant's cycle facts attached.

---

## A6 · L&D enablement agent
**The problem.** Skill gaps are known but learning is generic; nobody maps role + project needs to a path, and completion isn't tracked.

**System shape.** Generate personalised learning paths from role profile, current projects, and skill inventory; curate resources; build quizzes; evaluate submissions with a rubric; track progress and re-plan.

**The exam concepts it forces.**
- **D4 (20%)**: path generation is a structured-output problem — JSON schema (modules, prerequisites, hours, assessment), nullable fields where data is thin, few-shot examples for tone/level variation, batch generation across the cohort, and **rubric-based evaluation** (categorical criteria, not "be fair").
- **D3**: the whole thing is a packaged skill/workflow with progressive disclosure.
- **D5**: cohort progress as persistent state; per-learner facts block.

**Bonus.** This one can be pointed at CCA prep itself — question generation, spaced repetition, and weak-domain drilling from your practice results. The tool becomes your study system.

**Value.** Real upskilling pipeline; measurable progress.
**Lab done-when.** Paths validate against the schema 100% of runs; a learner with sparse data gets nulls, not invented experience; an evaluation of a weak submission cites the rubric criteria it failed.

---

# Track B — Customer-facing (where the portfolio pays for itself)

## B1 · Security-questionnaire & trust-portal concierge
**The problem.** Every enterprise deal drags a security questionnaire. Answering takes senior time, answers drift from reality, and evidence is scattered.

**System shape.** The customer-facing face of A2. An agent ingests a questionnaire (or a customer's trust page request), retrieves register evidence, drafts each answer with citations to control evidence + freshness, and routes unanswerable items to a human with a structured handoff. Optionally publishes an approved subset to a trust portal.

**Exam concepts.** **D1** (loop + coordinator over evidence retrieval), **D4** (schema for answer records: question ID, answer, evidence refs, confidence, status), **D5** (coverage annotations: "no evidence held for this claim" — never a confident fabrication; escalation triggers when policy is silent).
**Value.** Sales cycle compression, consistent answers, audit-grade traceability. This is usually the highest-ROI customer-facing agent a software company can build.
**Lab done-when.** Every drafted answer links to a register entry or is explicitly marked unsupported; a deliberately unsupported question escalates instead of being answered; re-answering the same questionnaire after a systems change updates the answers instead of duplicating them.

---

## B2 · Product support resolution agent
**The problem.** Your own product's support queue: the exam's scenario 1, on your real product.

**System shape.** `get_customer`, `lookup_subscription`, `search_docs`, `open_ticket`, `issue_credit` (gated), `escalate_to_human`. Target first-contact resolution with calibrated escalation.

**Exam concepts.** **D1/D2/D5** in their purest form; and it gives you a live feedback loop for the exam's exact tradeoffs: when to gate, when to describe better, when to escalate. Measure: first-contact resolution, escalation precision, retry rate on non-retryable errors.
**Value.** Deflection, CSAT, and the tightest possible exam rehearsal.
**Lab done-when.** You can show the escalation calibration improving after adding explicit criteria + few-shot examples (mirroring the official sample question's lesson), with before/after numbers.

---

# Track C — Engineering workflow (the build systems around it)

## C1 · CI review & test-generation agent
**The problem.** Reviews are inconsistent; test coverage gaps are found late; review noise erodes trust.

**System shape.** CI job invoking Claude non-interactively with a JSON schema for findings; categorical review criteria; per-file local passes plus a cross-file integration pass; prior-findings context so re-runs report only new or unaddressed items; separate independent instance for high-risk PRs.

**Exam concepts.** **D3 (20%)**: `-p` non-interactive mode, `--output-format json` + `--json-schema`, CLAUDE.md carrying the team's review criteria, session isolation for the reviewer. **D4**: explicit criteria over "be conservative", per-file + integration multi-pass, false-positive quarantine. **D5**: prior findings in context; attention-dilution control.
**Value.** Real CI improvement + direct rehearsal of official scenarios 2 and 5.
**Lab done-when.** A 14-file PR gets consistent per-file depth plus a cross-file pass; a re-run on the same commit posts no duplicates; you can show precision improving after replacing a vague instruction with categorical criteria.

---

## C2 · Contract & vendor-document extraction pipeline
**The problem.** SOWs, DPAs, vendor agreements, and licence terms hold obligations, dates, SLAs, and liability caps that nobody re-reads. Renewals surprise you; obligations go unowned.

**System shape.** Batch extraction into a structured obligations register: tool with a JSON schema (parties, dates, renewal terms, obligations, SLAs, caps, termination rights), nullable fields for absent clauses, validation-retry with error feedback, `calculated_total`-style checks (e.g., notice period vs renewal date consistency), field-level confidence routing to legal review.

**Exam concepts.** **D4 (20%)**: schema design, nullable fields, enum `"other"`+detail for unusual clauses, retry-with-feedback, overnight **Batch API** economics (50% cost, 24-hour window, no tool use mid-request), confidence routing. **D5**: the register *is* the case-facts block; coverage annotations for documents that failed extraction.
**Value.** Renewal control, obligation ownership, procurement leverage. Mirrors official scenario 6 exactly.
**Lab done-when.** A document with a missing clause yields null rather than a fabricated term; a failed subset is resubmitted by `custom_id` and reconciled without reprocessing successes; a notice-period/renewal-date inconsistency is flagged.

---

# Coverage matrix — pick to cover the blueprint

| Project | D1 27% | D2 18% | D3 20% | D4 20% | D5 15% | Official scenario mirror |
|---|---|---|---|---|---|---|
| A1 IT request agent | ●●● | ●●● | ○ | ○ | ●● | 1 |
| A2 ISO 27001 evidence ⭐ | ●●● | ●●● | ○ | ●● | ●●● | 1 + 3 |
| A3 Key lifecycle agent | ●● | ●●● | ○ | ○ | ●●● | 4 |
| A4 Payroll Q&A | ●● | ●● | ○ | ● | ●●● | 1 |
| A5 People-ops cycles | ●●● | ●● | ●●● | ○ | ●●● | 2 |
| A6 L&D enablement | ●● | ○ | ●●● | ●●● | ●● | 6 |
| B1 Questionnaire concierge | ●●● | ●● | ○ | ●●● | ●●● | 3 |
| B2 Product support | ●●● | ●●● | ○ | ○ | ●●● | 1 |
| C1 CI review | ●● | ○ | ●●● | ●●● | ●● | 2 + 5 |
| C2 Contract extraction | ● | ● | ○ | ●●● | ●●● | 6 |

**Minimum path to full coverage:** **A2 + C1 + C2 + A1.** Those four hit every domain at depth (and between them mirror all six official scenarios). Add **B1** and the same work serves customers.

**Suggested order (value-first):**
1. **A2** — you have an audit pressing, and it's the broadest coverage in one build.
2. **A1** — fastest to working prototype, tightest mirror of scenario 1.
3. **C1** — real CI value, covers D3.
4. **C2** — covers the structured-output half of D4 properly.
5. **B1** — extends A2 to customers once the register is trustworthy.

---

# A 4-week build-and-revise rhythm

Each week: build a slice, then *interrogate it with exam questions*. Writing questions about your own system is the highest-yield revision there is — you can't fake understanding of your own tradeoffs.

| Week | Build | Revision pairing |
|---|---|---|
| 1 | A1 loop + tools + gates + error taxonomy | Practice lab P1 (billing disputes) + D1/D2 deep dives |
| 2 | A2 collectors + register + verifier | Practice lab P3 (research) + D5 deep dive; write 10 questions about your own coverage gaps |
| 3 | C1 CI review + schema output | Practice lab P5 + D3/D4 deep dives |
| 4 | C2 extraction + batches | Practice lab P6 + D4 schema/nullable drills |

**Weekly proof-of-theory ritual (30 min):** pick one tradeoff you made, write it as an exam question with four options, and write the explanation of why the three wrong options fail. If you can't articulate why they fail, you've found your next reading.

---

## Why this beats more reading

Every exam domain is a *judgement* domain. Reading tells you that programmatic gates beat prompt instructions; building A1 makes you feel it the first time the model ignores your perfectly worded instruction at 2am on a Friday request. Reading tells you subagents don't inherit context; A2 makes you debug a collector that "knows nothing" — and you never forget it again. Reading tells you nulls must be permitted or the model fabricates; C2 shows you a fabricated renewal date in your own register.

That's the difference between passing an exam and being the person who should pass it.

---

*Official blueprint and scenario names: CCAR-F Exam Guide v1.0. Projects A1–C2 are proposals by the workspace agent, 2026-10-16.*
