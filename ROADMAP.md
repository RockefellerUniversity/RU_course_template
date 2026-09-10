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
render in a subprocess and measure. If the compute is genuinely heavy rather than
just memory-hungry, jump to item 9 (render on the HPC) instead.

---

## 7. CI hardening

- **Promote non-blocking legs.** `legacy-R-check` R 3.5/3.6/4.0 are
  `allow-failure: true` (documenting the R 4.1 floor). Some now-passing legs
  could become gating for a stronger signal.
- **Per-course `_freeze/` — currently a hole in a documented feature.** The plan
  is that a course commits its `_freeze/` so an edit re-executes only the changed
  session, locally *and* in CI. The engine *honours* a committed freeze (it copies
  `contentDir/_freeze` into the build tree), but the build runs in a `tempfile()`
  directory that is deleted on exit, so the freeze Quarto **writes** during a build
  is discarded — nothing in the pipeline can ever produce or refresh one.
  **Fix:** copy `pathToPres/_freeze` back to `contentDir/_freeze` after rendering.
  Then decide whether to commit it, weighing the payoff (fast rebuilds) against
  git churn — the cached figures are binaries and every re-execution rewrites
  them. Note a committed freeze publishes *whoever rendered it*'s results, which
  is why the canary legs (OS-check, legacy-R-check, the cron) force
  `full-rebuild: true`.
- **Skip pointless rebuilds.** The build workflows have no `paths` filter, so
  every push to master triggers a full render — including the `Autobuild` commit
  the publish step itself pushes (that rebuild produces no changes and exits via
  the "No changes to publish" guard, but still burns a full run). A
  `paths-ignore` for `docs/**` and `**.md` would cut the wasted builds; worth
  doing given the Actions quota pressure we hit.
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

---

## 9. Render heavy courses on the HPC

**What.** A path to run expensive computation on the RU SLURM cluster rather than
a laptop or a ~7 GB GitHub runner.

**Intended model (the target design).** An **HPC render** alongside the existing
local render: on the cluster the course renders with *everything executed* — all
chunks run, intermediate files regenerated — and that run's **`_freeze/` cache is
committed and pushed**. GitHub then never does the heavy work: CI re-renders from
the frozen results, so pushes stay cheap. In short, **HPC is the execution
environment and CI is only a formatting pass.**

**Why.** Genomics material (alignment, counting, large object loads) can't
realistically execute in CI. Today's workaround is the `eval=FALSE` +
`load("data/…")` pattern: heavy results are produced by hand, offline, and
shipped as data files. An HPC path would formalise *where those results come
from* instead of leaving it manual and undocumented.

**Hard constraint.** GitHub-hosted runners **cannot reach the RU cluster** (no
network route, no credentials), so an HPC render can never be a step inside the
Actions build. It has to be an **out-of-band step whose outputs are committed**
(or attached to a release) and then consumed by ordinary builds. Any design has
to start from that.

**Three levels, increasing ambition:**

1. **Whole-course render on the cluster.** Submit `compileSingleCourseMaterial()`
   as a SLURM job using the existing RU tooling (`~/Documents/RU/Analysis/HPC`,
   the `rocky9/` templates, `run_qmd_*` + `Herper::local_CondaEnv`; see the
   `ru-hpc-slurm` skill). Simplest, and sufficient if a course is heavy overall.
2. **Per-chunk offload.** Worth knowing: a chunk *option* alone can't do this —
   knitr has no "evaluate this elsewhere" hook. It needs either a custom engine
   (`knitr::knit_engines$set(hpc = …)`) that submits the chunk body and returns
   captured output, or `future.batchtools` with the rocky9 SLURM templates used
   explicitly inside the chunk. Because the job runs in a **separate R session**,
   such chunks need explicit disk-based inputs/outputs rather than shared
   in-memory state — which is exactly the existing `load()` pattern, so the
   content style already fits.
3. **Wire it to caching.** Have the expensive results land in `_freeze/` (or
   `data/`) so ordinary CI builds never re-execute them. This is the same
   underlying problem as the freeze work in item 7 — compute once, reuse
   everywhere — and the two should be designed together. See also item 6
   (subprocess isolation) for the lighter-weight, same-machine variant.

**Reproducibility note.** Results computed in an HPC conda env differ from CI's
Bioconductor docker; capture `sessionInfo()` alongside them and be explicit about
which environment produced the published output.

**Conditions that make the model work** (each is a real dependency, not a detail):

- **The `_freeze` copy-back is a hard prerequisite.** Today the build runs in a
  deleted `tempfile()` dir, so the cache an HPC render produces is thrown away
  (item 7). Until that's fixed there is nothing to push, and the whole model is
  blocked on it.
- **Editing a source invalidates that document's freeze.** With `freeze: auto`,
  CI will then try to execute the changed session itself — which on a heavy
  course means a slow build or an OOM. The discipline is therefore *content edit
  → re-render on HPC → push*. The failure mode is at least self-announcing (a red
  build tells you the HPC step was skipped). A per-course `freeze: true` would
  stop CI ever executing, at the cost of silently publishing stale results.
- **⚠ It conflicts with the full-rebuild canaries.** `OS-check` and
  `legacy-R-check` pass `full-rebuild: true` always, and the quarterly cron does
  too — that deliberately *deletes* the freeze to prove the code really runs. On
  an HPC-rendered course those legs would attempt the heavy compute in CI and
  fail. Heavy courses will need those canaries disabled, or scoped to a light
  subset. Worth deciding deliberately: it trades away the "does this still
  execute?" signal, which is the main thing those jobs exist for.
- **The HPC run pays double execution.** Quarto's knitr engine executes once per
  output format, and (per the migration finding) freeze did not dedupe across the
  two separate revealjs/html renders — so each session executes twice on the
  cluster. Cheap to accept there, but size the job accordingly.
- **Decide what the regenerated intermediates cost.** If the HPC run remakes
  large intermediate data, committing it feeds straight into the churn problem —
  see item 10 for whether those belong in git, in a release asset, or ignored.

**First step.** Fix the `_freeze` copy-back (item 7), then render one real course
end to end on the cluster with the existing `run_qmd_*` runner, commit the cache,
and confirm a CI build reuses it without executing. That single loop proves or
disproves the whole model before any per-chunk machinery is worth building.

---

## 10. Managing git churn

**The problem.** We commit generated output, so the repo grows monotonically:
- rendered decks and pages are **self-contained** (`embed-resources: true`), so
  each is multi-MB with base64-inlined assets, and *every* rebuild rewrites them;
- an `Autobuild` commit lands on **every push** — including the one the publish
  step itself pushes;
- adopting a committed `_freeze/` (item 7) would add cached figure PNGs that
  churn on every re-execution.

Git keeps every version of all of that, so history grows considerably faster than
the content does. It's tolerable on this template; it's the thing to get right
*before* migrating large, plot-heavy courses.

**Options, cheapest to most structural:**

1. **Cut pointless rebuilds** — `paths-ignore` for `docs/**` and `**.md`
   (item 7). Kills the self-triggered rebuild and doc-only churn. Worth doing
   regardless of what else we choose.
2. **Publish on release, not on every push.** Commit rendered `docs/` only when
   tagging. Churn drops to one commit per release, the committed site always
   corresponds to a released version, and the release-driven RAG ingest (item 2)
   gets exactly what it needs — it also dissolves the "tag after the build"
   ordering trap. Cost: the committed site lags master between releases.
3. **Deploy Pages from Actions** (`actions/deploy-pages`) instead of from a
   committed folder. `docs/` never enters git, so build churn goes to **zero**.
   Cost: a repo download no longer contains ready-made HTML — mitigate by
   attaching a built-site ZIP as a **release asset**, so downloads still get
   rendered material.
4. **Drop `embed-resources` for the decks.** Assets get shared instead of
   base64-duplicated into every file, cutting committed volume substantially.
   Cost: decks stop being individually shareable/self-contained, which was a
   deliberate earlier decision — a real trade, not a free win.
5. **Git LFS** for rendered HTML/figures. Keeps clones lean but adds quota and
   workflow friction; probably not worth it here.
6. **Last resort:** history rewrite or a fresh start if a course repo becomes
   unusable.

**Recommended combination.** Do (1) now. Then (3) for the live site plus
(2)/release assets for downloadable material: that yields an always-current
published site, zero build churn in git, and a clean per-release snapshot serving
both students and the RAG corpus.

**First step.** Measure before optimising — check `.git` size and per-build growth
on this template and on the largest real course, so the decision is driven by
actual numbers rather than instinct.
