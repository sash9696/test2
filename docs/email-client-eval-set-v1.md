# Email Client Eval Set v1 (9 Canonical Cases)

## Purpose
Benchmark the reviewer against fixed-quality submissions before prompt tuning.

## Score band thresholds
- Weak: `0-39`
- Basic: `40-59`
- Competent: `60-74`
- Strong: `75-89`
- Staff-like: `90-100`

---

## Weak cases (3)

### Case W1 — "UI-first, no sync model"
- `id`: `email_w1_ui_only`
- `expected_score_band`: `Weak`
- `expected_score_range`: `20-35`
- `diagram_state_text`:
  - "Three pane UI (folders, list, reading pane)."
  - "Calls backend API for send/read."
  - "Uses local state in components."
- `chat_context`:
  - "Candidate focuses on toolbar buttons and compose modal."
  - "Says offline is nice-to-have and can be ignored."
  - "No discussion of queue, retries, or rollback."
- `expected_key_gaps`:
  - Missing server-as-source-of-truth boundary.
  - No durable task queue for mutations.
  - No failure semantics for send/archive/flag.
  - No security handling for untrusted email HTML.
- `notes`:
  - Good visual decomposition but misses core reliability architecture.

### Case W2 — "POP-first outdated model"
- `id`: `email_w2_pop_primary`
- `expected_score_band`: `Weak`
- `expected_score_range`: `15-30`
- `diagram_state_text`:
  - "POP3 downloads all messages locally and deletes on server."
  - "Each device stores its own mailbox copy."
  - "SMTP used for sending only."
- `chat_context`:
  - "Candidate says this is simpler and faster."
  - "Does not address multi-device consistency issues."
  - "No fallback or IMAP-first strategy."
- `expected_key_gaps`:
  - Wrong primary retrieval model for modern multi-device behavior.
  - No authoritative server mailbox consistency model.
  - No reconciliation strategy across devices.
- `notes`:
  - Explicit architectural mismatch should trigger major deductions.

### Case W3 — "Unsafe rendering"
- `id`: `email_w3_raw_html_render`
- `expected_score_band`: `Weak`
- `expected_score_range`: `25-39`
- `diagram_state_text`:
  - "Renderer injects message HTML into main DOM for speed."
  - "Fetches remote images directly from sender URLs."
  - "Client has optimistic send flag in UI."
- `chat_context`:
  - "Candidate mentions performance optimizations."
  - "No sandbox/CSP/sanitization discussion."
  - "No tracking-pixel mitigation."
- `expected_key_gaps`:
  - Critical trust-boundary/security failure.
  - Missing HTML sanitization and iframe sandbox model.
  - Missing privacy protections for remote content.
- `notes`:
  - Even with decent architecture, security flaw keeps this weak.

---

## Medium cases (3)

### Case M1 — "Solid baseline, shallow failures"
- `id`: `email_m1_partial_reliability`
- `expected_score_band`: `Competent`
- `expected_score_range`: `62-72`
- `diagram_state_text`:
  - "IMAP for read, SMTP for send."
  - "Central store + local cache + three-pane UI."
  - "Basic operation queue for send/archive."
- `chat_context`:
  - "Mentions offline mode and retries."
  - "Does not explain rollback/reconciliation when server rejects."
  - "Touches on tradeoffs briefly."
- `expected_key_gaps`:
  - Incomplete reconciliation/rollback semantics.
  - Limited error-state UX detail.
  - Shallow tradeoff analysis.
- `notes`:
  - Good architecture, moderate depth gaps.

### Case M2 — "Good sync, weak security"
- `id`: `email_m2_sync_good_security_light`
- `expected_score_band`: `Competent`
- `expected_score_range`: `60-70`
- `diagram_state_text`:
  - "Durable task queue with retry/backoff."
  - "Optimistic updates for flags/moves/sends."
  - "Server mailbox source of truth."
- `chat_context`:
  - "Strong explanation of sync boundary."
  - "Security mention limited to 'sanitize HTML' with no mechanism."
  - "No image proxy/tracking mitigation."
- `expected_key_gaps`:
  - Security controls too vague.
  - Missing concrete containment model (sandboxed iframe/CSP).
- `notes`:
  - Reliability is strong, but trust-boundary rigor is insufficient.

### Case M3 — "Structured answer, under-modeled data"
- `id`: `email_m3_data_model_thin`
- `expected_score_band`: `Competent`
- `expected_score_range`: `58-68`
- `diagram_state_text`:
  - "Presents RADIO structure clearly."
  - "Defines modules but data model only as 'messages and users'."
  - "Queue and retries present."
- `chat_context`:
  - "Clear communication and sequencing."
  - "Little detail on folders/threads/attachments/drafts/contact entities."
  - "Mentions performance but no concrete mechanism."
- `expected_key_gaps`:
  - Thin data-model granularity.
  - Limited performance depth.
  - Some missing product-specific deep dives.
- `notes`:
  - Communication is strong; technical depth is moderate.

---

## Strong cases (3)

### Case S1 — "Production-ready sync core"
- `id`: `email_s1_sync_architecture_strong`
- `expected_score_band`: `Strong`
- `expected_score_range`: `80-88`
- `diagram_state_text`:
  - "Server mailbox as source of truth."
  - "Local normalized cache (Account/Folder/Thread/Message/Attachment/Draft)."
  - "Persistent queue with retry/backoff, id reconciliation, rollback."
- `chat_context`:
  - "IMAP canonical retrieval, SMTP submission, POP as legacy fallback."
  - "Explains optimistic UX + durable operations."
  - "Covers key tradeoffs (complexity vs UX/offline)."
- `expected_key_gaps`:
  - Minor: limited detail on accessibility and keyboard model.
- `notes`:
  - Strong architecture and reliability mechanics; near-complete answer.

### Case S2 — "Strong security + reliability"
- `id`: `email_s2_security_reliability_strong`
- `expected_score_band`: `Strong`
- `expected_score_range`: `82-90`
- `diagram_state_text`:
  - "Sandboxed iframe render for email body."
  - "CSP + script/event-handler stripping."
  - "Remote images proxied to reduce tracking leakage."
- `chat_context`:
  - "Also includes durable queue and failure semantics."
  - "Explains user-visible error states and retry feedback."
  - "Mentions undo-send as delayed dispatch pattern."
- `expected_key_gaps`:
  - Minor: fewer details on multi-account architecture boundary.
- `notes`:
  - Security and sync reasoning both high quality.

### Case S3 — "Staff-like reasoning but not perfect"
- `id`: `email_s3_staff_leaning`
- `expected_score_band`: `Staff-like`
- `expected_score_range`: `90-96`
- `diagram_state_text`:
  - "Isomorphic core + pluggable storage layer by runtime."
  - "Unified command dispatch from toolbar/menu/shortcuts."
  - "End-to-end sync lifecycle with reconciliation paths."
- `chat_context`:
  - "Systematic tradeoff discussion for each major decision."
  - "Clear sequence: requirements -> architecture -> data/API -> deep dives."
  - "Includes security, offline, UX, and performance mechanisms."
- `expected_key_gaps`:
  - Very minor implementation details only.
- `notes`:
  - Should score top band if reviewer is calibrated correctly.

---

## Quick validation checks for the reviewer
- Weak cases should never score above `45`.
- Strong/Staff-like cases should never score below `75`.
- Any case with raw HTML host-DOM rendering should trigger security penalties.
- Any case claiming offline without durable queue/retry/reconciliation should be capped to Basic/low Competent.
