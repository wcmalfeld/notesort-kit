# NoteSort build plan

This follows PRD §16 in the same order, with two changes. A device spike comes first, and the engine comes
before the UI, because the engine is the product. Each phase ends with exit criteria. Don't start the
next phase until every box is ticked. Commit at the end of each phase.

Phases 0 and 7 need the owner on a real iPhone. Claude Code prepares everything for them, then stops and asks.

---

## Phase 0 — Device spike (owner, about 20 minutes, no app code)

Goal: prove the iPhone round trip before building on it.

Claude Code: write `docs/SPIKE.md` with these steps, then pause for the results.

1. In Apple Notes, open a test note with a title, bullets, a checklist (one item ticked), a quote and a link.
   Share it, then choose **Export as Markdown**. Record whether the option exists, and what the exported file
   looks like: is the title a `# ` heading, and how do checklists appear?
2. Build a throwaway Shortcut: Receive Text / Rich Text / Files from the Share Sheet → **Get Text from Input** →
   **Quick Look**. Share the note directly to it, then the exported `.md` file. Record which input arrives
   with line breaks and checkboxes intact.
3. Build a second throwaway Shortcut: a **Text** action holding a sample organized Markdown note (from
   CLAUDE.md output rules) → **Make Rich Text from Markdown** → **Copy to Clipboard**. Paste it into a new note.
   Record whether H1/H2 render as headings, bullets as bullets, `>` as a quote, and what happens to `☐`/`☑`.
4. Repeat step 3 with **Copy to Clipboard** on the raw Markdown, for comparison.

Exit criteria
- [ ] `docs/SPIKE.md` holds the owner's results.
- [ ] The input path (export, direct share or clipboard) is chosen and noted in CLAUDE.md.
- [ ] The paste format is confirmed. If rich text pastes badly, update decision D2 in CLAUDE.md before Phase 4.

## Phase 1 — Scaffold and inspect (PRD Step 1)

- Inspect the repo first. If it's empty, create Next.js (App Router) + TypeScript + Tailwind + shadcn/ui, Vitest and Playwright.
- Add `.env.example` (PRD §14; set `ANTHROPIC_MODEL=claude-haiku-5-5` as the suggested default),
  plus startup env validation with zod that fails closed in production when `APP_ACCESS_TOKEN` is missing.
- Create `lib/categories/taxonomy.ts` from PRD §4 (ids, labels, order, descriptions).
- Move the provided `lib/ai/prompts.ts` and `tests/fixtures/golden.json` into place.
- Fill in the Commands section of CLAUDE.md.

Exit criteria
- [ ] `npm run dev`, `npm test`, `npm run lint` and `npm run typecheck` all run clean.
- [ ] Taxonomy unit test: 10 ids, correct order, `needs_review` last.

## Phase 2 — Parser, validator, formatter (PRD Steps 3–4, no AI yet)

Pure functions with tests written first. Follow the rules in CLAUDE.md.

- `lib/notes/parse.ts`: blocks, title extraction and re-run detection.
- `lib/notes/validate.ts`: zod schema for the classification output, plus coverage checks (missing, duplicate
  and unexpected IDs, invalid categories).
- `lib/notes/categorize.ts`: merges classifications with blocks; `low` confidence → `needs_review`.
- `lib/notes/format.ts`: Markdown output, plus `toRichHtml()` for the clipboard (☐/☑).
- `tests/fixtures/notes/`: the 10 notes from PRD §12.1. Write them realistic and messy: run-on lines,
  no punctuation, emojis, URLs, a code snippet, repeated lines, one note near 40,000 characters.

Exit criteria
- [ ] The preservation test passes on all 10 fixtures with a fake classifier that assigns random categories.
      Every `raw` block is recoverable from the output, exactly once, in the right section and order.
- [ ] Validator tests cover every rejection case in PRD §6.3.
- [ ] Formatter snapshot tests cover each block kind in and out of ACTION ITEMS.

## Phase 3 — Classification engine and API (PRD Steps 3, 5, 6)

- `lib/ai/provider.ts` interface, `mock.ts` (deterministic keyword rules, good enough for UI work), and
  `anthropic.ts` (tool use with a forced `classify` tool, prompt caching, 20 s timeout).
- `lib/ai/classification.ts`: call → validate → one retry with the validation error appended → error.
- `POST /api/organize` (PRD §8.3), `GET /api/health`, `POST /api/auth/verify`.
  Bearer auth, 40,000-character limit, best-effort per-token rate limit, `maxDuration = 30`, and logs with
  metadata only.
- `scripts/eval.ts` → `npm run eval`. Each golden item is sent inside a realistic surrounding note,
  as described in the golden file's README note. Report accuracy per category, a confusion matrix,
  the needs_review rate, consistency across `--repeat` runs, and p95 latency.

Exit criteria
- [ ] API integration tests (mock provider) cover each error code, auth failure, the size limit and a retry
      that succeeds.
- [ ] A grep or test confirms that no note text and no Authorization header reaches `console.*` or logging.
- [ ] `npm run eval` meets the targets in CLAUDE.md. If it doesn't, iterate on `prompts.ts` (examples and
      precedence rules) rather than loosening the targets. Report the confusion matrix to the owner.

## Phase 4 — Web UI (PRD Steps 2, 4; §5, §13)

Build it mobile-first, with the copy from PRD §13.1.

- Input state: token entry (sessionStorage), textarea with character count, optional title, `.md`/`.txt`
  upload with rejection message, clear button, privacy notice.
- Processing state: disabled button and "Organizing your thoughts…", with the input kept.
- Results state: counts, collapsible sections with item counts, a per-item category selector (instant,
  no AI call), Preview/Markdown toggle, sticky **Copy Organized Note** (rich text + plain, per D2), the manual-copy
  fallback, **Download** (`organized-notes-YYYY-MM-DD.md`), **Start over**, and **Re-sort everything** (D5).
- Settings: token entry and clear, plus an "About your data" text.

Exit criteria
- [ ] Playwright covers the full web flow at 390×844 and 1280×800 with the mock provider: paste,
      organize, re-categorize, copy and download.
- [ ] Keyboard-only run-through works, with visible focus and a status message announced for loading and errors.
- [ ] Lighthouse accessibility ≥ 95 on iPhone viewport.

## Phase 5 — Deploy (PRD Step 9)

- Write `docs/DEPLOYMENT.md`. Deploy to Vercel with env vars set and HTTPS confirmed.
- Smoke test the live `/api/health` and `/api/organize` with curl.

Exit criteria
- [ ] The live endpoint returns organized output for fixture note 1, and 401 without a token.

## Phase 6 — Shortcut docs (PRD Step 7, §9)

- Write `docs/APPLE_SHORTCUT_SETUP.md` and the `/shortcut-setup` page, built on the Phase 0 findings.
  Action order: Receive input (no input → Get Clipboard) → Get Text from Input → stop if empty →
  Get Contents of URL (POST, JSON, Bearer) → if `success` isn't true, show `error.message` and stop →
  Get Dictionary Value `data.markdown` → Make Rich Text from Markdown → Copy to Clipboard → notification.
- Don't produce or claim a `.shortcut` file unless it has really been built and tested.

## Phase 7 — Device test and handoff (owner + Claude Code, PRD Step 8)

- The owner runs the Shortcut on 5 real daily notes. Claude Code fixes what breaks.
- Do a manual pass of the PRD §12.4 checklist on iPhone Safari and desktop.
- Write the handoff report (PRD §18) into `docs/HANDOFF.md`.

Exit criteria
- [ ] Every PRD §12.4 box is ticked, including "Shortcut API workflow is tested with an actual iPhone."
- [ ] Share → paste takes ≤ 15 s of the owner's effort on a typical note.
