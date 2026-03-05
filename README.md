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
    outputs:
      - data/E001/figures/plot.png
```

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
