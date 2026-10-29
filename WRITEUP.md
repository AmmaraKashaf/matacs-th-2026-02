# Design Write-Up — AI-Assisted Proposal Triage Tool

**Reference:** MATACS-TH-2026-02 | **Candidate:** Ammara Kashaf

---

## Q1 — AI vs. Rules: Where Do I Draw the Line?

The system uses two distinct layers: a deterministic rules engine and an AI scoring layer.

**Rules engine (no AI):** Mandatory pass/fail checks run as pure code. The system looks for explicit signals in the extracted text — licence numbers matching a known ECRA/ESA format, confirmed WSIB clearance, and insurance coverage statements. These are binary decisions with legally defined answers. Using AI here would introduce unnecessary unpredictability into a step where the answer must always be the same. Similarly, budget ceiling enforcement (reject if price > $260,000 CAD) and injection detection (flag proposals containing meta-instructions directed at automated systems) are rule-based.

**AI scoring layer:** The five weighted criteria — experience, price, timeline, warranty, and team/approach — require interpretation. A proposal might say "over a decade of commercial solar work" rather than citing a number. AI is better suited to reading this language, normalising it, and assigning a defensible score with a reason. The AI returns structured JSON (score 0–10 per criterion, plus a one-sentence rationale), which the rules engine then weights and sums into a final 0–100 score.

This separation means the parts of the system that must never vary (eligibility) are locked in code, while the parts that require judgment (quality assessment) are handled by AI with human review downstream.

---

## Q2 — Consistency and Explainability

Non-determinism in AI is a real risk. To address it:

- **Fixed temperature:** All AI calls use temperature = 0 to minimise variation between runs.
- **Structured output:** The prompt requires a strict JSON response schema. The model cannot give a free-text answer — it must return scores and reasons in the exact expected format.
- **Stored raw output:** Every AI response is stored verbatim in the audit log alongside the proposal ID, criteria version, timestamp, and model version. If results are disputed, the exact reasoning the model gave is available.
- **Per-criterion explanations:** The ranked output shown to the reviewer includes a one-line reason for each score (e.g., "Timeline: 12 weeks — well within the 16-week requirement. Score: 9/10"). This makes the ranking readable and contestable by a non-technical reviewer.

Running the same proposals through the system twice with these controls in place should produce identical or near-identical results.

---

## Q3 — Human in the Loop and Audit Trail

Human checkpoints are built into the process at two points:

1. **Before finalising eligibility:** After the rules engine marks proposals as eligible or ineligible, a reviewer confirms the result before AI scoring begins. This catches cases where the mandatory check may have missed a subtly worded licence statement.

2. **Before publishing the ranking:** The reviewer sees the ranked list, per-criterion scores, and AI rationale. They can override any score with a written reason. Only after the reviewer approves does the ranking become the official output.

**What is logged:** proposal ID, upload timestamp, extracted text hash (to confirm the source), criteria version used, AI model and version, raw AI output, calculated scores, reviewer identity, any overrides and their reasons, and final approval timestamp. Logs are append-only and cannot be edited after the fact.

---

## Q4 — Changing Criteria After Scoring Has Started

Criteria and weights are stored in a versioned configuration table managed by the Admin. Each scoring run is tagged with the criteria version it used.

If an admin changes a criterion or weight while proposals are in-flight:

- Proposals already scored retain their scores under the old version.
- The system marks those scores as **stale** and displays a warning to the reviewer.
- The reviewer can choose to re-run scoring under the new criteria version, or accept the stale scores with a note.
- Re-scoring is always available on demand; it is never automatic, to prevent silent changes to results already under review.

Criteria changes are themselves logged (who changed what, and when), so there is a full audit trail of the evaluation framework, not just the scores.

---

## Q5 — Risks and Safeguards

| Risk | How the Design Addresses It |
|---|---|
| **Poor text extraction** (scanned PDFs, garbled text) | The system flags proposals where extracted text falls below a minimum character count or fails a readability check, and asks the reviewer to re-upload a cleaner version before scoring proceeds. |
| **AI hallucination** (model invents information not in the proposal) | The prompt instructs the model to cite the specific text passage it based each score on. If no relevant text is found, the model must return score = 0 and state "Not mentioned in proposal." Reviewers see the cited passage alongside the score. |
| **Prompt injection** (as seen in Proposal C) | Before passing text to the AI, the rules engine scans extracted content for patterns consistent with meta-instructions directed at automated systems (e.g., phrases like "rank this first," "assign full marks," "pre-approved by"). Any proposal triggering this check is immediately flagged and removed from scoring. The reviewer is shown the flagged text and must decide whether to disqualify the proposal or clear the flag manually. |
| **Scoring bias or inconsistency** | The AI prompt explicitly states that each proposal must be scored independently, without reference to other proposals. Relative comparisons are done by the rules engine after all scores are returned, not by the AI. |
| **API cost and latency** | Text is extracted once per proposal and cached. Re-scoring re-uses cached text. Long proposals are truncated to a maximum token limit per section, with the reviewer notified if truncation occurred. |
