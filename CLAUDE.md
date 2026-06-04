# dev_utils_for_cosmograph — agent guide

**Dev-only** tooling that *creates and diagnoses* the Single Source Of Truth (SSOT) for Cosmograph's
visualization parameters. It is **not** an end-user data-prep library. Because the JS side has no
single formal SSOT, this reconciles several sources into one Python-facing param SSOT.

## What it does
`ConfigsDacc` (`dev_utils_for_cosmograph/_resources.py`) merges, per-characteristic:
- defaults ← cosmos `variables.ts`; types ← 5 TS config files; descriptions ← cosmograph `configuration.mdx`;
  names+annotations ← Python traitlets — into `params_ssot.json` (consumed by the `cosmograph` package).

## Key files
- `_resources.py` — `ConfigsDacc` (SSOT generator), `ResourcesDacc` (color table), source-URL maps.
- `data/config_prep/*.json` — cached parser outputs (`parsed_defaults/types/descriptions`, `source_strings`).
- `_color_util.py` — `resolve_colors(...)`: reusable color column/value → per-row colors (genuinely useful).
- `_code_sync.py` — order params, build signatures, inject code. `_traitlets_util.py` — trait→type.
- `misc/defining_ssot_for_config_properties.ipynb` — the de-facto tutorial.

## Gotchas
- **Offline regeneration is broken**: `parsed_defaults` needs `esprima` (via `jy`); AI parsers need
  `oa` + network + `GITHUB_TOKEN`. The **cached artifacts** work offline; the live refresh doesn't.
- For *consuming* the SSOT, read `py_cosmograph/cosmograph/data/params_ssot.json` (or
  `cosmograph.util._params_ssot()`) — **not** this repo. This repo is how that file gets (re)built.
- A near-duplicate `ConfigsDacc` also lives in `py_cosmograph/cosmograph/_dev_utils/` — the two have
  diverged; reconciling them is an open task.
- `setup.cfg` declares no deps; real deps (dol, i2, pandas, cosmograph, +optional jy/oa/hubcap/ju) are implicit.

## Maintenance tracker
SSOT-maintenance tasks are filed as **issues in this repo** (see the Epic). This is a **public** repo:
never put absolute local paths, secrets, or personal info in issues/PRs/commits.

## Broader context
See the `cosmograph-ssot` skill in the parent `c/` workspace (`c/.claude/skills/`) for how the SSOT is
consumed across the ecosystem.
