# Course pipeline redesign — proposal

**Status:** draft for review · **Scope:** `compileCourses` engine + `RU_course_template` · **Depends on:** nothing (stands alone on rmarkdown) · **Precedes:** the Quarto migration

This document proposes a structural redesign of how courses are laid out and
compiled. It is deliberately **independent of the Quarto migration** so it can
be reviewed and adopted on its own merits; a closing section notes how it sets
up the Quarto stage. It does **not** cover the already-completed cleanup work
(workflow modernisation, dead-code removal, CI/correctness fixes).

---

## 1. Why change

The current pipeline (`compileSingleCourseMaterial()` in
`compileCourses/R/compileCourses.R`) has four structural problems that will get
worse as courses grow into heavier bioinformatics material:

1. **Double execution.** Every presentation is rendered twice (xaringan slides +
   single-page HTML) and every exercise twice (answers + exercise). Expensive
   chunks (alignment, counting, genome/object loading) therefore run 2×, with no
   result reuse. Exactly the heavy courses pay this cost twice.
2. **Fragile single-page derivation.** The single page is produced by textually
   stripping xaringan markup (`---`, `.pull-left[`, `.pull-right[`, `  ]`) with
   positional code-fence pairing. This silently corrupts output on unbalanced
   fences, `---` used in prose, or whitespace variation — and the **downloadable
   `.R` is purled from this mangled copy**, so broken markup leaks into student
   code. (We have since made it *fail loudly*, but the fragility is inherent.)
3. **Content buried in the package + fan-out duplication.** All content lives
   under `inst/extdata/{presRaw,data,imgs,customCSS,Descriptions}` and
   `inst/doc/`, is installed into the R library, then copied into the build
   tree. Images are duplicated into **five** output folders; data can live in
   multiple places. This is the "multiple data folders" redundancy raised in
   review.
4. **Unbounded memory.** All renders run sequentially in one R process with no
   isolation and no chunk caching, so peak memory is set by the single heaviest
   render and package namespaces accumulate. On ~7 GB CI runners, genomics
   renders will OOM non-deterministically.

## 2. Goals / non-goals

**Goals**
- Content authored at (or near) the **top level**: one `presentations/` (or
  `slides/`), one `exercises/`, one `data/`, one `imgs/`.
- Each source rendered **once**, or with results reused, so heavy chunks execute
  once.
- Remove the text-munging step entirely; derive outputs from a clean source.
- **Bound memory** and isolate failures between sessions.
- Keep dependency declaration (a `DESCRIPTION`) so the front page can keep
  generating install instructions and CI can keep resolving deps.

**Non-goals (for this redesign)**
- Switching the renderer to Quarto (separate, later stage).
- Changing course *content* or the three published output types (slides,
  single page, code) — the contract to students is unchanged.

## 3. Proposed structure

Separate **content** (flat, top-level) from the thin **package** (dependencies
only):

```
<course-repo>/
  DESCRIPTION            # dependencies + SystemRequirements ONLY (no content)
  _course.yml           # course config (as today)
  presentations/        # the raw session sources (was inst/extdata/presRaw)
  exercises/            # exercise sources (was inst/doc)
  descriptions/         # course/session overview text (was .../Descriptions)
  data/                 # ONE data folder, referenced by relative path
  imgs/                 # ONE image folder, referenced by relative path
  css/                  # course-specific CSS (shared themes stay in engine)
  r_course/             # build output (unchanged externally)
  docs/                 # GitHub Pages mirror (unchanged externally)
```

Key change: the engine reads content **directly from these top-level folders**
instead of `system.file()` on an installed package, and outputs **reference a
single shared `imgs/` and `data/`** by relative path rather than copying them
per output. The package is installed only to pull in R dependencies.

## 4. Proposed compile model

Replace `compileCourseMaterial()`'s render-twice-then-munge with:

- **Author once, derive many.** Options, in order of preference:
  - **(A) Single knit + two output formats.** Knit each session **once** to a
    cached intermediate, then write both the slide and single-page HTML from the
    *same* knitted results (knitr caching / a shared `knit()` + two
    `pandoc`/format passes). Heavy chunks run once; the single page is a
    formatting of the same output, not a text hack. The `.R` is purled from the
    **original source**, not a derived copy.
  - **(B) If a true single-knit proves impractical on xaringan**, at minimum
    enable a **shared knitr cache** keyed on chunk content so the second render
    reuses the first's computation. Removes double *compute* even if it keeps
    double *render*. Still drop the text-munging by authoring slide separators
    in a way both formats accept.
- **Process isolation + memory bound.** Render each session in its **own
  subprocess** (`callr::r()` / `xfun::Rscript_call`). Peak memory becomes one
  session at a time, and a crash/namespace clash in one session no longer takes
  down the whole build.
- **No per-output asset copies.** Point every output at the shared `imgs/` /
  `data/` (relative links or a single copy into `r_course/`), eliminating the
  five-way image duplication.

## 5. Compatibility & migration

- **Existing courses** (Intro_To_R, RU_RNAseq, …) all use the package/`inst`
  layout. Provide a one-off **migration script** that moves
  `inst/extdata/presRaw → presentations/`, `inst/doc → exercises/`,
  `inst/extdata/{data,imgs,Descriptions,customCSS} → {data,imgs,descriptions,css}/`
  and trims `DESCRIPTION`.
- **Engine** supports the new layout; keep a compatibility shim for the old
  `inst`-based layout for one release so courses migrate incrementally.
- **External contract unchanged:** `r_course/` / `docs/` structure and the
  navbar links stay the same, so published URLs don't break.

## 6. Risks / open questions

- Can xaringan slides and the single page genuinely share one knit, or is (B)
  (shared cache) the realistic ceiling on rmarkdown? (This is precisely what
  Quarto removes — see §7.) **Decision needed.**
- Chunk caching correctness for chunks with side effects / large objects — cache
  invalidation strategy and on-disk cache size for big data.
- Subprocess rendering changes how `knit_root_dir` and relative `data/` paths
  resolve; must be validated against the current data-loading pattern.
- Whether to keep `_course.yml`'s space/comma-delimited format or move to a
  structured list (lower-risk to do here than during Quarto).

## 7. Relationship to the Quarto migration

This redesign is the **structural foundation** the Quarto stage slots into:

- Top-level content + single `data/`/`imgs/` + a `DESCRIPTION`-for-deps map
  almost directly onto a Quarto project (`_quarto.yml` + top-level `.qmd`).
- Quarto makes the hard part of §4 **native**: one `.qmd` renders to `revealjs`
  slides *and* `html` *and* extractable code from a **single execution**,
  deleting the double-render and the text-munging outright.

So the recommended sequencing is: **adopt this structure/compile redesign first
(still rmarkdown)** to de-risk the layout and memory changes independently, then
**swap the renderer to Quarto** on top of the already-flat structure. If instead
we want to do the structure change *and* the renderer swap together, this doc
becomes the structure half of the Quarto plan.

## 8. Suggested phasing

1. Engine: add top-level-layout support + subprocess rendering + shared assets
   (behind a flag), keeping the old path working.
2. Template: convert `MyCoursePackage` → flat layout; prove parity of `r_course/`.
3. Remove double-execution (approach A or B) and delete the munging.
4. Migration script + convert live courses one at a time.
5. (Later) Quarto renderer swap.
