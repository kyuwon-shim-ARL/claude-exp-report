# claude-exp-report

Claude Code plugin for generating multi-tier experiment synthesis reports from MANIFEST.yaml.

## Install

```bash
claude plugin add git@github.com:kyuwon-shim-ARL/claude-exp-report.git
```

Or add the marketplace to Claude Code:

```bash
claude plugin marketplace add https://github.com/kyuwon-shim-ARL/claude-exp-report.git
claude plugin install exp-report
```

## What It Does

Reads your MANIFEST.yaml, filters `status: final` experiments, and generates a 4-Tier synthesis report:

| Tier | Audience | Content | Lines |
|------|----------|---------|-------|
| **Tier 0** | Non-specialists | Plain Language Summary (jargon-free rewrite) | 20-30 |
| **Tier 1** | Executives | Decision Brief + Decision Table | 20-30 |
| **Tier 2** | Scientists | Evidence Narrative (conclusion-first, 1 figure per question) | 100-150 |
| **Tier 3** | Reviewers | Technical Reference (collapsible specs, cross-reference) | 60-100 |

Outputs: Markdown + self-contained HTML + PDF + deliverable package (ZIP with figures, tier files, and interpretation guide).

## Usage

```
/exp-report:exp-report
```

Or just say "experiment report", "synthesis report", "실험 보고서", or "통합 보고서" — the skill auto-activates.

## Requirements

- MANIFEST.yaml with `status: final` experiments (at least 3)
- Each experiment needs: `description`, `findings`, `path` (alternative field names accepted — see Phase 0 normalization)
- Python packages: `markdown` (HTML conversion), `PyMuPDF >= 1.21` (PDF conversion, requires Story API)

## MANIFEST.yaml Format

```yaml
version: 1
experiments:
  e001:
    path: data/E001/
    status: final
    description: "What this experiment tested"
    findings: "Key results with numbers"
    data_dir: data/E001/results/    # optional: auto-include all data files from this directory
    outputs:
      - data/E001/figures/plot.png
```

## Including Data Files in Deliverables

Figures in `{exp_path}/figures/` are auto-discovered. Data files require opt-in via one of two methods:

**Method 1: List individually in `outputs`** (precise control)
```yaml
e001:
  outputs:
    - data/E001/figures/roc_curve.png
    - data/E001/results.csv           # listed explicitly
```

**Method 2: Set `data_dir`** (directory-level inclusion)
```yaml
e001:
  data_dir: data/E001/results/        # all CSV/TSV/JSON/etc. in this dir
  outputs:
    - data/E001/figures/roc_curve.png
```

**Recommended convention**: Place final data files in a `results/` subdirectory within each experiment, then set `data_dir` to point there. This keeps intermediate files separate from deliverable-ready outputs.

Supported data extensions: `.csv`, `.tsv`, `.json`, `.xlsx`, `.xls`, `.yaml`, `.yml`, `.parquet`, `.feather`, `.npy`, `.npz`, `.pkl`, `.pickle`.

## Features

- MECE-validated question discovery with narrative arc ordering (Claim → Mechanism → Boundary → Practical)
- Parallel tier generation (3+1 agents: 3 parallel + Tier 0 sequential after Tier 1)
- Cross-tier numeric verification (Tier 1 numbers must appear in Tier 0 and Tier 2)
- MANIFEST field normalization (`title`→`description`, `result`→`findings`, auto-derive `path`)
- Auto language detection (Korean/English) from MANIFEST content
- Designer readability review on HTML output (CSS-extraction pattern) — skip with `--fast`
- **Deliverable packaging**: Bundles report + original figures + interpretation guide into a shareable folder + ZIP
- Auto-generates `md_to_html.py` and `md_to_pdf.py` if project lacks them
- Updates MANIFEST.yaml and experiment-log.md automatically
- See `examples/MANIFEST.yaml` for a reference MANIFEST with edge cases

## Compatibility

- Works standalone (no dependencies on other plugins)
- Enhanced with oh-my-claudecode if installed (uses specialized designer agent)

## License

MIT
