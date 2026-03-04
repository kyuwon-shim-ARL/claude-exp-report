# claude-exp-report

Claude Code plugin for generating 3-Tier experiment synthesis reports from MANIFEST.yaml.

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

Reads your MANIFEST.yaml, filters `status: final` experiments, and generates a 3-Tier synthesis report:

| Tier | Audience | Content | Lines |
|------|----------|---------|-------|
| **Tier 1** | Executives | Decision Brief + Decision Table | 20-30 |
| **Tier 2** | Scientists | Evidence Narrative (conclusion-first, 1 figure per question) | 100-150 |
| **Tier 3** | Reviewers | Technical Reference (collapsible specs, cross-reference) | 60-100 |

Outputs: Markdown + self-contained HTML + PDF.

## Usage

```
/exp-report:exp-report
```

Or just say "experiment report", "synthesis report", "3-tier report", or "통합 보고서" — the skill auto-activates.

## Requirements

- MANIFEST.yaml with `status: final` experiments (at least 3)
- Each experiment needs: `description`, `findings`, `path`
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

- Hybrid question discovery (auto-suggests from metadata, user confirms)
- Parallel tier generation (3 agents)
- Cross-tier numeric verification (every Tier 1 number must appear in Tier 2)
- Designer readability review on HTML output
- Auto-generates `md_to_html.py` and `md_to_pdf.py` if project lacks them
- Updates MANIFEST.yaml and experiment-log.md automatically

## Compatibility

- Works standalone (no dependencies on other plugins)
- Enhanced with oh-my-claudecode if installed (uses specialized designer agent)

## License

MIT
