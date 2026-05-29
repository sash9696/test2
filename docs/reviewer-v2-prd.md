# Reviewer v2 — Product Requirements (PRD)

**Status:** Draft — build from scratch using this doc  
**Question (first):** `email-client`  
**Replaces (when done):** `fds/reviewers/email-client.v1.json`  
**Source content:** `fds/content/email-client.md` (from `final-content/email-client-from-screencapture.md`)

---

## 1. What we are building

A **final-answer reviewer** for front-end system design practice.

After a candidate finishes all RADIO steps and submits, the app sends their **diagram + merged notes**. The reviewer returns a **calibrated score**, **per-section breakdown**, **strengths/gaps**, and **short reasoning** — like a strict senior interviewer, not a cheerleader.

**In scope for v2:** Reviewer only (step 7).  
**Out of scope for v2:** Coach, Tutor, Mock Interviewer (separate PRDs later).

---

## 2. Why v2 exists (learnings from v1 experiment)

| Problem in v1 / messy v2 attempt | v2 fix |
|----------------------------------|--------|
| Rubric copied in 3 places (notebook, JSON, prompts) | **One prompt file** + freeze exports a copy |
| Model could invent security points | **Evidence-only** scoring + **caps in code** |
| v2 changed section weights but kept old eval labels | Eval tied to **final content gold answer** |
| Raw HTML in DOM still got security points | **Hard cap:** total ≤ 39 |
| “Sanitize HTML” with no mechanism still scored high | **Hard cap:** total ≤ 74 without sandbox/CSP/proxy |
| Mid answers (m1, m3) scored too low or too high | **3 few-shot examples** at v2 weights |
| Staff band almost never used | **Policy:** 90+ is rare; eval case s3 = Strong, not Staff |
| 5/9 batch with no clear spec | This PRD + **7/9 gate** before ship |

We are **not** deleting final content or the app. We are **replacing the reviewer brain** with a clean, testable pipeline.

---

## 3. User flow (product)

```text
Steps 1–6  →  Coach (hints only, no score)     [later PRD]
Step 7     →  User submits diagram + full chat
           →  POST /api/ai/review
           →  Show score, band, breakdown, strengths, gaps
```

**Reviewer runs only on complete submissions** — not after step 1 or 3 alone.

---

## 4. Inputs

| Field | Source | Notes |
|-------|--------|--------|
| `question_slug` | `"email-client"` | First question only for MVP |
| `diagram_text` | Canvas export | `Components:` / `Connections:` lines |
| `chat_summary` | Merged session notes | Must include steps 1, 4, 5, 6 (requirements, sync, security, tradeoffs) |

The reviewer must **only** use text in these fields. No hidden context.

---

## 5. Outputs

### 5.1 What the **model** returns (JSON only, no markdown)

```json
{
  "strengths": ["..."],
  "gaps": ["..."],
  "staff_level_additions": ["..."],
  "reasoning": {
    "requirements": "...",
    "architecture": "...",
    "sync_reliability": "...",
    "security": "...",
    "tradeoffs_communication": "..."
  },
  "breakdown": {
    "requirements": 0,
    "architecture": 0,
    "sync_reliability": 0,
    "security": 0,
    "tradeoffs_communication": 0
  }
}
```

**Model must NOT return:** `score_out_of_100`, `band`, or extra fields.

### 5.2 What the **app** computes (deterministic)

```text
total = sum(breakdown)           # after caps applied
band  = Weak | Basic | Competent | Strong | Staff-like
```

| Band | Score |
|------|-------|
| Weak | 0–39 |
| Basic | 40–59 |
| Competent | 60–74 |
| Strong | 75–89 |
| Staff-like | 90–100 (rare) |

### 5.3 Section weights (must sum to 100)

| Section | Max | What we grade |
|---------|-----|----------------|
| requirements | 15 | Scope, offline, desktop vs webmail, constraints |
| architecture | 25 | Source of truth, cache, boundaries, data flow |
| sync_reliability | 25 | Queue, retry, rollback, reconciliation |
| security | 20 | Untrusted HTML, sandbox/CSP, image proxy |
| tradeoffs_communication | 15 | Explicit choices + justification |

---

## 6. Scoring philosophy (human rules for the prompt)

**Do reward**

- Correct system thinking tied to **quoted evidence**
- Tradeoffs with cost/complexity
- Reliability when queue + retry + failure paths are described
- Concrete security **mechanisms** (named in diagram or chat)

**Do not reward**

- Buzzwords without mechanism
- Long answers with no production depth
- Security points when nothing was said

**Do not punish**

- Short but correct answers
- Simple diagrams if reasoning is strong

**Partial credit:** OK when clearly implied in notes — **never** invent sandbox/CSP/proxy if not stated.

---

## 7. Hard rules (code, not prompt)

These run **after** the model response, every time. Same result on every deploy.

| Rule ID | If input contains… | Action |
|---------|-------------------|--------|
| `CAP_RAW_DOM` | Renders/injects email HTML into **host DOM** (not iframe/sandbox) | `security` → min(current, 1); **total → min(total, 39)** |
| `CAP_NO_SECURITY` | Email client and **no** sandboxed iframe, CSP, or image proxy mentioned | **total → min(total, 74)** |
| `CLAMP_SECTIONS` | Any breakdown > section max | Clamp to max |

Store which caps fired in the API response (`caps_applied: []`) for debugging.

---

## 8. Gold reference (what “good” looks like)

Canonical answer lives in content, not in the prompt:

- **File:** `fds/content/email-client.md`
- **Sections:** Requirements → Architecture/HLD → Data model → API → Deep dives (sync, security, performance) → Tradeoffs

**Use gold answer to:**

1. Write the email-specific appendix in the reviewer prompt  
2. Build/update the **9 eval cases** (`docs/email-client-eval-set-v1.md`)  
3. Define expected **score ranges** per case (not only band names)

**Eval policy for s3 (“staff leaning”):** expect band **Strong (80–88)**, not Staff-like — unless the candidate clearly exceeds gold depth.

---

## 9. Prompt structure (single source of truth)

```text
prompts/email-reviewer/
  base.md              # Systems interviewer rules + section guidance + JSON schema
  email-client.md      # Question-specific signals + anti-patterns + 3 few-shots
```

**Freeze artifact** (for production):

```text
fds/reviewers/email-client.v2.json
  - scoring_guide: base + email-client concatenated
  - schema, section_maxima, model, provider
  - eval_accuracy, batch_results
  - prompt_version hash
```

No duplicate rubric inside notebooks after sign-off.

---

## 10. Model & infra

| Setting | Value |
|---------|--------|
| Provider | OpenAI (production) |
| Model | `gpt-4o` |
| Temperature | `0` |
| Response format | JSON object |

Anthropic optional for lab experiments only — not production until JSON + calibration match OpenAI.

---

## 11. Evaluation gate (before ship)

### 11.1 Nine canonical cases

Keep existing case IDs (`w1`…`s3`). For each case document:

- `expected_band`
- `expected_score_range` (e.g. 62–72 for m1)
- `must_not_exceed` / `must_not_fall_below` where caps apply

**Ship when:** band match **≥ 7/9** OR every mismatch is written down and accepted in `ai-labs/outputs/reviewer-v2-signoff.md`.

### 11.2 Quick checks (from eval set)

- Weak cases never above **45**
- Strong cases never below **75** (except documented cap cases)
- w3 (raw HTML DOM) never above **39** after caps

### 11.3 RADIO fixtures

Run **3 full session fixtures** through the app (`npm run test:radio` in `fds/app`) — scores plausible, no JSON errors.

### 11.4 Optional before “production-grade” label

- 3 runs per case, median score — flag if spread > 3 points

---

## 12. Build plan (step by step)

Do these in order. Do not skip to the app until step 6 passes.

| Step | Task | Done when |
|------|------|-----------|
| **0** | Archive old reviewer experiments to `_archive/reviewer-v1-v2-experiment/` (notebooks 03/06, old `prompts/system-reviewer-*`) | Folder exists; README points here |
| **1** | Copy/sync final content → `fds/content/email-client.md` | Content matches gold |
| **2** | Write `prompts/email-reviewer/base.md` (your systems rubric + output schema) | Readable, <400 lines |
| **3** | Write `prompts/email-reviewer/email-client.md` (signals, anti-patterns, 3 few-shots at v2 weights) | Few-shots match section max |
| **4** | Implement `reviewer/caps.py` + `reviewer/schema.py` (Pydantic + cap rules) | Unit tests for w3, m2, s1 |
| **5** | One notebook: `ai-labs/notebooks/01_email_reviewer.ipynb` — setup → single run → one case → batch | You can run end-to-end |
| **6** | Run 9-case batch; tune **one thing at a time**; log in notebook tune table | ≥ 7/9 |
| **7** | `freeze_reviewer.py` → `fds/reviewers/email-client.v2.json` | Freeze committed |
| **8** | Wire `fds/app` review route: load v2, apply caps, return `caps_applied` | Manual submit works |
| **9** | Sign-off doc + update `AI-LABS.md` / `BUILD.md` | v2 marked Done |

**Coach / Tutor / Mock:** start separate PRDs after step 9.

---

## 13. API response shape (app)

```json
{
  "strengths": [],
  "gaps": [],
  "staff_level_additions": [],
  "reasoning": { },
  "breakdown": { },
  "score_out_of_100": 0,
  "band": "Competent",
  "caps_applied": ["CAP_NO_SECURITY"],
  "reviewer_version": "email-client.v2"
}
```

---

## 14. Success metrics

| Metric | Target |
|--------|--------|
| 9-case band accuracy | ≥ 7/9 |
| w3 total after caps | ≤ 39 |
| m1, m3 in range | 60–74 (Competent) |
| s1, s2 in range | 75–89 (Strong) |
| Latency p95 | < 15s (single review) |
| JSON parse failure rate | 0% on eval set |

---

## 15. Explicit non-goals (v2)

- Multi-question reviewer (add `email-client.md`-style overlay per question later)
- Replacing human interviewers
- Scoring partial RADIO steps
- Fine-tuning or custom models
- Perfect use of 0–100 scale (bands matter more)

---

## 16. File map (target end state)

```text
fds/
  content/email-client.md              # Gold content
  reviewers/email-client.v2.json         # Frozen production prompt
  docs/reviewer-v2-prd.md              # This file
  app/...                                # Review route uses v2 + caps

frontend-design-project/
  docs/email-client-eval-set-v1.md       # 9 cases + ranges
  docs/email-client-rubric-v1.md           # Human rubric reference
  ai-labs/
    prompts/email-reviewer/base.md
    prompts/email-reviewer/email-client.md
    reviewer/caps.py                       # Shared with labs via path or copy
    reviewer/schema.py
    notebooks/01_email_reviewer.ipynb
    labs/freeze_email_reviewer.py
    outputs/reviewer-v2-signoff.md
```

---

## 17. Decision log

| Decision | Choice | Reason |
|----------|--------|--------|
| Model outputs total? | No — app sums | Deterministic banding |
| Caps in prompt or code? | **Code** | Prompt failed on w3 |
| Keep v1 until v2 passes? | **Yes** | 6/9 shippable dogfood |
| s3 expected band | **Strong** | Staff-like reserved for exceptional |
| First build surface | **Notebook + caps tests** | Fast iteration |

---

## 18. Next action

Start at **Build plan step 0–2**: archive old prompts, confirm content, write `base.md` + `email-client.md`.

When those exist, implement **step 4 (caps)** before tuning the prompt again.
