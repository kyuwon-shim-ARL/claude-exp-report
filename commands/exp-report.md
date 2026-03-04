---
name: exp-report
description: Generate a 3-Tier synthesis report from MANIFEST.yaml experiments
---

# /exp-report — 3-Tier Experiment Synthesis Report

Generate a 3-Tier synthesis report from all `status: final` experiments in MANIFEST.yaml.

## What It Does

1. Reads MANIFEST.yaml and filters `status: final` experiments
2. Auto-suggests 4-8 synthesis questions from experiment metadata
3. Asks user to confirm/edit questions
4. Generates 3 tiers in parallel:
   - **Tier 1**: Decision Brief (~20-30 lines, executive summary + decision table)
   - **Tier 2**: Evidence Narrative (~100-150 lines, conclusion-first per question, 1 figure each)
   - **Tier 3**: Technical Reference (~60-100 lines, collapsible specs + cross-reference)
5. Assembles into single report, verifies numeric consistency across tiers
6. Converts to HTML + PDF with designer readability review
7. Updates MANIFEST.yaml and experiment-log.md

## Usage

```
/exp-report:exp-report
/exp-report:exp-report [path/to/MANIFEST.yaml]
```

## Requirements

- MANIFEST.yaml with at least 3 `status: final` experiments
- Each experiment needs `description`, `findings`, and `path` fields
- Python with `markdown` package (for HTML conversion)
- PyMuPDF >= 1.21 / `fitz` package (for PDF conversion, requires Story API)

## Output

```
data/F{NNN}/
  F{NNN}_report.md          # Assembled 3-tier report
  F{NNN}_report.html        # Self-contained HTML with embedded images
  F{NNN}_report.pdf         # PDF version
  tier1_decision_brief.md   # Tier 1 source
  tier2_evidence_narrative.md  # Tier 2 source
  tier3_technical_reference.md # Tier 3 source
```

## Invoke the Skill

This command activates the `experiment-report` skill. Read the full skill documentation for details on the 7-phase workflow, agent dispatch table, and verification protocol.
