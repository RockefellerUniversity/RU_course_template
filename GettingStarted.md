# Teaching at RU

A template repository from which to build workshops and other teaching materials
in a standard manner, so they integrate with other RU material. An example can be
found [here](https://rockefelleruniversity.github.io/RU_RNAseq/).

This guide is for **authoring a new course from this template**. To migrate an
existing (older, package-based) course onto this pipeline, see
[`MIGRATION.md`](MIGRATION.md).

## How to use this template

* Click **'Use this template'** at the top right of the repository to create your
  own course repository.

* All course content lives in the top-level **`docs/`** folder — the single tree
  that holds both your **sources** and the **rendered site**. GitHub Pages serves
  the rendered site from `docs/` (Settings → Pages → *master / docs*), and the
  GitHub Actions in `.github/` recompile it whenever you push.

* You do not rename or manage an R package. The repo root holds only a thin
  `DESCRIPTION` (your dependencies) plus the `.github/` workflows; everything you
  edit as an author is under `docs/`.

* To set up your course you mainly: fill in the content sources under `docs/`,
  and update the two config files (`docs/_course.yml` and the root `DESCRIPTION`).
  There are placeholders, examples and formatting guides throughout. Anything
  surrounded by double question marks **`[??]`** is helper text to be replaced.

* **Edit sources, not generated files.** Under `docs/` you edit the `.qmd`/`.Rmd`
  sources, `data/`, `imgs/` and `_course.yml`. Everything else in `docs/` is
  regenerated on each build — do not edit it by hand:
  `presentations/{slides,singlepage,r_code}/`, `exercises/{answers,exercises}/`,
  `index.html`, `releases.html`, `search.json`, `site_libs/`, and the root
  `README.md` (which is generated from your Course Overview).

* If you find mistakes, or parts of the template that could be clearer, please
  fork and submit a pull request, or raise an issue.

* To see how the template compiles, visit the
  [web page](https://rockefelleruniversity.github.io/RU_course_template/), or look
  at other [compiled courses](https://rockefelleruniversity.github.io/RU_RNAseq/).

## Course content

Content is authored as **Quarto** documents (`.qmd`). Each presentation renders
from one source to slides, a single page and downloadable R code.

#### Course slides
Find these at **`docs/presentations/*.qmd`**.
Check out `Session1.qmd` for formatting (slide breaks with `---`, two columns,
section-divider slides, `eval=FALSE` + `load()` for heavy chunks, etc.).

#### Exercises
Find these at **`docs/exercises/*.qmd`**.
Check out the example for how the `toMessage` parameter shows/hides solutions
(answers vs exercise versions).

## Config files

#### DESCRIPTION
Find this at the repo root: **`DESCRIPTION`**.
A thin dependency manifest — list only the packages your **course content** uses
(it is installed just to resolve dependencies and to build the front-page install
instructions; the rendering engine supplies its own build tooling). Do **not**
add build-only packages such as `rmarkdown`/`knitr` here.

If you have non-R dependencies, put them in the `SystemRequirements` field using
the name of the conda package. We use Herper to install that software.

#### Descriptions
Find these at **`docs/descriptions/`**.
These files contain the descriptive text the cover page is built from. There are
two kinds:

1. **Course Overview** (`CourseOverview.Rmd`) — a description of the overall
   course. This section also becomes the repo's top-level `README.md`.
2. **Session overview** (`SessionNOverview.Rmd`) — one per session you break the
   course into (there can be just one for a single-session course).

#### _course.yml
Find this at **`docs/_course.yml`**.
Lists the names of all the content files. Update the `CourseName` first. If any
file is renamed from the template, update the `.yml` to match. Order matters, so
your first session's `.qmd` should be first. Entries are space-separated, except
exercises: exercises are comma-separated within a session and space-separated
between sessions (e.g. in the template the first two exercises belong to the first
session and the third to the final session).

## .github files for compiling

The compilation workflows live in the top-level `.github/` directory. They detect
pushes, recompile the content into `docs/`, and run basic checks of the R code.
There are three, plus a link checker:

1. **compilation-check** — builds the course (rendering in place into `docs/`) and
   publishes it; GitHub Pages serves `docs/`.
2. **OS-check** — checks the course compiles across macOS, Windows and Ubuntu
   Linux (R release and devel).
3. **legacy-R-check** — attempts to compile on every major R release since 3.5
   (Linux only), documenting the R-version compatibility floor.
4. **link-check** — after a successful build, verifies the site's links (local
   files/anchors + remote reachability) and reports broken ones.

The shared install-and-compile logic lives in the composite action at
`.github/actions/compile-course`, which the build workflows call. During the
migration period the workflows pin the Quarto rendering engine with
`compilecourses-ref: quarto-migration` — keep that pin until the engine is cut
over to its default branch. Unlike the old template, there is no package name to
replace in these scripts (they read the content from the repo directly). Most
standard courses work as-is; occasional customization is course-specific.

## Finished?

Once you have finished, let us know and we will take a fork onto the Rockefeller
GitHub, start the process of getting it compiling, and help review the content.

## Help

If you need help, contact the BRC [brc@rockefeller.edu], or raise an issue.
