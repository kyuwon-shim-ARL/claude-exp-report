---
name: slides
description: Generate reveal.js HTML slide deck from an existing exp-report workspace (data/F{NNN}/). Activate when user mentions "slides", "슬라이드", "presentation", "발표 자료", or requests slide generation after report creation.
---

# Slide Generation Skill

## CRITICAL: Prerequisites

**This skill requires a completed exp-report workspace.** It reads tier files, registries, and config from an existing `data/F{NNN}/` directory. If no workspace exists, direct the user to run `/exp-report:exp-report` first.

**You MUST execute ALL phases (0→1→2→3→4→5). DO NOT stop after Phase 2.**

## The Insight

Stakeholders often need presentation-ready slide decks alongside document reports. This skill transforms the existing multi-tier report workspace into a self-contained reveal.js HTML slide deck. It reads the same tier files (Tier 0/1/2/3) and figure registry that the report pipeline produces, maps them to evidence-based slide patterns, and applies designer review for presentation-quality output.

The slide deck is a **downstream consumer** of the report workspace — not a pipeline phase. This keeps the report pipeline (10 phases) unchanged and lets users generate slides on-demand.

## Recognition Pattern

- "slides", "슬라이드", "slide deck", "presentation", "발표 자료" → this skill
- `/exp-report:slides` → this skill
- User requests slide generation after report creation

## Workflow (6 Phases)

### Phase 0: Pre-flight

1. **Locate workspace**: Accept path argument or auto-detect:
   ```
   Argument provided:  data/F{NNN}/
   Auto-detect:        Read MANIFEST.yaml → find latest f{nnn} entry → use its path
   ```

2. **Validate workspace files** (all must exist):

   | File | Purpose | Required |
   |------|---------|----------|
   | `questions.json` | Slide structure backbone | YES |
   | `tier0_plain_language.md` | Opening/closing slides | YES |
   | `tier1_decision_brief.md` | Executive summary + decision table | YES |
   | `tier2_evidence_narrative.md` | Per-question evidence slides | YES |
   | `figure_registry.json` | Figure paths for Base64 embedding | YES |
   | `experiments.json` | Experiment metadata | YES |
   | `config.json` | Language settings | NO (default: single-lang English) |
   | `tier3_technical_reference.md` | Appendix slides | NO (skip appendix if absent) |

   If any required file is missing, HALT:
   ```
   >>> Missing workspace files: {list}
   >>> Run /exp-report:exp-report first to generate the report workspace.
   ```

3. **Read config**: Load `config.json` if present. Extract:
   - `dual_lang`: boolean (default `false`)
   - `manifest_language`: "English" | "Korean" (default "English")
   - `report_id`: F{NNN} identifier

4. **Build figure map**: Read `figure_registry.json`, resolve all paths, verify files exist on disk. For each figure, prepare Base64 encoding:
   ```python
   import base64, mimetypes
   def encode_figure(path):
       mime = mimetypes.guess_type(path)[0] or 'image/png'
       with open(path, 'rb') as f:
           b64 = base64.b64encode(f.read()).decode()
       return f"data:{mime};base64,{b64}"
   ```
   For each figure, also generate a **caption string** from existing registry fields:
   `"{exp_id}: {stem} | Source: {exp_description}"` where `exp_description` comes from `experiments.json` (field: try `"description"`, then `"exp_description"`, then `"title"`; use first non-null). Store captions alongside Base64 data for use in `<figcaption>` elements.

5. **Print scope**:
   ```
   >>> Slide generation: {report_id}
   >>> Source: {workspace_path}
   >>> Language: {manifest_language} {"+ " + secondary_lang if dual_lang else ""}
   >>> Figures: {N} registered, {M} found on disk
   >>> Questions: {Q} synthesis questions → ~{Q+6} slides
   ```

### Phase 1: Slide Structure Planning

Map tier content to slide sections using the **IMRaD narrative arc**:

| Section | Slides | Tier Source | Arc Role |
|---------|--------|-------------|----------|
| Title | 1 | experiments.json + config | Setup |
| Motivation | 1-2 | Tier 0 (first paragraph) | Hook (primacy) |
| Methods Overview | 1-2 | Tier 2 (methods sections) | Context |
| Results (×Q) | 1 per synthesis question | Tier 2 (per-question) | Core |
| Decision Summary | 1 | Tier 1 (decision table) | Synthesis |
| Conclusion | 1-2 | Tier 0 (final paragraph) + Tier 1 | Takeaway (recency) |
| Appendix | 1 per experiment | Tier 3 | Reference |

**Slide count guardrail**: If total slides > 25 main deck (excluding appendix), warn:
```
>>> WARNING: {N} main-deck slides exceeds recommended maximum of 25.
>>> Consider consolidating Results slides or reducing synthesis questions.
```

#### Density Rules

**English slides** (default):

| Tier Source | Max Words | Max Bullets | Visual % |
|-------------|-----------|-------------|----------|
| Tier 0 | 20 | 2 | 30% |
| Tier 1 | 35 | 3 | 50% |
| Tier 2 | 45 | 4 | 65% |
| Tier 3 | 80+ | 6-8 | 40% |

**Korean slides** (when `manifest_language == "Korean"`):

| Tier Source | Max 어절 | Max Bullets | ~Max 자 |
|-------------|----------|-------------|---------|
| Tier 0 | 12 | 2 | ~54 |
| Tier 1 | 22 | 3 | ~99 |
| Tier 2 | 28 | 3 | ~126 |
| Tier 3 | 37 | 4 | ~166 |

Korean bullet count drops by 1 at Tier 2/3 due to CJK line-height (1.75 vs 1.5).

#### Density Overflow Rule

When extracted content exceeds the density limit for its tier:
1. **Distill** to fit: compress to the word/bullet limit, keeping only the verdict and top supporting metrics.
2. **Overflow** the full original text into `<aside class="notes">` (speaker notes). Never truncate — move.
3. This applies to both English (word count) and Korean (어절 count) limits identically.

#### Content Extraction Rules

For each slide, extract content from the corresponding tier file:

1. **Title slide**: Report title from `config.json` or first heading of `tier1_decision_brief.md`. Subtitle: experiment count + date range.

2. **Motivation slides**: First 1-2 paragraphs of `tier0_plain_language.md`. Apply Tier 0 density limits.

3. **Methods slides**: Extract methodology description from `tier2_evidence_narrative.md` (text before the first synthesis question heading). If absent, skip.

4. **Results slides** (1 per question):
   - Parse `questions.json` for question list and order
   - For each question, extract the corresponding section from `tier2_evidence_narrative.md`
   - **Down-distillation**: Compress the Tier 2 extract to **Tier 1 register** (≤35 words / ≤3 bullets in English; ≤22 어절 / ≤3 bullets in Korean). Keep only the verdict and top 2-3 supporting metrics. Apply the Density Overflow Rule for any excess.
   - **Conclusion-first title**: Use the verdict/conclusion as the slide title (not the question)
   - **Figure**: If `figure_registry.json` maps a figure to this question's experiments, embed it. If no figure → text-only slide with enlarged verdict text.
   - **Speaker notes**: Full Tier 2 section (unabridged) — this is where the technical detail lives

5. **Decision Summary slide**:
   - Extract decision table from `tier1_decision_brief.md`
   - **Hero number extraction**: Pull 3-4 key metrics from the table for the main slide
   - Full table goes to appendix backup slide
   - Apply Tier 1 density limits

6. **Conclusion slides**: Last paragraph of `tier0_plain_language.md` + key recommendations from `tier1_decision_brief.md`. Repeat the single most important finding (primacy+recency: 2.2× recall advantage).

7. **Appendix slides**: One per experiment from `tier3_technical_reference.md`. Higher density allowed (Tier 3 rules). Full decision table backup here.

#### 6 Slide Design Patterns

Each slide maps to one of these archetypal patterns:

| Pattern | Use For | Density | Visual |
|---------|---------|---------|--------|
| **KeyFinding** | Motivation, Conclusion | Low | Large text, optional icon |
| **Comparison** | A vs B results | Medium | Side-by-side columns |
| **Dashboard** | Decision summary | Medium | 3-4 KPI tiles + mini-table |
| **EvidenceNarrative** | Per-question results | Medium-High | Figure + bullets |
| **Summary** | Conclusion, Takeaway | Low | Numbered list, bold verdict |
| **Appendix** | Per-experiment technical | High | Dense table, small font |
| **SectionTransition** | Section breaks, wrapup | Minimal | Full-bleed accent bg, 1-sentence recap |

#### SectionTransition Auto-Insert Rules

- After every **3 consecutive Results slides**, auto-insert a SectionTransition slide that recaps the key findings so far.
- **Pre-Conclusion wrapup**: Always insert a SectionTransition between Decision Summary and Conclusion (recap the deck's main takeaway before closing).

#### Narrative Thread (Phase 1 prerequisite)

Before mapping content to slides, extract two named strings from `tier0_plain_language.md`:
- **`hook_question`**: The motivating question or problem statement (from the opening paragraph).
- **`takeaway`**: The single most important conclusion (from the closing paragraph).

These MUST appear verbatim: `hook_question` in the Motivation slide, `takeaway` in the Conclusion slide. The SectionTransition before Conclusion should bridge from evidence to `takeaway`.

### Phase 2: HTML Generation

Generate a self-contained reveal.js HTML file. **ALL JavaScript and CSS must be inlined** — no external CDN dependencies in the output file. This ensures offline viewing and ZIP deliverable compatibility.

#### Embedded reveal.js HTML Template

The slide generator MUST produce HTML following this structure. Use the embedded template below as the base, populating `{{SLIDES_CONTENT}}` with generated `<section>` elements.

```python
REVEAL_HTML_TEMPLATE = '''<!DOCTYPE html>
<html lang="{{LANG}}">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{{TITLE}}</title>

<!-- reveal.js core CSS (inlined from CDN at build time) -->
<style id="reveal-core-css">
/* AGENT INSTRUCTION: Fetch https://cdn.jsdelivr.net/npm/reveal.js@5.1.0/dist/reveal.css
   and inline the full contents here. If fetch fails, use the minimal fallback below. */

/* === MINIMAL FALLBACK (use only if CDN fetch fails) === */
.reveal{position:relative;overflow:hidden;touch-action:pinch-zoom}
.reveal .slides{position:absolute;width:100%;height:100%;top:0;right:0;bottom:0;left:0;overflow:visible;z-index:1;text-align:center;-webkit-perspective:600px;perspective:600px;-webkit-perspective-origin:50% 40%;perspective-origin:50% 40%}
.reveal .slides>section{perspective:600px;backface-visibility:hidden;transform-style:preserve-3d;position:absolute;width:100%;padding:20px 0;pointer-events:none;min-height:100%;box-sizing:border-box;display:flex;flex-direction:column;justify-content:center;align-items:center}
.reveal .slides>section.present{pointer-events:auto;z-index:10}
.reveal .slides>section>section{min-height:100%}
.reveal[data-transition-speed="fast"] .slides section{transition-duration:.4s}
.reveal[data-transition-speed="default"] .slides section{transition-duration:.8s}
.reveal[data-transition-speed="slow"] .slides section{transition-duration:1.2s}
</style>

<!-- Theme: white (optimized for data presentations) -->
<style id="reveal-theme-css">
:root {
  --r-background-color: #ffffff;
  --r-main-font: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen, Ubuntu, sans-serif;
  --r-main-font-size: 28px;
  --r-main-color: #2c3e50;
  --r-heading-font: var(--r-main-font);
  --r-heading-color: #1a252f;
  --r-heading-font-weight: 700;
  --r-heading-letter-spacing: -0.02em;
  --r-heading1-size: 2.2em;
  --r-heading2-size: 1.6em;
  --r-heading3-size: 1.2em;
  --r-link-color: #2980b9;
  --r-link-color-hover: #3498db;
  --r-selection-background-color: #3498db;
  --r-selection-color: #fff;
}
.reveal { font-family: var(--r-main-font); font-size: var(--r-main-font-size); color: var(--r-main-color); }
.reveal h1, .reveal h2, .reveal h3 { color: var(--r-heading-color); font-weight: var(--r-heading-font-weight); letter-spacing: var(--r-heading-letter-spacing); line-height: 1.2; margin: 0 0 20px 0; }
.reveal h1 { font-size: var(--r-heading1-size); }
.reveal h2 { font-size: var(--r-heading2-size); }
.reveal h3 { font-size: var(--r-heading3-size); }
.reveal p { line-height: 1.5; margin: 0 0 16px 0; }
.reveal ul, .reveal ol { display: inline-block; text-align: left; margin: 0 0 0 1em; }
.reveal li { margin: 8px 0; line-height: 1.5; }
.reveal a { color: var(--r-link-color); text-decoration: none; }
.reveal a:hover { color: var(--r-link-color-hover); text-decoration: underline; }
.reveal img { max-width: 85%; max-height: 60vh; margin: 15px 0; border: none; box-shadow: 0 2px 8px rgba(0,0,0,0.12); border-radius: 4px; }
.reveal table { margin: auto; border-collapse: collapse; font-size: 0.75em; }
.reveal table th { background: #2c3e50; color: #fff; padding: 8px 16px; font-weight: 600; }
.reveal table td { padding: 8px 16px; border-bottom: 1px solid #ecf0f1; }
.reveal table tr:nth-child(even) td { background: #f8f9fa; }
.reveal blockquote { background: #f0f7ff; border-left: 4px solid #3498db; padding: 12px 20px; margin: 16px auto; width: 80%; text-align: left; font-style: normal; font-size: 0.9em; }
.reveal code { font-family: "SF Mono", "Fira Code", "Cascadia Code", monospace; background: #f4f4f4; padding: 2px 6px; border-radius: 3px; font-size: 0.85em; }
.reveal pre code { display: block; padding: 16px; overflow-x: auto; background: #1e1e1e; color: #d4d4d4; border-radius: 6px; font-size: 0.65em; line-height: 1.4; }
</style>

<!-- Slide-specific styles -->
<style id="slide-patterns-css">
/* === KeyFinding pattern === */
.slide-keyfinding { text-align: center; }
.slide-keyfinding .verdict { font-size: 1.8em; font-weight: 800; color: #1a252f; line-height: 1.3; margin: 0.5em 0; }
.slide-keyfinding .subtext { font-size: 0.85em; color: #7f8c8d; max-width: 70%; margin: 0 auto; }

/* === Comparison pattern === */
.slide-comparison { text-align: left; }
.slide-comparison .compare-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; width: 90%; margin: 0 auto; }
.slide-comparison .compare-col { background: #f8f9fa; border-radius: 8px; padding: 20px; }
.slide-comparison .compare-col h3 { font-size: 1em; margin-bottom: 12px; }
.slide-comparison .compare-col.winner { border: 2px solid #27ae60; background: #f0faf4; }

/* === Dashboard pattern === */
.slide-dashboard .kpi-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 16px; width: 90%; margin: 16px auto; }
.slide-dashboard .kpi-tile { background: #f8f9fa; border-radius: 8px; padding: 16px; text-align: center; }
.slide-dashboard .kpi-tile .kpi-value { font-size: 2em; font-weight: 800; color: #2c3e50; }
.slide-dashboard .kpi-tile .kpi-label { font-size: 0.7em; color: #7f8c8d; text-transform: uppercase; letter-spacing: 0.05em; margin-top: 4px; }

/* === EvidenceNarrative pattern === */
.slide-evidence { text-align: left; }
.slide-evidence .evidence-layout { display: grid; grid-template-columns: 55% 42%; gap: 3%; width: 95%; margin: 0 auto; align-items: start; }
.slide-evidence .evidence-text { font-size: 0.85em; }
.slide-evidence .evidence-text li { margin: 6px 0; }
.slide-evidence .evidence-figure img { max-width: 100%; max-height: 50vh; border-radius: 4px; }
.slide-evidence .evidence-figure figcaption { font-size: 0.6em; color: #95a5a6; text-align: center; margin-top: 4px; }

/* === Summary pattern === */
.slide-summary { text-align: left; }
.slide-summary .summary-list { list-style: none; padding: 0; width: 85%; margin: 0 auto; counter-reset: summary-counter; }
.slide-summary .summary-list li { counter-increment: summary-counter; padding: 10px 0; border-bottom: 1px solid #ecf0f1; font-size: 0.9em; }
.slide-summary .summary-list li::before { content: counter(summary-counter) "."; font-weight: 700; color: #2980b9; margin-right: 8px; }

/* === Appendix pattern === */
.slide-appendix { text-align: left; font-size: 0.75em; }
.slide-appendix table { font-size: 0.85em; width: 95%; }
.slide-appendix h2 { font-size: 1.2em; }

/* === SectionTransition pattern === */
.slide-transition { text-align: center; background: linear-gradient(135deg, #2c3e50 0%, #3498db 100%); color: #fff; }
.slide-transition h2 { color: #fff; font-size: 1.6em; font-weight: 700; }
.slide-transition .recap { font-size: 0.9em; color: #ecf0f1; max-width: 70%; margin: 0.5em auto 0; }

/* === Speaker notes === */
aside.notes { display: none; }

/* === Print/PDF === */
@media print {
  .reveal .slides section { page-break-after: always; min-height: 100vh; }
  .reveal img { box-shadow: none; }
}

/* === Slide number === */
.reveal .slide-number { font-size: 0.5em; color: #95a5a6; }

/* === Progress bar === */
.reveal .progress { height: 3px; color: #3498db; }
</style>

<!-- Korean CJK typography (injected for lang="ko") -->
<style id="korean-cjk-css">
:lang(ko) .reveal,
.reveal[lang="ko"] {
  font-family: "Noto Sans KR", "Pretendard", "Apple SD Gothic Neo",
               "Malgun Gothic", "나눔고딕", sans-serif;
  line-height: 1.75;
  word-break: keep-all;
  overflow-wrap: break-word;
  letter-spacing: -0.02em;
}
:lang(ko) .reveal .slides section,
.reveal[lang="ko"] .slides section {
  text-align: left;
}
:lang(ko) .reveal .slides section p,
:lang(ko) .reveal .slides section li {
  font-size: 1.05em;
  line-height: 1.75;
  letter-spacing: -0.02em;
  word-break: keep-all;
}
:lang(ko) .reveal h1, :lang(ko) .reveal h2, :lang(ko) .reveal h3 {
  font-weight: 700;
  letter-spacing: -0.03em;
  line-height: 1.4;
  word-break: keep-all;
}
:lang(ko) .reveal .slides section li li {
  font-size: 0.875em;
  line-height: 1.7;
}
</style>

{{DESIGNER_CSS}}

</head>
<body>
<div class="reveal">
<div class="slides">

{{SLIDES_CONTENT}}

</div>
</div>

<!-- reveal.js core (inlined) -->
<script id="reveal-core-js">
/* AGENT INSTRUCTION: Fetch https://cdn.jsdelivr.net/npm/reveal.js@5.1.0/dist/reveal.js
   and inline the full contents here. If fetch fails, use the minimal fallback below. */

/* === MINIMAL FALLBACK (use only if CDN fetch fails) === */
!function(){var e="undefined"!=typeof globalThis?globalThis:"undefined"!=typeof self?self:window;
/* Minimal reveal.js initialization shim */
e.Reveal={initialize:function(config){
  var slides=document.querySelector('.reveal .slides');
  var sections=slides.querySelectorAll(':scope > section');
  var current=0;
  function show(n){
    sections.forEach(function(s,i){s.style.display=i===n?'flex':'none';s.classList.toggle('present',i===n)});
    if(config.slideNumber){var sn=document.querySelector('.slide-number');if(sn)sn.textContent=(n+1)+'/'+sections.length}
    if(config.progress){var p=document.querySelector('.progress span');if(p)p.style.width=((n+1)/sections.length*100)+'%'}
  }
  document.addEventListener('keydown',function(e){
    if(e.key==='ArrowRight'||e.key===' '){current=Math.min(current+1,sections.length-1);show(current);e.preventDefault()}
    if(e.key==='ArrowLeft'){current=Math.max(current-1,0);show(current);e.preventDefault()}
    if(e.key==='s'||e.key==='S'){var n=document.querySelector('section.present aside.notes');if(n)alert(n.textContent)}
  });
  var mc=slides;mc.addEventListener('click',function(e){
    var r=mc.getBoundingClientRect();
    if(e.clientX>r.left+r.width/2){current=Math.min(current+1,sections.length-1)}else{current=Math.max(current-1,0)}
    show(current)
  });
  show(0);
  /* slide number element */
  if(config.slideNumber){var sn=document.createElement('div');sn.className='slide-number';sn.style.cssText='position:fixed;bottom:16px;right:16px;font-size:14px;color:#95a5a6;z-index:100';document.body.appendChild(sn)}
  /* progress bar */
  if(config.progress){var pb=document.createElement('div');pb.className='progress';pb.style.cssText='position:fixed;bottom:0;left:0;width:100%;height:3px;z-index:100';pb.innerHTML='<span style=\"display:block;height:100%;background:#3498db;transition:width .3s\"></span>';document.body.appendChild(pb)}
  show(0);
}};
}();
</script>

<script>
Reveal.initialize({
  width: 1280,
  height: 720,
  margin: 0.08,
  hash: true,
  slideNumber: true,
  progress: true,
  transition: 'fade',
  transitionSpeed: 'default',
  pdfMaxPagesPerSlide: 1,
  pdfSeparateFragments: false
});
</script>
</body>
</html>'''
```

**AGENT INSTRUCTION for inlining reveal.js**:

When generating the slide HTML, attempt to fetch the full reveal.js CSS and JS from CDN:
```python
import urllib.request
REVEAL_CSS_URL = "https://cdn.jsdelivr.net/npm/reveal.js@5.1.0/dist/reveal.css"
REVEAL_JS_URL = "https://cdn.jsdelivr.net/npm/reveal.js@5.1.0/dist/reveal.js"

try:
    reveal_css = urllib.request.urlopen(REVEAL_CSS_URL, timeout=10).read().decode()
except:
    reveal_css = None  # Use fallback in template

try:
    reveal_js = urllib.request.urlopen(REVEAL_JS_URL, timeout=10).read().decode()
except:
    reveal_js = None  # Use fallback in template
```

If fetch succeeds, replace the fallback sections in the template. If fetch fails, the minimal fallback provides basic slide navigation (arrow keys, click, slide numbers).

#### Generating `<section>` Elements

Each slide is a `<section>` element. Use `data-auto-animate` for smooth transitions between related slides.

**Title slide**:
```html
<section class="slide-keyfinding">
  <h1>{{REPORT_TITLE}}</h1>
  <p class="subtext">{{EXPERIMENT_COUNT}} experiments · {{DATE_RANGE}}</p>
  <p class="subtext" style="font-size:0.7em; margin-top:2em">Generated by exp-report</p>
</section>
```

**Motivation slide** (KeyFinding pattern):
```html
<section class="slide-keyfinding">
  <h2>Why This Matters</h2>
  <p class="verdict">{{HOOK_QUESTION}}</p>
  <p class="subtext">{{TIER0_CONTEXT_SENTENCE}}</p>
  <aside class="notes">{{TIER0_FIRST_PARAGRAPH_FULL}}</aside>
</section>
```

**Results slide** (EvidenceNarrative pattern, 1 per question):
```html
<section class="slide-evidence">
  <h2>{{CONCLUSION_AS_TITLE}}</h2>
  <div class="evidence-layout">
    <div class="evidence-text">
      <ul>
        <li><strong>{{KEY_METRIC_1}}</strong></li>
        <li>{{SUPPORTING_POINT_2}}</li>
        <li>{{SUPPORTING_POINT_3}}</li>
      </ul>
    </div>
    <div class="evidence-figure">
      <figure>
        <img src="{{BASE64_FIGURE}}" alt="{{FIGURE_ALT}}">
        <figcaption>{{EXP_ID}}: {{FIGURE_STEM}} | Source: {{EXP_DESCRIPTION}}</figcaption>
      </figure>
    </div>
  </div>
  <aside class="notes">{{TIER2_FULL_SECTION}}</aside>
</section>
```

**Decision Summary slide** (Dashboard pattern):
```html
<section class="slide-dashboard">
  <h2>Key Decisions</h2>
  <div class="kpi-grid">
    <div class="kpi-tile">
      <div class="kpi-value">{{METRIC_1_VALUE}}</div>
      <div class="kpi-label">{{METRIC_1_LABEL}}</div>
    </div>
    <!-- repeat for 3-4 KPIs -->
  </div>
  <aside class="notes">{{DECISION_TABLE_FULL}}</aside>
</section>
```

**Conclusion slide** (Summary pattern):
```html
<section class="slide-summary">
  <h2>{{TAKEAWAY}}</h2>
  <ol class="summary-list">
    <li><strong>{{RECOMMENDATION_1}}</strong></li>
    <li>{{RECOMMENDATION_2}}</li>
    <li>{{RECOMMENDATION_3}}</li>
  </ol>
  <aside class="notes">{{TIER0_CLOSING + TIER1_RECOMMENDATIONS}}</aside>
</section>
```

**Comparison slide** (Comparison pattern, for A vs B results):
```html
<section class="slide-comparison">
  <h2>{{COMPARISON_TITLE}}</h2>
  <div class="compare-grid">
    <div class="compare-col {{IF_WINNER_A: 'winner'}}">
      <h3>{{OPTION_A_NAME}}</h3>
      <ul>
        <li><strong>{{METRIC_1}}</strong>: {{VALUE_A1}}</li>
        <li><strong>{{METRIC_2}}</strong>: {{VALUE_A2}}</li>
      </ul>
    </div>
    <div class="compare-col {{IF_WINNER_B: 'winner'}}">
      <h3>{{OPTION_B_NAME}}</h3>
      <ul>
        <li><strong>{{METRIC_1}}</strong>: {{VALUE_B1}}</li>
        <li><strong>{{METRIC_2}}</strong>: {{VALUE_B2}}</li>
      </ul>
    </div>
  </div>
  <aside class="notes">{{TIER2_COMPARISON_SECTION}}</aside>
</section>
```

**Appendix slide** (Appendix pattern, 1 per experiment):
```html
<section class="slide-appendix">
  <h2>{{EXPERIMENT_ID}}: {{EXPERIMENT_TITLE}}</h2>
  <table>
    <tr><th>Parameter</th><th>Value</th></tr>
    <!-- rows from Tier 3 -->
  </table>
  <aside class="notes">{{TIER3_FULL_SECTION}}</aside>
</section>
```

**SectionTransition slide** (inserted per auto-insert rules):
```html
<section class="slide-transition">
  <h2>{{RECAP_TITLE}}</h2>
  <p class="recap">{{ONE_SENTENCE_RECAP_OF_FINDINGS_SO_FAR}}</p>
</section>
```

**Zero-figure Results slide** (KeyFinding pattern fallback):
```html
<section class="slide-keyfinding">
  <h2>{{CONCLUSION_AS_TITLE}}</h2>
  <p class="verdict">{{KEY_FINDING_ENLARGED}}</p>
  <ul style="font-size:0.8em; text-align:left; max-width:70%; margin:0 auto">
    <li>{{SUPPORTING_POINT_1}}</li>
    <li>{{SUPPORTING_POINT_2}}</li>
  </ul>
  <aside class="notes">{{TIER2_FULL_SECTION}}</aside>
</section>
```

#### File Size Guard

After generating the HTML, check file size:
```python
size_mb = len(html_content.encode('utf-8')) / (1024 * 1024)
if size_mb > 15:
    print(f">>> WARNING: Slide HTML is {size_mb:.1f}MB (>{15}MB threshold)")
    print(f">>> Consider reducing figure count or resolution")
```

### Phase 3: Designer Critique Loop (2-iteration)

Apply a **2-iteration fixed designer critique loop** for CSS refinement. This loop is reusable for both report HTML and slide HTML.

#### Iteration 1: Designer generates CSS

Invoke the designer agent with CSS-extraction constraint:

```
Agent(subagent_type="oh-my-claudecode:designer", model="sonnet", prompt="
[SLIDE DESIGNER REVIEW - Iteration 1]

You are reviewing a reveal.js HTML slide deck for presentation quality.
Read the HTML file at: {slide_html_path}

OUTPUT CONSTRAINT: You MUST output ONLY a <style> block. Do NOT rewrite the HTML.
Do NOT output any text other than CSS inside <style>...</style> tags.

Focus areas:
1. TYPOGRAPHY: Font stack, heading hierarchy, line-height for readability at projection distance (>=24pt body)
2. TABLE READABILITY: Alternating rows, sticky headers if scrollable, adequate cell padding
3. RESPONSIVE: @media queries for different screen sizes (1280x720 base)
4. FIGURE PRESENTATION: Captions, centering, max-width constraints, shadow/border
5. SLIDE TRANSITIONS: Smooth data-auto-animate transforms
6. COLOR: WCAG AA contrast (4.5:1 normal, 3.0:1 large text), consistent palette
7. INFORMATION DENSITY: Adequate whitespace between elements, breathing room

{IF manifest_language == 'Korean':}
KOREAN TYPOGRAPHY: Verify word-break:keep-all, line-height:1.75, font-size:1.05em optical correction, Noto Sans KR font stack.
{END IF}

Output a single <style> block with your improvements.
")
```

**Inject CSS**: Insert the designer's `<style>` output before `</head>`, replacing `{{DESIGNER_CSS}}`.

**Corruption guard**:
```python
pre_size = len(html_before)
html_after = html_before.replace('{{DESIGNER_CSS}}', designer_css)
assert len(html_after) >= pre_size, "Designer CSS injection must only add bytes"
```

#### Iteration 2: Critique + Refine

Invoke a critique agent to evaluate the result:

```
Agent(subagent_type="oh-my-claudecode:critic", model="sonnet", prompt="
[SLIDE DESIGN CRITIQUE - Iteration 2]

Review the designer-enhanced slide HTML at: {slide_html_path}

Evaluate these 5 dimensions and rate each HIGH/MEDIUM/LOW severity:

1. WCAG_CONTRAST: All text meets 4.5:1 ratio (normal) or 3.0:1 (large >=18pt/14pt bold)
2. FONT_HIERARCHY: Clear heading > subheading > body > caption size progression
3. INFO_DENSITY: No slide exceeds density limits (Tier0:20w/2b, Tier1:35w/3b, Tier2:45w/4b)
4. RESPONSIVE: Layout works at 1280x720 AND 1024x768 AND mobile widths
5. COLOR_CONSISTENCY: <=5 distinct colors, consistent use across patterns
6. NARRATIVE_ARC: Does Motivation's hook_question echo in Conclusion's takeaway? Are Results ordered by impact?
7. CONTENT_REGISTER: Are main-deck slides at Tier 0/1 register (not Tier 2 technical detail)?
8. CITATION_PRESENCE: Do all figures have <figcaption> with experiment source?

Output format:
CRITIQUE_RESULT:
  WCAG_CONTRAST: [HIGH|MEDIUM|LOW] - [description]
  FONT_HIERARCHY: [HIGH|MEDIUM|LOW] - [description]
  INFO_DENSITY: [HIGH|MEDIUM|LOW] - [description]
  RESPONSIVE: [HIGH|MEDIUM|LOW] - [description]
  COLOR_CONSISTENCY: [HIGH|MEDIUM|LOW] - [description]
  NARRATIVE_ARC: [HIGH|MEDIUM|LOW] - [description]
  CONTENT_REGISTER: [HIGH|MEDIUM|LOW] - [description]
  CITATION_PRESENCE: [HIGH|MEDIUM|LOW] - [description]

SPECIFIC_FIXES:
  - [CSS fix 1]
  - [CSS fix 2]

OUTPUT: A refined <style> block incorporating all fixes.
")
```

**Apply refinement**: Replace the first designer CSS with the critique-refined CSS. Same corruption guard applies.

**Skip condition**: If user invokes with `--fast` or `--no-design`, skip Phase 3 entirely (use base CSS only).

### Phase 4: Dual-Language Generation

If `config.json` has `dual_lang: true`:

1. **Primary language slide** already generated in Phase 2-3
2. **Secondary language slide**: Read the secondary-language tier files (`*_ko.md` or `*_en.md`)
   - If secondary tier files exist in the workspace → use them as source
   - If not → skip secondary language with a note

3. **Korean slide adjustments**:
   - Set `<html lang="ko">`
   - Korean CJK CSS is already embedded in the template (activates via `:lang(ko)`)
   - Apply Korean density rules from Phase 1

4. **Output naming**:
   - Primary: `F{NNN}_slides.html`
   - English: `F{NNN}_slides_en.html` (if dual_lang and manifest_language != English)
   - Korean: `F{NNN}_slides_ko.html` (if dual_lang)

### Phase 5: Output & Deliverable Integration

1. **Write slide files** to workspace:
   ```
   data/F{NNN}/
     F{NNN}_slides.html              # Primary language
     F{NNN}_slides_en.html           # English (if dual_lang)
     F{NNN}_slides_ko.html           # Korean (if dual_lang)
   ```

2. **Deliverable integration** (if `deliverable/` folder exists):
   - Copy slide HTML into `data/F{NNN}/deliverable/`
   - Append slide section to `GUIDE.md`:
     ```markdown
     ## Slide Deck

     The slide deck provides a presentation-ready summary of this report.

     ### Viewing
     - Open `F{NNN}_slides.html` in any modern browser
     - Navigate: Arrow keys or click left/right halves of the screen
     - Speaker notes: Press `S` key
     - Overview: Press `O` key (if full reveal.js loaded)

     ### PDF Export (Playwright headless)
     ```bash
     npx playwright pdf "file://$(pwd)/data/F{NNN}/F{NNN}_slides.html?print-pdf" \
       "data/F{NNN}/F{NNN}_slides.pdf" --wait-for-timeout 3000
     ```
     - The `?print-pdf` query param triggers reveal.js print layout (landscape handled by reveal.js CSS)
     - Playwright's headless Chromium renders slides with full CSS/font fidelity
     - If Playwright unavailable: open HTML in Chrome, add `?print-pdf`, Ctrl+P → Save as PDF
     ```
   - Update ZIP if it exists: add slide HTML to the archive

3. **MANIFEST update**: Add slide outputs to the report entry:
   ```yaml
   f{nnn}:
     outputs:
       - data/F{NNN}/F{NNN}_slides.html
       # existing outputs preserved
   ```

4. **Summary output**:
   ```
   >>> Slide generation complete: {report_id}
   >>> Main deck: {N} slides ({M} with figures)
   >>> Appendix: {A} slides
   >>> Files:
   >>>   {workspace}/F{NNN}_slides.html ({size_kb} KB)
   >>>   {workspace}/F{NNN}_slides_ko.html ({size_kb} KB)  # if dual_lang
   >>> Deliverable: {"updated" if deliverable_exists else "not found (run exp-report first)"}
   ```

## Agent Dispatch Table

| Phase | Agent | Model | Purpose |
|-------|-------|-------|---------|
| 0 | — | — | Pre-flight validation (direct, no agent needed) |
| 1 | `scientist` | haiku | Content extraction from tier files |
| 2 | `executor` | sonnet | HTML generation from template |
| 3.1 | `designer` | sonnet | CSS generation (iteration 1) |
| 3.2 | `critic` | sonnet | CSS critique + refinement (iteration 2) |
| 4 | `executor` | sonnet | Dual-language generation |
| 5 | — | — | File I/O (direct) |

## Verification Checklist

Before declaring completion, verify:

- [ ] `F{NNN}_slides.html` exists and is >10KB (not empty/stub)
- [ ] HTML opens in a browser and slides are navigable (arrow keys work)
- [ ] All figures from `figure_registry.json` are Base64-embedded in the HTML
- [ ] Slide count is within expected range (questions + 6 ± 3)
- [ ] No CDN URLs remain in the HTML (fully self-contained)
- [ ] If `dual_lang`: both `_en.html` and `_ko.html` exist
- [ ] If `dual_lang` + Korean: `<html lang="ko">` is set and CJK CSS is active
- [ ] File size < 15MB (or warning printed)
- [ ] Speaker notes contain full tier content (not empty)
- [ ] Decision table hero numbers appear on Dashboard slide
- [ ] Full decision table appears in Appendix
- [ ] Motivation and Conclusion slides share the key finding (primacy+recency)
- [ ] If `deliverable/` exists: slides copied in and GUIDE.md updated
- [ ] Results slides use Tier 1 register (≤35 words / ≤22 어절, ≤3 bullets per slide)
- [ ] SectionTransition slides present after every 3+ consecutive Results
- [ ] hook_question in Motivation echoes as takeaway in Conclusion
- [ ] All figures have `<figcaption>` with experiment source
- [ ] Density overflow content moved to speaker notes (not truncated)
