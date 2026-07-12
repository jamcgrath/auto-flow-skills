# Changelog

Notable changes to the auto-flow skills, newest first. This is the *what* at a glance —
the *why* (Intent / Approach / Alternatives ruled out) lives in each commit's Decision Log,
and the standing design rationale in [`skills/auto-dev-flow/SPEC.md`](skills/auto-dev-flow/SPEC.md).

Phase 1 / experimental — not yet versioned or released, so everything sits under _Unreleased_.

## Unreleased

### Changed

- **Test skills — criteria now fall back past the ticket.** `auto-author-acceptance-tests`,
  `auto-audit-tests`, and `auto-verify-build` read acceptance criteria from
  `.dev-flow/<task>/TICKET_CONTEXT.md` **if it exists, otherwise from the approved `PLAN.md` /
  the task description**. Previously all three hardcoded `TICKET_CONTEXT.md`, so an ad-hoc /
  typed-task run (where `auto-verify-ticket` is skipped and no `TICKET_CONTEXT.md` is written) left
  them with no criteria source. Mirrors the same fix already shipped in dev-flow.

- **`auto-plan-brief` — recon fan-out scaled to the surface.** Reconnaissance now hands the search to
  `Explore` subagent(s) **in proportion to the surface**: one for a bounded, familiar area; 2–3 in
  parallel — each tracing a distinct facet — for a broad or unfamiliar one, then the main thread reads
  the union and judges relevance. Small changes still get a single pass (a needless spread just burns
  tokens). Grounds the unattended build better, where there's no human to catch a thin plan.

- **`auto-dev-flow` — compaction-safe by construction.** The unattended flow is now explicitly
  robust to the harness summarizing (compacting) context during a long run. Every value the flow
  depends on across a phase boundary — `base` rev, the protected acceptance-test set, open decisive
  forks, the latest verdict — is written to `.dev-flow/<task>/` before it's needed, so a
  summarization is recoverable rather than lossy. Re-reading is a **lazy fallback, not a routine**:
  a specific fact is pulled from the last saved artifact only when it isn't reliably in context,
  never reloaded wholesale at every phase (which would add tokens and make compaction fire *more*
  often). The flow never drives `/compact` itself; isolating the build is the named pressure-relief
  lever if context ever bites. Rationale in `SPEC.md` → _Compaction-safety_.

- **`auto-implement-brief` — builder's own verify scoped to the committed tests.** Step 5 is now a
  **pre-flight, not a verdict**: lint + typecheck + run the committed acceptance tests at their
  layer, and stop. The adversarial, at-the-layer verification stays with the independent
  `auto-verify-build` (fresh context). This protects the build↔verify loop's 3-attempt budget — the
  builder catches contract failures itself instead of burning a verify pass — and sharpens the
  separation of powers (builder ≠ verifier). The builder no longer eyeballs criteria beyond the
  committed tests; that untrusted signal already rides to the human at the merge gate.

- **`auto-implement-brief` — description made layer-agnostic.** The skill's one-line description no
  longer says "verify in the browser" (dev-flow's front-end inheritance) — it now says "verify at
  the change's layer (browser for UI, tests/DB otherwise)," matching how the skill actually verifies.
