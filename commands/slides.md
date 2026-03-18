---
name: slides
description: Generate reveal.js HTML slide deck from an existing exp-report workspace
---

# /exp-report:slides — Slide Deck Generation

Generate a self-contained reveal.js HTML slide deck from an existing report workspace.

## What It Does

1. Reads tier files (Tier 0/1/2/3) and figure registry from `data/F{NNN}/`
2. Maps content to 6 slide design patterns (KeyFinding, Comparison, Dashboard, EvidenceNarrative, Summary, Appendix)
3. Generates self-contained HTML with inlined CSS/JS (no CDN dependency, works offline)
4. Applies 2-iteration designer critique loop for presentation-quality CSS
5. Generates dual-language slides (EN/KO) if the workspace has dual-language tier files
6. Integrates with existing deliverable folder and ZIP

## Usage

```
/exp-report:slides                        # Auto-detect latest F{NNN} workspace
/exp-report:slides data/F004/             # Specify workspace path
/exp-report:slides --fast                  # Skip designer review
/exp-report:slides data/F004/ --fast       # Specify path + skip design
```

## Requirements

- A completed exp-report workspace (`data/F{NNN}/`) with tier files and registries
- Run `/exp-report:exp-report` first if no workspace exists

## Output

```
data/F{NNN}/
  F{NNN}_slides.html           # Primary language slide deck
  F{NNN}_slides_en.html        # English slides (if dual_lang)
  F{NNN}_slides_ko.html        # Korean slides (if dual_lang)
```

If `deliverable/` folder exists, slides are automatically copied in and GUIDE.md is updated.

## Slide Structure

- **Title** (1 slide) — Report title + experiment count
- **Motivation** (1-2 slides) — Tier 0 opening, KeyFinding pattern
- **Methods** (1-2 slides) — Tier 2 methods overview
- **Results** (1 per synthesis question) — Tier 2 evidence, EvidenceNarrative pattern
- **Decision Summary** (1 slide) — Tier 1 KPIs, Dashboard pattern
- **Conclusion** (1-2 slides) — Tier 0 closing + recommendations, Summary pattern
- **Appendix** (1 per experiment) — Tier 3 technical specs

## Features

- Fully self-contained HTML — works offline, no external dependencies
- Base64-embedded figures from figure registry
- Arrow key / click navigation
- Speaker notes (press `S`) with full tier content
- PDF export via `?print-pdf` in Chrome
- Korean CJK typography with optimized density rules
- 2-iteration designer CSS review for presentation quality

## Invoke the Skill

This command activates the `slides` skill. Read the full skill documentation for details on the 6-phase workflow, slide patterns, and density rules.
