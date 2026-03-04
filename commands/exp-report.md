---
name: exp-report
description: Generate a synthesis report from MANIFEST.yaml experiments with plain language summary
---

# /exp-report — Experiment Synthesis Report

Generate a synthesis report from all `status: final` experiments in MANIFEST.yaml.

## What It Does

1. Reads MANIFEST.yaml and filters `status: final` experiments
2. Auto-suggests 4-6 synthesis questions with MECE validation and narrative arc ordering (Claim → Mechanism → Boundary → Practical)
3. Asks user to confirm/edit questions (shows arc labels and experiment assignments)
4. Generates 4 tiers in parallel:
   - **Tier 0**: Plain Language Summary (~20-30 lines, jargon-free rewrite for non-specialists)
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
  F{NNN}_report.md             # Assembled report
  F{NNN}_report.html           # Self-contained HTML with embedded images
  F{NNN}_report.pdf            # PDF version
  tier0_plain_language.md      # Tier 0 source (jargon-free)
  tier1_decision_brief.md      # Tier 1 source
  tier2_evidence_narrative.md  # Tier 2 source
  tier3_technical_reference.md # Tier 3 source
```

## Invoke the Skill

This command activates the `experiment-report` skill. Read the full skill documentation for details on the 7-phase workflow, agent dispatch table, and verification protocol.
