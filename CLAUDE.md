# NoteSort — guide for Claude Code

NoteSort turns an unstructured daily Apple Notes note into a categorized note, without changing a word.
The full spec is `docs/PRD.md`. The build order is `docs/BUILD_PLAN.md`. This file holds the rules that
apply in every session. **Where this file and the PRD disagree, this file wins.** The differences are
listed under "Decisions that extend the PRD" below and were agreed with the owner.

## Non-negotiables

1. **The model never produces note text.** It returns only `{blockId, categoryId, confidence}`.
   All output text is assembled by our code from the original source blocks. If you find yourself
   asking the model to return, rewrite or echo content, stop: that's a bug.
2. **Every source block appears in the output exactly once.** Nothing is dropped, duplicated, merged
   or split. This is enforced in code (`lib/notes/validate.ts`) and by tests, not by the prompt.
3. **Never fake success.** If validation fails after one retry, return an error and leave the user's
   input untouched. The Shortcut must never copy partial output.
4. **Never persist or log note content.** No database, no localStorage for notes, no note text in
   logs, error reports or analytics. Never log the `Authorization` header.
5. **Don't overbuild.** No database, accounts, auth platform, analytics or extra infrastructure.
   Out-of-scope items are in PRD §2.2 and §15.

## Commands

Fill these in once the project is scaffolded, and keep them current.

- `npm run dev` — local dev server (uses the mock provider when `AI_PROVIDER=mock` or no API key)
- `npm test` — unit and integration tests (Vitest)
- `npm run test:e2e` — Playwright, including iPhone viewport
- `npm run eval` — runs `tests/fixtures/golden.json` against the real provider and prints accuracy,
  NEEDS REVIEW rate, consistency and latency. Needs `ANTHROPIC_API_KEY`. Run it after every prompt change.
- `npm run lint && npm run typecheck`

## Architecture in one breath

`parse` (remark → source blocks with IDs) → `classify` (provider → JSON) → `validate` (zod + coverage)
→ retry once → `categorize` (low confidence → needs_review) → `format` (fixed order → Markdown)
→ response. The UI re-runs `format` locally when the user changes a category; no new AI call.

- Taxonomy is the single source of truth: `lib/categories/taxonomy.ts` (id, label, order, description).
  Prompt, validator, formatter and UI all import from it.
- Provider abstraction: `lib/ai/provider.ts`. `anthropic.ts` and `mock.ts` implement it. The model comes from
  `ANTHROPIC_MODEL`; never hardcode a model name elsewhere. Enforce the schema with tool use
  (a single `classify` tool with `tool_choice` forced), then validate again with zod regardless.
- The system prompt, few-shot examples, `CLASSIFY_TOOL` schema and `buildUserMessage()` live in
  `lib/ai/prompts.ts` (already written; refine, don't rewrite). Mark the system prompt for prompt caching.
- The tool schema adds one optional field to PRD §6.3: `candidates` (up to 2 category ids) on
  needs_review or low-confidence items. The validator accepts it, and the results screen shows the candidates
  as one-tap buttons on NEEDS REVIEW items. It never reaches the Markdown.

## Parsing rules (lib/notes/parse.ts)

- Normalize: strip BOM, CRLF/CR → LF. Change nothing else.
- Use remark (`unified` + `remark-parse` + `remark-gfm`) and slice the **original source text** by node
  positions. Never re-stringify the AST, because that changes the user's formatting.
- One block per top-level paragraph, list item (with its nested sub-items), checklist item, blockquote,
  heading, code block, table or thematic-break-separated chunk. Wrapped lines of one paragraph are one block.
- Store per block: `id` (`block_001`…), `position`, `raw` (exact source slice), `kind`
  (`paragraph|list_item|task|quote|heading|code|table|other`), `checked` (for tasks).
- **Title.** Use the explicit `title` field if given. Otherwise, if the note's *first* block is an H1, it
  becomes the title and is not classified. Otherwise the title is `Organized Notes`. Never invent a title or date.
- **User headings** (other than the title) are blocks like any other. They are sent to the model with
  the blocks under them as context. A heading with no standalone meaning goes to `needs_review`, not the bin.
- **Re-runs.** If an H2 exactly matches a NoteSort category label, the blocks under it are pre-assigned to
  that category and are not sent to the model, unless the request sets `options.resortAll: true`.
  The matched H2 headings themselves are dropped, since the formatter regenerates them.

## Output rules (lib/notes/format.ts), from PRD §7 plus the agreed decisions

- `# Title`, then `## LABEL` per populated category in taxonomy order, blank line between sections.
- Items keep original relative order within a section.
- Emit each block's own text unchanged. Only the list marker is normalized:
  - In ACTION ITEMS: `- [ ] text` / `- [x] text`. Keep the original checked state.
  - Quotes (`kind: quote`) keep their `> ` lines in any category.
  - Tasks filed outside ACTION ITEMS keep their checkbox and state.
  - Other paragraphs and list items: `- text`. Continuation lines of multi-line blocks are indented 2 spaces.
  - Code blocks, tables and headings are emitted as-is, never bulleted.
- No confidence scores, rationales or AI commentary in the output.
- **Preservation test.** Stripping our added markers from the output must give back every block's `raw`
  text, character for character, in the right section.

## Decisions that extend the PRD (agreed 2026-11-09)

| # | Decision | Effect on the build |
|---|---|---|
| D2 | Clipboard carries **rich text**, so Apple Notes shows real headings and bullets. Raw Markdown stays available. | Web copy writes a `ClipboardItem` with both `text/html` (rendered from our Markdown) and `text/plain` (the Markdown). The Shortcut runs **Make Rich Text from Markdown** before **Copy to Clipboard**. The API still returns `data.markdown`. |
| D3 | Checklists come back as text, not native Notes checklists. | In the HTML render only, `- [ ]` → `☐` and `- [x]` → `☑` (pasted `<input>` checkboxes don't survive). The Markdown keeps `- [ ]`. Say this once on `/shortcut-setup`. |
| D4 | Next.js on Vercel with the owner's own Anthropic key | As PRD §8. Set the organize route's `maxDuration` to 30 s. |
| D5 | Re-runs keep already-filed items | See "Re-runs" above. Results screen offers "Re-sort everything". |
| — | Measurable quality bar | `npm run eval` targets: ≥ 90% category match on the golden set (100% on rows marked `unambiguous`), ≤ 10% needs_review, ≥ 97% identical categories across 3 runs, p95 ≤ 8 s for a 3,000-word note. |
| — | Device spike first | Before UI work, verify on a real iPhone that Notes' Markdown export or the share sheet reaches a Shortcut, and that rich text pastes cleanly into Notes. See BUILD_PLAN Phase 0. |

PRD values that stand: 40,000-character limit (`MAX_NOTE_CHARACTERS`), web token in **sessionStorage**,
`low` confidence → `needs_review`, no block splitting in MVP, error codes from PRD §8.3, and the API shape from PRD §8.3.

## Working style

- Follow `docs/BUILD_PLAN.md` phase by phase. Finish each phase's exit criteria before starting the next,
  and tick them off in that file.
- Write the test first for parser, validator and formatter changes; they're pure functions.
- Use the mock provider for UI work. Use the real provider only for evals and the end-to-end check.
- Golden labels in `tests/fixtures/golden.json` belong to the owner. If you think a label is wrong,
  say so in your summary rather than editing it to make the eval pass.
- Never claim the iPhone Shortcut works unless the owner has tested it on a device.
