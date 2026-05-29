# AI labs first (gate before product)

**Rule:** Finish notebook + eval for each MVP AI mode in `frontend-design-project/ai-labs/` before building Tutor/Coach/Mock in `fds/app/`.

**Reviewer v2 spec:** [reviewer-v2-prd.md](./reviewer-v2-prd.md) — build from scratch checklist.

Product code (`fds/app/`) only ports **frozen** prompts that passed eval.

---

## Location

```text
frontend-design-project/ai-labs/
  notebooks/     # prototype + eval
  labs/          # CLI scripts after notebook sign-off
  outputs/       # saved runs, batch results
```

Reviewer freeze copied to `fds/reviewers/` — re-freeze from ai-labs when prompt changes.

---

## MVP modes → labs

| Priority | Mode | Notebook | Script | Eval | Status |
|----------|------|----------|--------|------|--------|
| 1a | **Reviewer v1** | `03_diagram_review.ipynb` | `03_diagram_review.py`, `freeze_diagram_reviewer.py` | 9-case + RADIO | **Done** → `email-client.v1.json` |
| 1b | **Reviewer v2** | `06_system_reviewer.ipynb` | `04_system_reviewer.py`, `run_system_reviewer_batch.py`, `freeze_system_reviewer.py` | 9-case batch (compare to v1) | **In progress** |
| 2 | **Coach** | `04_practice_coach.ipynb` *(create)* | `04_practice_coach.py` *(create)* | 6 steps × 2–3 scenarios | **Not started** |
| 3 | **Tutor** | `02_tutor_turn.ipynb` | `02_tutor_turn.py` | 5 guide sections × weak/strong answer | **Script only** |
| 4 | **Interviewer** | `04_mock_interviewer.ipynb` | `05_mock_interviewer.py` *(rename)* | 1 full mock transcript | **Placeholder** |
| — | Learning path | `01_learning_path.ipynb` | `01_learning_path.py` | 3 onboarding profiles | Optional MVP |
| — | Layer B summarizer | `05_layerb_summarizer.ipynb` | — | sample 10 URLs | **Post-MVP** |

---

## Per-lab done criteria

### Reviewer (sign-off checklist)

- [x] Compact `scoring_guide` + few-shot anchors
- [x] `DiagramReview` schema stable
- [x] 9-case batch ≥7/9 band accuracy (or documented exceptions)
- [x] 3 full RADIO fixtures tested (`npm run test:radio` in fds or `freeze_diagram_reviewer.py`)
- [x] Frozen JSON in `fds/reviewers/email-client.v1.json`
- [x] v1 eval summary (implicit in freeze JSON)
- [ ] v2 sign-off: `ai-labs/outputs/reviewer-v2-eval-summary.md` — **5/9 baseline**, tune before replacing v1

### Coach

- [ ] System prompt: one hint OR one question; never full architecture
- [ ] Input: `step` (1–6), `diagram_text`, `step_notes`, `question_slug`
- [ ] Checklist per step (from `fds/docs/practice-mode.md`)
- [ ] Eval: run 12–18 fixed scenarios; human read for “leaks solution?”
- [ ] Freeze → `fds/coaches/email-client.v1.json` *(new artifact type)*

### Tutor

- [ ] Load section from `fds/content/email-client.md` (or slice in notebook)
- [ ] Output schema: `explanation` (short) + `socratic_question` + optional `hint`
- [ ] Eval: weak vs strong user answers on 5 sections; no full solutions
- [ ] Freeze → `fds/tutors/email-client.v1.json` or section prompts file

### Interviewer

- [ ] Phases: requirements → hld → deep_dive → tradeoffs → wrap
- [ ] Persona: hiring manager; no teaching
- [ ] Eval: 1 scripted run + 1 free run; phase transitions make sense
- [ ] Freeze → `fds/interviewers/email-client.v1.json`
- [ ] Optional: end report = call Reviewer on transcript

---

## Workflow (each lab)

```text
1. Notebook — markdown goal + single-run cells
2. Batch / scenario eval — save to outputs/
3. Tune prompt (one change at a time)
4. Freeze JSON + short eval summary
5. Port to labs/*.py
6. Copy freeze to fds/reviewers|coaches|tutors|interviewers/
```

**Do not** port to `fds/app/` until step 4 sign-off for that mode.

---

## Product build (blocked until)

| fds step | Blocked on |
|----------|------------|
| Step 2 Coach UI | Coach lab freeze |
| Step 3 Tutor UI | Tutor lab freeze |
| Step 4 Mock UI | Interviewer lab freeze |
| Step 5 Deploy | All four freezes + 3+ questions |

Reviewer-only dogfood in `fds/app/` is OK while other labs run.

---

## Commands

```bash
cd frontend-design-project/ai-labs
uv run python labs/03_diagram_review.py
uv run python labs/freeze_diagram_reviewer.py
uv run python labs/04_system_reviewer.py
uv run python labs/run_system_reviewer_batch.py
uv run python labs/freeze_system_reviewer.py
uv run python labs/02_tutor_turn.py

cd ../../fds/app && npm run test:radio
```
