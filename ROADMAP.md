# Roadmap — planned and deferred work

Forward-looking plans for the RU course pipeline (`compileCourses` engine +
this template). These are things we designed, discussed or deliberately set
aside during the Quarto migration and the flat/single-tree restructure.

This file is **not** the day-to-day go-live checklist (pushing branches, merging
the open PR, cutting releases) — that's tracked in the PR itself. For migrating
an existing course see [`MIGRATION.md`](MIGRATION.md); for authoring a new one
see [`GettingStarted.md`](GettingStarted.md).

Each item: **what**, **why it's deferred**, and a concrete **first step**.

---

## 1. Interactive exercises / quizzes in the slides and notebooks

**What.** Let learners type and *run* code inside the material — e.g. "write an
expression that returns 2" — with the answer auto-checked. Aimed at intro-level
courses.

**Tool.** [`quarto-live`](https://r-wasm.github.io/quarto-live/) — runs R via
**WebR** and Python via **Pyodide** entirely in the browser (WebAssembly). It
fits this pipeline because it:
- needs **no server**, so it works on GitHub Pages;
- covers **R and Python**, matching the ambidextrous plan;
- provides `exercise` cells with hints, solutions and **grading**, not just a
  runnable console;
- supports both `revealjs` and `html`, which matters because every session
  renders to both from one source.

Rejected alternatives: **`learnr`** (needs a Shiny server — rules out Pages),
**`shinylive`** (overkill for a quiz). For plain multiple-choice with no code
execution, **`webexercises`** is far lighter and worth considering separately.

**Why deferred.** Several unknowns interact with our current engine config:
- **`embed-resources: true`** on decks/single pages vs. WebR loading its runtime
  and assets at page load — likely needs `embed-resources: false` on interactive
  pages (as we already did for the front page).
- **Extension shipping** — Quarto extensions live in `_extensions/`, so the
  engine would need to stage it into the content root (same pattern as the
  `.scss` themes and the Lua filter).
- **WASM package availability** — WebR has WASM builds for much of CRAN but
  **thin Bioconductor coverage**, so this is realistic for intro/base-R teaching,
  not genomics workloads.
- **Payload** — the R runtime is tens of MB on first load.
- **Purl interaction** — exercise cells would land awkwardly in the extracted
  code output and probably need excluding.

**First step.** Spike it: stage the extension from the engine and add one graded
exercise to a session, then render and inspect (a) the deck, (b) the single page,
and (c) the purled code. That answers the embed-resources and dual-format
questions concretely before committing.

---

## 2. RAG corpus — release-driven and versioned

**What.** Ingest the course material into the RAG store (the `ragnar` R package)
so answers are grounded in, and cite, a specific course version.

**Approach — ingest from releases.** Releases are the right unit because the
published site only ever holds the *current* build (each `Autobuild` overwrites
`docs/`), so **prior versions exist only in the repo's tags**. A tag is immutable,
reproducible, and carries the version label intrinsically.

Pipeline: enumerate releases via the GitHub API (the same call the Releases page
makes) → for each tag get the tree (`git worktree add ../v2.0 v2.0`, or the tag
tarball) → ingest that tag's `docs/presentations/singlepage/*.html` and exercise
pages → stamp `version = <tag>` as a **chunk metadata column** (not page text —
you want to filter and cite on it, and HTML→markdown drops `<meta>` tags).

**Design points / gotchas.**
- **Tag *after* the build.** The rendered `docs/` is committed by `Autobuild`; a
  tag cut before that build captures stale HTML against newer sources.
- **Near-duplicate chunks across versions** — most content repeats between
  releases, so retrieval can surface the same paragraph N times. Default queries
  to the latest version and filter on `version`, or retain only the last few
  releases. This is the main quality risk.
- **Unreleased/current state** — between releases the corpus lags the site. If
  RAG should answer about current material, ingest master separately as
  `version: "dev"` and exclude it by default.
- **Coverage** depends on tagging discipline (history starts when tagging did).
- **Optional `manifest.json`** — the engine could emit
  `{course, version, built, commit, sessions}` into `docs/` at build time for
  finer provenance. Largely optional under release-driven ingest, since the tag
  already supplies the version.

**Already done.** `ragnar` handles HTML→markdown, heading-aware chunking (each
chunk carries its heading trail) and source provenance, so no engine-side "RAG
export" is needed. The authoring conventions (headings as chunk boundaries,
self-contained sections, code-with-prose, figure captions, what *not* to ingest)
are written up in `GettingStarted.md` → *Authoring for RAG*.

**First step.** Write the ingestion script against one course with real releases
(e.g. `Intro_To_R_1Day`, which has v1.0/v2.0/v3.0) and check retrieval quality
across versions before generalising.

---

## 3. Migrate the live courses

**What.** Move the real courses (RNAseq, ChIPseq, …) onto the Quarto + flat
single-tree pipeline, one at a time, following `MIGRATION.md`.

**Why deferred.** Deliberate sequencing: finish and prove the template/engine
first, then migrate. Each course is its own effort (content conversion plus the
layout move).

**First step.** Pick one course, work `MIGRATION.md` end to end, and treat any
friction as a fix to the guide.

---

## 4. Final engine cut-over

**What.** Once *all* courses are migrated: merge the engine's `quarto-migration`
branch into engine `master` and strip every `compilecourses-ref: quarto-migration`
pin from the course workflows.

**Why deferred.** The pin is the opt-in mechanism during the phased migration;
engine `master` intentionally remains the **old rmarkdown compiler** so
un-migrated courses keep building.

**First step.** Only after item 3 completes — then a single coordinated PR per
course to drop the pins.

---

## 5. Python / multi-language courses

**What.** Make a Python-based course a first-class citizen.

**State.** The engine was structured to keep the two language-specific seams
swappable, and the front page is already language-neutral (the presentation link
is "Code", not "R code", and "Notebook" links to the `.qmd`). What's still
R-specific:
- **Dependency install** — currently the R package/`DESCRIPTION` model; Python
  needs conda/pip via `Config/reticulate` (the front page already has a Python
  install branch).
- **Code extraction** — `knitr::purl` → `.R`; a Python course wants `.py` or a
  `quarto convert` to `.ipynb`.
- **The Docker tab** — the recipe is the Bioconductor image, so it's R-only. A
  non-R course currently gets a fallback line rather than a container recipe.

**First step.** Define the Python container recipe and the `.py`/`.ipynb`
extraction, then run a small Python course through the pipeline.

---

## 6. Heavy-course robustness (from the original redesign, still open)

**What.** The pre-Quarto redesign proposal (`REDESIGN.md`, since removed from the
repo — see git history) suggested rendering each session in its **own subprocess**
(`callr` / `xfun::Rscript_call`) to bound peak memory and isolate failures. That
part was never implemented — Quarto solved the double-render/text-munging
problems, but not memory.

**Why it matters.** Genomics courses (alignment, counting, large object loads) on
~7 GB CI runners can OOM non-deterministically; today all renders share one R
process, so peak memory is set by the heaviest session and namespaces accumulate.

**Related, accepted for now.** Quarto's knitr engine executes **once per output
format**, so each session renders twice; a committed `_freeze/` makes the second
pass reuse results. Fine for current content, worth revisiting for heavy courses.

**First step.** Only when a real course actually OOMs — then wrap the per-session
render in a subprocess and measure.

---

## 7. CI hardening

- **Promote non-blocking legs.** `legacy-R-check` R 3.5/3.6/4.0 are
  `allow-failure: true` (documenting the R 4.1 floor). Some now-passing legs
  could become gating for a stronger signal.
- **Per-course `_freeze/`.** Commit frozen results once content stabilises, so
  editing one session re-executes only that session locally *and* in CI.
- **Link-check policy.** The check now fails the run on **any** broken link. If
  the known-broken links (item 8) are kept long-term, consider an ignore-list so
  the gate reds only on *new* breakage rather than sitting permanently red.

---

## 8. Content and site cleanups

- **The known-broken links.** `link-check` is red by design right now: 14 dead
  cross-references to `Intro_To_R_1Day/.../introToR_Session1.html` (7 source
  lines in `docs/notebooks/Session1.qmd` and `Session2.qmd`) plus 2 deliberately
  planted `github.com/rafelleruniversity` test links. Fix the targets, remove
  the lines, or ignore-list them — see item 7.
- **Private-repo badges.** The Course Integrity workflow badges are remote SVGs
  that only render for a viewer authenticated to the private repo (or once it's
  public). Not a build problem, but it looks broken to anonymous viewers.
- **Navbar "Notebooks" menu.** The per-session links now offer four outputs
  (Slide / Single Page / Code / Notebook) while the navbar has three menus.
  Adding a Notebooks menu would restore parity.
- **`_course.yml` format.** The space/comma-delimited encoding of sessions and
  exercises is terse and easy to get wrong; a structured list was flagged in the
  original redesign proposal as a lower-risk change to make outside a big
  migration.
- **Duplicate chunk label.** `index.qmd` uses the label `showSysInstall` twice
  (course description child, system-requirements child). It renders today, but
  it's fragile if knitr ever enforces unique labels.
