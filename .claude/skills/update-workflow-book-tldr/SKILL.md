---
name: update-workflow-book-tldr
description: "Generate or refresh docs/workflow-book/TLDR.md, the condensed half-hour summary of the workflow book, from the book's own chapters. Computes a drift window from the baseline SHA stamped on TLDR.md line 1 to HEAD over docs/workflow-book/ source, refreshes only the summary sections whose chapters changed, and re-stamps. TRIGGER: the workflow book (README/TOC/chapters) changed since TLDR.md was last built. SKIP: no change to docs/workflow-book/ since the stamp."
argument-hint: "[--rebuild (ignore stamp, rebuild from all chapters)]"
user-invocable: true
---

Keep `docs/workflow-book/TLDR.md` in step with the book it condenses. The book (`docs/workflow-book/`) is the authoritative source; `TLDR.md` is a derived summary that must not drift from it. This skill refreshes the summary when the book changes, touching only the sections whose source chapters moved.

The output is an updated `TLDR.md`, not a change to the book. This skill never edits the chapters, `README.md`, or `TOC.md`.

## When to use this, and when not

Use it when the workflow book has changed and the TL;DR needs to catch up: "refresh the workflow-book TL;DR", "the book changed, update the summary", "rebuild TLDR.md". Also use it to build `TLDR.md` from scratch if the file is missing.

Do not use it to edit the book itself — that is the book-builder pipeline's job (`workflow-book-builder/PIPELINE.md`). This skill reads the book and writes only the summary. If the book and the summary disagree, the book wins.

## How the summary tracks the book

`TLDR.md` carries a baseline stamp on its first line:

```
<!-- book-source-sha: <40-char SHA> -->
```

The SHA is the most recent commit that touched the book's source (`docs/workflow-book/README.md`, `docs/workflow-book/TOC.md`, or `docs/workflow-book/chapters/`), reachable from `HEAD` when the summary was last built. The drift window is every book-source commit between that stamp and the current `HEAD`. An empty window means the summary is current; a non-empty window names the chapters that changed, and only the summary sections mapped to those chapters are rewritten.

This is the same drift-window model the book itself uses against `.claude/workflow/` (see `workflow-book-builder/PIPELINE.md` Step 1), applied one level up: the book is to the workflow what this summary is to the book.

## Chapter → summary-section map

Each book chapter feeds one or more numbered sections of `TLDR.md`. When a chapter changes, refresh the mapped section(s) and no others. Keep this table in sync if the book's chapter structure changes (a `new-or-restructure` chapter may need a new summary section or a remap).

| Book chapter | TLDR section(s) |
|---|---|
| 01 — workflow at a glance | §1 (two ideas), §2 (five-phase shape) |
| 02 — minimal change end-to-end | §2, §3 (the minimal-run paragraph) |
| 03 — tiers and the tier gate | §3 |
| 04 — Phase 0 research | §4 |
| 05 — Phase 1 design document | §5 (design-document part) |
| 06 — Phase 1 plan and tracks | §5 (derived-plan part) |
| 07 — phases, sessions, phase ledger | §10, and the one-session rule in §1/§2 |
| 08 — Phase 2 plan review | §6 |
| 09 — Phase A decomposition | §7 (Phase A) |
| 10 — Phase B implement-test-commit | §7 (Phase B) |
| 11 — dimensional review agents | §8 |
| 12 — Phase C track review & completion | §7 (Phase C) |
| 13 — Phase 4 final artifacts | §9 |
| 14 — mid-flight changes | §11 |
| 15 — drift and migration | §12 |
| 16 — house style & self-improvement | §13 |

## Procedure

1. **Establish the baseline.** Read line 1 of `docs/workflow-book/TLDR.md` and extract the `book-source-sha` value (call it `TLDR_SHA`). If the file is missing, has no stamp, or `$ARGUMENTS` contains `--rebuild`, treat the baseline as empty (full rebuild from every chapter). Record the current `HEAD` (call it `NEW_SHA`):

   ```
   git rev-parse HEAD
   ```

2. **Compute the drift window.** Unless rebuilding, list the book-source commits since the stamp:

   ```
   git log <TLDR_SHA>..<NEW_SHA> --name-only -- \
     docs/workflow-book/README.md docs/workflow-book/TOC.md docs/workflow-book/chapters/
   ```

   - **Empty output** → the summary is current. Report "TLDR.md is up to date (no book-source changes since `<short TLDR_SHA>`)" and stop. Do not rewrite or re-stamp.
   - **Non-empty** → collect the set of changed chapter files (and note whether `TOC.md` changed, which can signal a restructure). These are the only chapters in scope this run.

3. **Read the changed chapters in full.** For each changed chapter, read the current file under `docs/workflow-book/chapters/`. If `TOC.md` changed, read it too and check whether the chapter set itself changed (a chapter added, split, renumbered, or removed) — that is a `new-or-restructure` signal that may require adding, splitting, or dropping a summary section and updating the map table above.

4. **Refresh only the mapped sections.** Using the map, rewrite each affected `TLDR.md` section so its claims match the current chapter. Preserve the summary's structure, its Mermaid diagrams (update a diagram only if the chapter's flow changed), and its voice. Do not rewrite sections whose chapters did not change. On a full rebuild, regenerate the whole file from the chapters, keeping the section order and diagram set below.

5. **Hold the summary's contract.** The summary must remain:
   - **A half-hour read** — roughly 8–10 pages. If a chapter grew, distill harder rather than transcribe; the book holds the detail.
   - **Faithful** — every claim must be true of the current chapters. When the book and a habit disagree, follow the book.
   - **Self-orienting** — keep the "About this summary" note, the five-phase and tier diagrams, the phase-machine and execution diagrams, and the closing "gist" list.
   - **House-style aware** — this is a `docs/` Markdown file, so `.claude/output-styles/house-style.md` applies (BLUF, banned vocabulary, em-dash discipline). Run the `ai-tells` skill over any section you rewrote if in doubt.

6. **Re-stamp.** Set line 1 to the most recent book-source commit reachable from `HEAD`:

   ```
   git log -1 --format='%H' -- \
     docs/workflow-book/README.md docs/workflow-book/TOC.md docs/workflow-book/chapters/
   ```

   Write it as `<!-- book-source-sha: <SHA> -->`. Update the human-readable baseline note in the "About this summary" block to the matching short SHA. (Stamping with the last book-source commit, not raw `HEAD`, keeps unrelated commits from re-triggering a refresh.)

7. **Report and offer to commit.** Summarize which chapters drove the refresh and which sections changed. Do not commit automatically; offer a commit (for example, `Refresh workflow-book TL;DR against <short SHA>`) and let the user confirm. Re-runs with no book change are no-ops (step 2 stops early), so the skill is safe to run on a schedule or before a release.

## Validation

Before reporting done, confirm:

- Line 1 is a well-formed `<!-- book-source-sha: <40 hex> -->` stamp, and the "About" note's short SHA matches it.
- Every numbered section (§1–§13) and the closing "gist" list is present; no section was dropped unless a chapter was removed and the map updated to match.
- Every Mermaid block still parses (fenced ```mermaid, balanced brackets).
- The file is still a single self-contained read of roughly 8–10 pages; if it grew well past that, distill further.
- `git diff --stat docs/workflow-book/TLDR.md` shows changes only in the sections whose chapters moved (plus the stamp), on an incremental run.
