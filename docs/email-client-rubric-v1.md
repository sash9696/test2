# Email Client Rubric v1 (Atomic, Reviewer-Ready)

## Purpose
Use this rubric to evaluate a candidate's front-end system design answer for `email-client`.
This rubric is designed for deterministic AI scoring, not generic feedback.

## Scoring model
- Total score: **100**
- Score = sum of section scores after penalties
- Clamp final score to `0..100`

## Section weights
- Requirements and scope: **15**
- Architecture and data flow: **30**
- Sync, offline, and reliability: **25**
- Security and trust boundaries: **10**
- Tradeoffs, depth, and communication quality: **20**

## Section 1: Requirements and scope (15)

### Must-have checks (12 total)
- **R1 (4 pts):** Explicitly distinguishes desktop email client vs webmail.
- **R2 (4 pts):** States core capabilities: send, retrieve, local access/search/offline browsing.
- **R3 (4 pts):** Clarifies at least one important scope constraint (e.g., multi-account out of scope, threading optional).

### Bonus checks (3 total)
- **R4 (3 pts):** Mentions target platforms or practical assumptions (Windows/macOS/Linux or protocol abstraction assumptions).

### Deduction triggers (apply once each)
- **RD1 (-3):** Treats problem as generic backend design and skips client goals.
- **RD2 (-2):** No explicit functional/non-functional distinction when discussing requirements.

## Section 2: Architecture and data flow (30)

### Must-have checks (24 total)
- **A1 (6 pts):** Uses server-mailbox-as-source-of-truth architecture (not client-only source).
- **A2 (6 pts):** Includes local cache/store for responsive UI and offline read/search.
- **A3 (6 pts):** Includes central state pattern (store/reducer/command flow) or equivalent coherent app data flow.
- **A4 (6 pts):** Defines major modules and boundaries: view layer, store/model, data-access/sync boundary, server boundary.

### Bonus checks (6 total)
- **A5 (3 pts):** Defines normalized entities or clear data model (Account/Folder/Thread/Message/Attachment/Draft/Contact).
- **A6 (3 pts):** Describes unified command dispatch across multiple triggers (toolbar/menu/shortcut).

### Deduction triggers
- **AD1 (-6):** Chooses POP/store-and-forward as primary architecture for modern multi-device usage without justification.
- **AD2 (-4):** No clear component responsibilities (boxes but no role clarity).

## Section 3: Sync, offline, and reliability (25)

### Must-have checks (20 total)
- **S1 (5 pts):** Uses optimistic local updates for user actions (send/archive/flag/move/delete) with eventual server sync.
- **S2 (5 pts):** Includes persistent durable task/operation queue (survives restart).
- **S3 (5 pts):** Includes retry/backoff and failure handling (server reject, timeout, offline).
- **S4 (5 pts):** Explains reconciliation/rollback behavior to prevent silent divergence.

### Bonus checks (5 total)
- **S5 (3 pts):** Mentions IMAP as canonical retrieval and SMTP as submission (POP as fallback only).
- **S6 (2 pts):** Covers at least one advanced reliability behavior (undo-send delay model, conflict/state drift handling, ID reconciliation).

### Deduction triggers
- **SD1 (-8):** Claims offline support but provides no queue/retry/reconciliation mechanism.
- **SD2 (-4):** Treats send success as immediate without ack/error semantics.

## Section 4: Security and trust boundaries (10)

### Must-have checks (8 total)
- **Q1 (4 pts):** Treats incoming email HTML as untrusted content.
- **Q2 (4 pts):** Proposes concrete containment/sanitization (sandboxed iframe/CSP/script stripping/event-handler stripping).

### Bonus checks (2 total)
- **Q3 (2 pts):** Mentions remote image proxying/tracking-pixel mitigation or equivalent privacy safeguard.

### Deduction triggers
- **QD1 (-6):** Suggests rendering raw email HTML directly in host DOM.

## Section 5: Tradeoffs, depth, and communication quality (20)

### Must-have checks (14 total)
- **T1 (5 pts):** Discusses at least two tradeoffs with context-based choice (not one-solution-only).
- **T2 (5 pts):** Prioritizes product-relevant deep dives (sync/offline/perf/a11y/security) instead of irrelevant tooling debates.
- **T3 (4 pts):** Explanation is structured and logically sequenced (requirements -> architecture -> data/interface -> deep dive).

### Bonus checks (6 total)
- **T4 (3 pts):** Includes practical UX details tied to architecture (loading/error states, keyboard-heavy workflows, toasts/feedback).
- **T5 (3 pts):** Covers performance with concrete mechanism (list virtualization/windowing, lazy rendering, caching strategy).

### Deduction triggers
- **TD1 (-4):** Heavy buzzwords without mechanism explanation.
- **TD2 (-4):** Spends major time on irrelevant backend/DevOps details for this prompt.

---

## Score bands
- **0-39: Weak**
  - Missing core architecture for sync/offline or has severe correctness/security gaps.
- **40-59: Basic**
  - Understands problem at surface level, but misses critical reliability/security/system details.
- **60-74: Competent**
  - Solid baseline architecture and partial depth; some missing rigor in failure/reconciliation/tradeoffs.
- **75-89: Strong**
  - Coherent architecture, reliable sync model, concrete risk handling, and relevant tradeoff discussion.
- **90-100: Staff-like**
  - Strong + clear boundary ownership, robust failure semantics, explicit tradeoffs, and production-grade details.

## Reviewer output expectations
When this rubric is used, reviewer output should include:
- `strengths`: mapped to passed checks
- `gaps`: mapped to failed must-have checks or major deductions
- `staff_level_additions`: mapped to missing bonus/advanced reliability depth
- `score_out_of_100` and category-aware `breakdown`

## Strict scoring guidance
- Do not award points for vague statements without mechanism.
- Prefer "explicitly mentioned and connected to architecture" over isolated keyword mentions.
- If a candidate describes an equivalent mechanism with different naming, award points.
- Deduction triggers can stack, but avoid double-penalizing the exact same omission in multiple categories.
