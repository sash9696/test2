# FDS AI Labs — Reviewer v2 + Final Content

Starter repo for the **reviewer v2** pipeline spec and full **front-end system design** gold content.

## Docs (AI labs)

| Path | Description |
|------|-------------|
| [docs/reviewer-v2-prd.md](docs/reviewer-v2-prd.md) | Reviewer v2 PRD — build step-by-step from here |
| [docs/AI-LABS.md](docs/AI-LABS.md) | Lab workflow and ship gates |
| [docs/email-client-eval-set-v1.md](docs/email-client-eval-set-v1.md) | 9 canonical eval cases (email-client) |
| [docs/email-client-rubric-v1.md](docs/email-client-rubric-v1.md) | Human rubric reference |

## Final content (all questions)

[`final-content/`](final-content/) — 21 case studies (markdown) + pattern map + raw scrape:

| File | Topic |
|------|--------|
| `email-client-from-screencapture.md` | Email client (Outlook-style) — **first reviewer v2 question** |
| `chat-app-from-screencapture.md` | Chat app |
| `news-feed-facebook-from-screencapture.md` | News feed |
| `google-docs-from-screencapture.md` | Google Docs |
| `rich-text-editor-from-screencapture.md` | Rich text editor |
| `video-streaming-from-screencapture.md` | Video streaming |
| … | See folder for full list (autocomplete, data table, e-commerce, etc.) |
| `pattern-mapping.md` | Pattern index |
| `fes_ai_pattern_map.svg` | Pattern diagram |
| `case-study-pages.raw.jsonl` | Raw captured pages (~10MB) |

Each `*-from-screencapture.md` includes question, requirements, architecture/HLD, data model, deep dives, and tradeoffs where captured.

## Next steps

1. Read `docs/reviewer-v2-prd.md` (build plan steps 0–9).
2. Start with `final-content/email-client-from-screencapture.md` for reviewer calibration.
3. Add `ai-labs/` (prompts, caps, notebook) in this repo when ready.

## Related

Production app and frozen prompts live in the main `fds` monorepo (private).
