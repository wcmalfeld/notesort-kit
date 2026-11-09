# First message to paste into Claude Code

Set up first. Create an empty folder (or repo) called `notesort`, then copy in this kit:

```
notesort/
├── CLAUDE.md
├── docs/
│   ├── PRD.md              ← save your full PRD here
│   └── BUILD_PLAN.md
├── lib/ai/prompts.ts
└── tests/fixtures/golden.json
```

Then open Claude Code in that folder and paste:

---

Read CLAUDE.md, docs/BUILD_PLAN.md and docs/PRD.md in full before doing anything.

We're building NoteSort. CLAUDE.md holds the rules and the decisions that override the PRD. BUILD_PLAN.md is
the order of work.

Start with Phase 0: write docs/SPIKE.md with the iPhone test steps, then stop. I'll run them on my phone
and paste back the results. Don't scaffold the app until I've done that.

Before you write SPIKE.md, give me a short summary: anything in the three files that is unclear,
contradictory, or that you'd do differently, and why. Keep it under 15 bullets.

---

After Phase 0, start each phase with:

> Do Phase N from docs/BUILD_PLAN.md. Use plan mode first and show me the plan. When you're done, tick the
> exit criteria you've met and tell me plainly about any you haven't.
