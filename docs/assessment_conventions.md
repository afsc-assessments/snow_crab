# Assessment conventions — EBS snow crab, September SAFE

Working rules for anyone editing this repository. This assessment sets federal OFL/ABC:
a wrong number here becomes a wrong regulation, so accuracy comes before speed.

Companion documents: `README.md` (layout and how to run), `docs/SESSION_HANDOFF.md`
(current state and blockers), `docs/CLEANUP_BACKLOG.md` (known debt).

---

## Rules

1. **Never invent a number.** Every quantity in the SAFE comes from a model run, a derived CSV, or
   a cited source. Not computed yet? Write `NA` and flag it. A plausible placeholder that survives
   to print is the worst failure mode this repo has.
2. **Never hand-edit a model `.DAT`/`.CTL`.** Use `scripts/00_advance_model.R`. And never
   `readLines()`/`writeLines()` a GMACS file — they silently rewrite the CRLF line endings and
   non-ASCII comment glyphs those files carry. Use `read_raw_lines()` / `write_raw_lines()` from
   `R/gmacs_io.R`.
3. **Refactors that touch numbers must be proven identical.** Save the outputs, re-run, diff, state
   the diff in the commit message. "Looks right" is not verification.
4. **cwd is the repo root, always.** Repo-relative forward-slash paths. No `setwd()` outside the
   peel/jitter loops — and those need `on.exit(setwd(orig_wd))` so a GMACS crash doesn't strand the
   session inside a peel folder.
5. **One source of truth per quantity.** OFL, ABC, `ABC_buffer`, MMB and M come from the shared
   scalars defined once in the Rmd setup chunk. Never recompute one locally in a chunk — that is how
   two tables in one document end up disagreeing.
6. **`[[author]]`, `[[TODO]]` and `[[VERIFY]]` markers are the lead author's calls.** Surface them;
   do not guess them.
7. **Commit messages:** subject <= 72 characters, imperative mood. The body says *why*, and gives
   the numbers that changed.
8. **Do not edit `Reports/`** (reference documents, including the previous SAFE), **`archive_2025/`**
   (prior cycle) or **`data/adfg_removals/`** (immutable dated ADF&G snapshots).
9. **Opportunistic cleanup only.** Fix the file you were already editing, in the same commit.
   Everything else becomes a line in `docs/CLEANUP_BACKLOG.md`.
10. **Model runs and renders are consequential.** A GMACS run, a 10-peel retrospective, a 100-run
    jitter or a knit takes real time and overwrites artifacts in place.
11. **Never widen GMACS parallelism.** Each worker is a full ADMB process holding a core at 100%.
    A wide fan-out draws more sustained power than a laptop chassis can shed, and the machine can
    hard-reset mid-run, corrupting the peel or jitter directory being written. Worker counts come
    from `gmacs_max_workers()` (`R/gmacs_io.R`), default **4**. Never substitute `detectCores()`.
    To raise it for one session: `GMACS_MAX_WORKERS=8`.

---

## R style

**Banner comments.** File header states purpose / inputs / outputs / terminal-year knobs / notes;
the body is split by numbered section rules. Copy the shape from `R/gmacs_io.R` or
`scripts/00_advance_model.R`.

**Match the file you are in.** `01`/`02`/`04`/`06` are dplyr + `%>%`. `00`/`03`/`05`/`07`/`R/` are
base R. Do not mix idioms within a file.

- Plots: `ggplot2` + `theme_bw()`, written as `png("plots/<name>.png", ...); print(p); dev.off()`.
- CSV: `read.csv()` / `write.csv(..., row.names = FALSE)`.
- Names: snake_case verb-first functions (`read_raw_lines`, `bin_to_model_sizes`), `is_*` for
  predicates, leading dot for internal helpers, SCREAMING_SNAKE for constants (`END_YEAR`).
- Assertions: `stopifnot()` with a named message, wherever a silent wrong answer is possible —
  bin counts, row alignment, year-range agreement between files.
- Never write output to the repo root. Use `data/derived/` or `plots/`.

**Comments are for a fisheries scientist, not a programmer.** Explain the assessment reason, not the
R. Give units and the year convention. Date and attribute non-obvious decisions. One or two lines;
anything longer belongs in `docs/`.

```r
# Bad — narrates the code
bin_edges[length(bin_edges)] <- 999   # set last element to 999

# Good — states the assessment reason
# Top edge 999 folds all crab >132.5 mm into the plus group. Without it the
# comps silently DROP large crab (this was a real bug, fixed 2026-07).
bin_edges[length(bin_edges)] <- 999
```

This terse style applies to code comments and project docs. **It does not apply to SAFE narrative
prose**, which matches the register of the previous SAFE in `Reports/`.

---

## Domain conventions

- **Crab year.** End year N = the N/N+1 fishery **plus the N+1 summer survey**. Fishery data through
  `END_YEAR`; survey data through `END_YEAR + 1`.
- **Size bins.** 22 bins, `seq(27.5, 132.5, by = 5)` midpoints, columns named `m27.5` … `m132.5`.
- **Binning: `right = FALSE` everywhere.** A crab on a 5-mm cutoff goes to the **upper** bin. The
  plus group is a top edge of 999 and is independent of `right=`.
- **Comps sum to 1.** Every row, every file.
- **2020 has no survey** (COVID). Handle the gap; never interpolate across it silently.
- **M = 0.27** yr⁻¹ base mature-male.
- **Currency:** morphometric maturity is the recommendation; >= 95 mm and > 101 mm are shown for
  comparison.

---

## Known traps

- **Reference points need the Hessian.** `-nohess` skips ADMB's sd phase, so BMSY/Fmsy/Fofl/OFL come
  back as exactly `0.0` — not missing, *zero*. A run that used `-nohess` must never reach a SAFE
  table or figure.
- **Peels have no reference points, and this is GMACS, not the pipeline.** `gmacsbase.TPL` 2.20.34
  gates both reference-point call sites on the peel count (`nyrRetroNo == 0`), so every run with
  `nyrRetro > 0` returns all 18 derived quantities as exactly 0 in value. The retrospective
  therefore reports MMB and recruitment only. This applies to the 2025 assessment too.
- **A freshly built model directory already contains the TEMPLATE's results.**
  `scripts/00_advance_model.R` regenerates `out_dir` by copying `template_dir`, which brings the
  template's `gmacs.par`, `gmacs.std`, `Gmacsall.out`, `gmacs.rep` and `Gmacsall.std` with it, and
  nothing marks them stale. Check `gmacs.par`'s mtime against the run before believing any number
  from a new model directory.
- **A stale model directory yields a plausible table, not an error.** The Rmd substitutes `NA`/`0`
  for missing models. Before trusting a model directory, check `Year_range` in `Gmacsall.out` and
  the datafile named in `gmacs_files_in.dat` — a `.dat` filename from the wrong cycle is the tell.
  For the 26 model, a B_MSY near 178 rather than ~150 identifies a stale directory.
- **`scripts/07_calc_tier4.R` and `scripts/02_prep_survey_data.R` both pull crabpack with a
  hardcoded year range.** They must be advanced together, by hand.
- **After a crash during `05`/`06`,** a half-written peel or jitter directory still looks plausible.
  Re-run with `--force` (05) or delete the affected `retro/<n>` / `jitter/<nnn>` directory rather
  than resuming onto it.
- **`05_run_jitter.R` writes six shared paths** (`Models/rda_jitter.RData` and five plot names) in
  addition to its per-model outputs, and only for the report model. A jitter of any other model must
  not be allowed to leave its results at those shared paths.
- **Underscores in chunk labels break the PDF.** bookdown writes the label into the caption as
  `(\#tab:name)`, where `_` is a LaTeX math character. Use hyphens. Word wants the opposite, so
  `scripts/08_render_report.R` rewrites the bookmarks in the `.docx` after rendering.
- **`flextable::set_table_properties()` replaces properties, it does not merge them.** A second call
  silently discards `opts_pdf` set by the first. Set `layout` and `opts_pdf` in one call.
