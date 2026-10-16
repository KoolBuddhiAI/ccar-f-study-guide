# M365 Discovery Prompt
## Find customer use cases hiding in your communications

*Written 2026-10-16. A prompt for Claude with Microsoft 365 access (Outlook, Teams, SharePoint/OneDrive), to mine recent customer communications for candidate automation and agent use cases. Two versions: a compact one you can paste straight in, and a full one with output schema and guardrails.*

---

## What this does

It turns scattered signals — repeated manual work, approval stalls, evidence chasing, re-keying between systems — into a ranked, evidence-backed shortlist of use cases you could sell or build. The design is deliberately exam-shaped: **explicit categorical criteria** (not "find interesting things"), **few-shot calibration** of ambiguity, **structurally separated evidence and metadata**, and **coverage gap annotations** instead of confident invention. Using it is itself D4 practice.

---

## Compact version — paste this

```
You are analysing Microsoft 365 communications to surface candidate AI/agent use cases for a software company that builds solutions for its customers.

Scope: [DATE RANGE, e.g. last 6 months] · [MAILBOXES/TEAMS/SP SITES] · [CUSTOMER(S) or ALL].

Include a signal only if it shows ONE OR MORE of:
- the same manual, repetitive task appearing in 3+ separate messages
- work stalling on approvals, access, or missing information
- data re-entered or copied between systems
- documents/evidence chased for audits, compliance, or customer assurance
- knowledge hunted for repeatedly (same question asked by different people)
- handoffs where context is lost between teams or tools
- reports/status updates assembled by hand
- onboarding or scheduling toil

Exclude: one-off incidents, personnel/HR matters, personal data unrelated to work processes, vendor marketing, and anything you cannot cite.

For every candidate, output:
1. Customer/team · 2. Use case name (one line) · 3. Problem statement (2 sentences) ·
4. Evidence: at least 2 citations — source (mail/chat/doc name), date, short quote or close paraphrase ·
5. Frequency estimate + how you derived it · 6. Current workaround · 7. Impact if solved (time/cost/risk) ·
8. Proposed solution shape: agent / tool / automation / dashboard — and which systems it must touch ·
9. Data sensitivity + any consent concerns · 10. Confidence: high/medium/low + what would raise it ·
11. Coverage gap: what you could NOT see that might change this.

Rules: never fabricate a citation; if evidence is thin, say so and mark low confidence. Do not quote personal, financial, or health data — paraphrase and mark the category instead. Aggregate across people rather than naming individuals. Output first as a JSON array using exactly those 11 fields, then as a ranked markdown table (impact × frequency × confidence).

Rank by: business value × evidence strength × feasibility. End with the 3 recommendations you'd take to a customer meeting, each with the single strongest piece of evidence.
```

---

## Full version — for a serious run

Use this when you want reliable, repeatable output across a large mailbox.

```
ROLE
You are a solution-discovery analyst for a software company that designs AI and agent systems for customers. You read Microsoft 365 communications (Outlook, Teams, SharePoint/OneDrive) to find recurring operational friction that AI could remove.

GOAL
Produce a ranked, evidence-backed shortlist of candidate use cases. Quality bar: every claim traceable to a citation a human could open and verify.

SCOPE
- Window: [e.g. 2025-04-01 to 2025-10-01]
- Sources: [mailboxes / Teams channels / SharePoint sites]
- Customers: [names, or "all"]
- Batch: work customer by customer, then merge; do not mix evidence across customers in the per-customer section.

INCLUSION CRITERIA (a signal must match at least one)
1. Repetition — same manual task in 3+ separate messages or threads.
2. Bottleneck — work waiting on approvals, access, or missing information.
3. Re-entry — the same data typed, copied, or reconciled across systems.
4. Evidence chase — pulling proof for audits, compliance, or customer assurance.
5. Knowledge hunt — the same question asked repeatedly by different people.
6. Handoff loss — context dropped between teams, tools, or shifts.
7. Manual reporting — status updates or reports assembled by hand.
8. Lifecycle toil — onboarding, offboarding, scheduling, cyclical reviews.

EXCLUSION CRITERIA
- One-off incidents with no pattern.
- Personnel, disciplinary, health, or personal matters (exclude outright).
- Anything you cannot cite to a specific artefact.
- Marketing newsletter noise, calendar logistics with no process content.

METHOD
1. Work in time-sliced batches (e.g. per month) to keep attention consistent; note any period you could not read.
2. Within each batch, collect candidate signals with citations before evaluating anything — collection and evaluation are different passes.
3. Deduplicate across threads: the same problem seen five times is ONE use case with five citations, not five use cases.
4. Score each candidate: Impact (1-5), Frequency (1-5), Evidence strength (1-5), Feasibility (1-5).
5. Rank by Impact × Frequency × Evidence, then sanity-check Feasibility separately.
6. Write a cross-customer theme section: problems appearing for 3+ customers are the strongest commercial signals.

EVIDENCE RULES
- Every candidate needs at least two citations: source type, name, date, and a short quote or faithful paraphrase.
- Never invent a citation, date, or quantity. If you are inferring, label it "inference" and say what it rests on.
- Do not reproduce personal, financial, or health data. Refer to categories ("a payslip query") rather than values.
- Prefer aggregated language: "three people in the same team asked" beats naming one person.
- Where data is missing, record it as a coverage gap rather than estimating silently.

OUTPUT — PART 1: JSON
Return a JSON array; each object has exactly these fields:
{
  "customer_or_team": "",
  "use_case": "",
  "problem_statement": "",
  "evidence": [{"source_type":"mail|chat|doc|meeting","name":"","date":"","quote_or_paraphrase":""}],
  "frequency": {"estimate":"","basis":""},
  "current_workaround": "",
  "impact": {"time":"","cost":"","risk":"","score_1_5":0},
  "proposed_shape": {"type":"agent|tool|automation|dashboard|report","systems_touched":[],"why_this_shape":""},
  "data_sensitivity": "",
  "consent_or_privacy_notes": "",
  "confidence": {"level":"high|medium|low","what_would_raise_it":""},
  "coverage_gaps": "",
  "scores": {"frequency_1_5":0,"evidence_1_5":0,"feasibility_1_5":0,"total":0}
}

OUTPUT — PART 2: ranked markdown table
Columns: Rank · Customer/Team · Use case · Impact · Frequency · Evidence · Score · Strongest evidence (one line).

OUTPUT — PART 3: themes
Cross-customer patterns worth productising, each with the customers involved and why the pattern matters commercially.

OUTPUT — PART 4: what I could not see
Coverage gaps: periods, mailboxes, or doc types not read; assumptions made; questions to ask a human.

TUNING
- If output is vague, re-run that slice with: "cite the specific message for each claim, and drop any candidate with fewer than two citations."
- If you get too many weak candidates, tighten to: "only candidates where the same workaround appears in three or more threads."
- If tone drifts into speculation, add: "label every non-cited statement as inference and state its basis."
- Keep one run to one customer set where possible; merge at the end.
```

---

## Variants

**Single-customer deep dive.** Scope to one customer, add: *"produce a process map of their week: which recurring activities consume the most person-hours, which are most error-prone, and which would a customer most readily pay to have automated?"* This is the pre-sales version — it turns a discovery call into a proposal.

**Questionnaire & assurance mining.** Filter to security/compliance threads only, add: *"extract every question a customer asked about security, compliance, or evidence in the window, group them, and note which recur across customers — these are the requirements for a trust portal or questionnaire-automation product."*

**Renewal & churn signals.** Filter to commercial threads, add: *"identify which friction points customers mentioned in proximity to renewal decisions, and separate product gaps from service gaps."*

**Internal first.** Run it on internal IT/ops channels before customer mailboxes: *"scope: internal IT, procurement, and audit channels only."* Internal friction is easier to verify, and a fixed internal problem is a reference customer you already own.

---

## Using it well

- **Run it where it can cite.** A discovery output with no citations is a guess with better formatting.
- **Do not let it name individuals.** Ask for aggregates; review before sharing anything outside the team.
- **Treat the ranking as a queue, not a verdict.** Take the top three to a human — sales or ops — and ask which matches a budget that already exists.
- **Keep the coverage-gaps section.** The things it couldn't see are usually where the real use case hides; that section is your next discovery question, not a disclaimer.
- **Check policy before pointing it at customer mail.** Consent, retention, and data-residency rules apply; internal-only is the safe default for the first run.

---

## Why the prompt is shaped this way

Not decoration — each technique is load-bearing, and each is exam material:

| Technique in the prompt | Why it's there | Exam concept |
|---|---|---|
| Explicit categorical inclusion criteria | "Find interesting things" produces noise; categories produce consistency | Categorical criteria over vague instructions |
| Few-shot-free calibration via examples in the criteria | The examples *are* the calibration; they generalise to unseen patterns | Few-shot judgement transfer |
| JSON schema with fixed fields | Makes output comparable, rankable, and mergeable | Structured output via schema |
| Separate collection and evaluation passes | Prevents premature filtering that discards weak-but-real signals | Multi-pass review |
| At-least-two-citations rule | Kills single-thread overreaction; forces corroboration | Evidence strength / false positives |
| "Label inference" rule | Separates what was seen from what was concluded | Anti-hallucination discipline |
| Coverage gaps section | The unread period is the most dangerous blind spot | Coverage annotations |
| Retry-with-feedback tuning lines | Fixes drift by feeding back the specific failure | Retry with error feedback |
| Aggregation over naming | Privacy protection that also improves signal quality | Context minimisation |

---

*Companion to the CCAR-F study guide. Written 2026-10-16 by the workspace agent.*
