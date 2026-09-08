# Eastern Bering Sea snow crab — September 2026 SAFE

Assessment code and SAFE report for eastern Bering Sea snow crab (*Chionoecetes opilio*).
Authors: Grant Adams and Cody Szuwalski, Alaska Fisheries Science Center.

The 2026/27 harvest specification is calculated under **Tier 4** from a random-effects (REMA) smooth
of the NMFS summer trawl survey, following the October 2025 and June 2026 SSC recommendations that
Tier 4 be used until the Tier 3 model's convergence problems are resolved. Six Tier 3 GMACS models
are presented to document progress; none is proposed for specification this cycle.

> The assessment document in this repository is a **draft**. It carries the NOAA pre-dissemination
> disclaimer and has open author decisions marked `[[author]]`. Do not cite it as final advice.

---

## Repository layout

| Path | Contents |
|---|---|
| `2026_snowcrab_safe_draft.Rmd` | The SAFE document. All numbers are computed at knit time. |
| `scripts/` | Numbered pipeline, `00`–`08`. Run from the repository root. |
| `R/` | Shared function libraries (`gmacs_io.R`, `gmacs_jitter.R`). Nothing runs on `source()`. |
| `Models/` | One directory per GMACS model: data, control and projection files, and the fit. |
| `data/derived/` | The six-file interface between data preparation and the model. |
| `data/tier4/` | REMA fit and the Tier 4 reference points. |
| `data/historical/` | Specification history read from past SSC reports. |
| `Reports/` | Reference documents — past SAFEs, SSC and CPT reports, FMP and rebuilding text. |
| `plots/` | Generated figures. |
| `docs/` | Conventions, backlog, porting notes, report index. |
| `GMACs/` | GMACS source trees used to build the executables. |
| `archive_2025/` | Previous cycle, retained for reference. |

`Reports/` filenames follow `YYYY-MM_what.pdf`, dated by the document's own date, and are indexed in
`docs/REPORTS_INDEX.md`.

---

## Running the assessment

R 4.5 or later. Requires `gmacsr`, `wtsGMACS`, `crabpack`, `rema`, `flextable`, `officedown`,
`bookdown` and a LaTeX installation (TinyTeX is sufficient).

Always run from the repository root; scripts resolve paths relative to it.

```r
Rscript scripts/01_prep_fishery_data.R      # ADF&G removals + NORPAC -> data/derived/
Rscript scripts/02_prep_survey_data.R      # crabpack survey pull -> comps, indices, ogive
Rscript scripts/00_advance_model.R <template_dir> <out_dir> <end_year> <dat_name>
#   ... then run gmacs in the model directory, to convergence ...
Rscript scripts/03_build_results_object.R  # model directories -> Models/rda_ModelsResLst.RData
Rscript scripts/05_run_jitter.R            # starting-value sensitivity
Rscript scripts/06_run_retrospective.R     # peels and Mohn's rho
Rscript scripts/07_calc_tier4.R            # REMA fit and Tier 4 reference points
Rscript scripts/08_render_report.R         # -> PDF   (add `docx` or `both` for Word)
```

Two ordering points are easy to get wrong:

- **`00_advance_model.R` runs third**, despite its number. It consumes the output of `01` and `02`.
- **The jitter (`05`) runs before the retrospective (`06`).** A jitter can find a better optimum and
  promote it into the model directory, which invalidates any retrospective computed against the
  previous fit.

`scripts/0-models.R` defines the model set, short names and display order. Case names must match
across `0-models.R`, `03_build_results_object.R` and the Rmd.

---

## Data

`data/derived/` is the contract between data preparation and the model: six tidy files, each with a
header row, an explicit `year` column, and values already in model units.

`directed_catch.csv` · `bycatch_catch.csv` · `fishery_size_comps.csv` · `survey_size_comps.csv` ·
`survey_indices.csv` · `male_maturity_ogive.csv`

Changing this schema means changing `scripts/00_advance_model.R` as well.

**Raw confidential inputs are not distributed here.** NORPAC observer records
(`data/norpac_catch_report/`, `data/norpac_length_report/`) and the ADF&G removals snapshots
(`data/adfg_removals/`) are excluded. Obtain them from AKFIN and ADF&G and place them at those paths
before running `01`. Everything downstream of `01` and `02` is included, so the assessment can be
reproduced from `data/derived/` without them.

Survey data are pulled from the `crabpack` API at run time. Note that `02_prep_survey_data.R` and
`07_calc_tier4.R` both carry a hardcoded year range; advance them together.

---

## Conventions

Read `docs/assessment_conventions.md` before editing. It carries the rules that keep this repository
reproducible and the known traps that have already cost time — among them that GMACS returns
reference points as exactly zero for retrospective peels and for any `-nohess` run, and that a
freshly built model directory still contains the template's fit until the model is run.

Two conventions matter for reading any number in this repository:

- **Crab year.** End year *N* means the *N*/*N*+1 fishery plus the *N*+1 summer survey. Fishery data
  run through `END_YEAR`, survey data through `END_YEAR + 1`.
- **2020 has no survey.** The survey was cancelled; the gap is handled explicitly and never
  interpolated across.
