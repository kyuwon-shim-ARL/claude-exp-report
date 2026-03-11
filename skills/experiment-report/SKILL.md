---
name: experiment-report
description: Generate multi-tier synthesis report from MANIFEST.yaml experiments. Activate when user mentions "experiment report", "exp-report", "synthesis report", "3-tier report", "통합 보고서", "실험 보고서", "종합 보고서", or requests report generation after experiment finalization.
---

# Experiment Synthesis Report

## CRITICAL: Mandatory Phase Execution

**You MUST execute ALL 10 phases (0→1→2→3→4→5→6→6.5→7→8). DO NOT stop after Phase 4.**

The most common failure is stopping after MD assembly (Phase 4) without running HTML/PDF conversion (Phase 5). The report is NOT complete until HTML and PDF files exist on disk. If you skip Phase 5, the user gets only raw Markdown — this defeats the purpose of the plugin. Phase 4 is complete ONLY when Phase 5 has been executed and `F{NNN}_report.html` exists on disk.

**Checkpoint rule**: After Phase 4, explicitly print `">>> Phase 5: Converting to HTML + PDF..."` before proceeding. If conversion scripts are missing, generate them from the embedded templates below. If Python packages are missing, print clear install instructions and attempt conversion anyway.

## The Insight

Experiment-based analysis projects accumulate results across many experiments (E001, E002, ...) tracked in MANIFEST.yaml. Stakeholders need different levels of detail: executives want decisions (Tier 1), scientists want evidence narratives (Tier 2), and reviewers want technical specs (Tier 3). This skill generates a multi-tier synthesis report (Tier 0: Plain Language, Tier 1: Decision Brief, Tier 2: Evidence Narrative, Tier 3: Technical Reference) from all `status: final` experiments, using parallel agents for each tier, then converts to HTML+PDF.

The multi-tier structure eliminates redundancy (where the same experiment listing appears across multiple parts) by organizing information by *audience need*, not by *experiment sequence*.

## Recognition Pattern

- "통합 보고서 만들어줘" / "종합 보고서" → this skill
- "experiment report" / "exp-report" / "synthesis report" → this skill
- "3-tier report" / "실험 보고서" → this skill
- User requests report generation after experiments are finalized
- Explicit command: `/exp-report:exp-report`

## Prerequisites

1. **MANIFEST.yaml** must exist with experiment entries having `status: final`
2. At least 3 final experiments (otherwise a simple summary suffices)
3. Each experiment should have `description`, `findings`, and `path` fields (alternatives accepted — see Phase 0 normalization)
4. Figure files should exist at `{experiment.path}/figures/*.png`

## Lifecycle Position

This skill fits into experiment-based workflows after experiments are finalized:

```
experiment init (once)
    ↓
experiment start → plan → execute → finalize (per experiment)
    ↓
/exp-report:exp-report  ← THIS SKILL (synthesizes all final experiments)
    ↓
bundle / share deliverables
```

## Workflow (10 Phases)

### Phase 0: Pre-flight

1. **Locate MANIFEST.yaml**: Search in order: `outputs/MANIFEST.yaml` → `MANIFEST.yaml` → prompt user. **Set `today`** = current date in `YYYY-MM-DD` format. **Set `PLUGIN_VERSION`** = `"1.10.0"`.
2. **Filter experiments**: Extract only `status: final` entries. **Exclude synthesis/report entries** (keys matching `/^f\d+/i` such as f001, f002) — these are reports, not source experiments. **Mark superseded experiments**: If an experiment's `notes` or `findings` field indicates it was superseded (e.g., "superseded by E017"), flag it as `superseded: true` — it counts toward exhaustiveness but should be cited as `E010→E017 (superseded)` rather than discussed independently. **If 0 final source experiments remain after filtering, HALT and inform the user — do not proceed to Phase 1.** **If only 1-2 final source experiments remain, HALT — synthesis requires at least 3 experiments for meaningful MECE question clustering. Suggest the user finalize more experiments or use a simple summary instead.**
3. **Normalize MANIFEST fields**: Not all projects use the same field names. Apply this normalization chain before proceeding:
   - `description` ← fall back to `title` if `description` is missing
   - `findings` ← fall back to `result`, then `conclusion`, then `description` if `findings` is missing
   - `path` ← if missing, derive from `outputs[0]` parent directory (e.g., `data/E001/figures/plot.png` → `data/E001/`). If `outputs[0]` is also a bare filename with no directory component, log a warning and skip this experiment from figure registry (path unresolvable).
   - When an output entry is a bare filename (no path separator), resolve as `{path}/{filename}`
   Log which fields were normalized and continue — do not fail on missing canonical field names. **Escalation**: If ALL experiments required `findings` fallback to `description`, warn the user that the Decision Table will contain descriptions rather than empirical findings.
4. **Detect MANIFEST language**: Inspect the `description` fields of the first 3 experiments. If majority are Korean → set `manifest_language = "Korean"`. If majority are English → `"English"`. For mixed cases, default to the language of the first experiment and log a note. **Propagation mechanism**: Include `manifest_language` as an explicit variable in every agent prompt by prepending `Language: {manifest_language}` to each agent's Input section. Do NOT rely on agents to auto-detect language from content — explicit injection ensures consistency across all 4 tiers.
   **Technical term policy**: When `manifest_language = "Korean"`, set `term_policy = "preserve_technical_english"`. This policy instructs agents to write prose in Korean but preserve English for technical terms. The term boundary is **rule-based** (not an exhaustive list):
   - **Always preserve in English**: ALL-CAPS abbreviations (AUC, CI, ROC, GBM, IC50, EC50), capitalized method names (Lasso, Random Forest, XGBoost) and established English-only method terms used as technical vocabulary (logistic regression, support vector machine), SI units and scientific notation (uM, mg/kg, log₁₀), hyphenated compounds where one part is an abbreviation (p-value, dose-response, Hill coefficient), and any term appearing verbatim in English in the MANIFEST `findings` or `notes` fields.
   - **Korean equivalents acceptable**: Common nouns with established Korean scientific equivalents (accuracy/정확도, sensitivity/민감도, specificity/특이도) — agents should prefer the English term when adjacent to a metric value for scannability (e.g., "AUC 0.92, sensitivity 87%"), but Korean equivalents are acceptable in pure prose context.
   - **Never translate**: MANIFEST field values (experiment IDs, parameter names, numeric values), code blocks, file paths.
   - **Tier 0 exception**: Tier 0 uses `gloss_first` policy — English term preserved but must be defined on first use in Korean parentheses (e.g., "AUC(모델 성능 지표, 1.0이 최고) 0.92"). Subsequent uses may use the English term alone.
   When `manifest_language = "English"`, `term_policy` is not set (no action needed — all terms are already in English).
5. **Build figure registry**: Collect figures from **two sources** (MANIFEST `outputs` field is primary, glob is supplementary):
   ```python
   SUPPORTED_EXTS = {'.png', '.jpg', '.jpeg', '.gif', '.svg'}
   figure_registry = []
   seen_paths = set()
   for exp_id, exp in final_experiments.items():
       # Primary: resolved outputs from MANIFEST
       for output_path in exp.get('outputs', []):
           p = Path(output_path)
           if p.suffix.lower() in SUPPORTED_EXTS and str(p) not in seen_paths:
               seen_paths.add(str(p))
               figure_registry.append({
                   "original_path": str(p),
                   "exp_id": exp_id.upper(),
                   "stem": p.stem,
                   "suffix": p.suffix,
                   "deliverable_name": f"{exp_id.upper()}_{p.stem}{p.suffix}",
                   "size_bytes": p.stat().st_size if p.exists() else 0
               })
       # Supplementary: glob for figures not listed in outputs
       for p in sorted(Path(exp['path']).glob('figures/*')):
           if p.suffix.lower() in SUPPORTED_EXTS and str(p) not in seen_paths:
               seen_paths.add(str(p))
               figure_registry.append({
                   "original_path": str(p),
                   "exp_id": exp_id.upper(),
                   "stem": p.stem,
                   "suffix": p.suffix,
                   "deliverable_name": f"{exp_id.upper()}_{p.stem}{p.suffix}",
                   "size_bytes": p.stat().st_size if p.exists() else 0
               })
   ```
   The `deliverable_name` uses `{exp_id}_{stem}{suffix}` to prevent filename collisions. **Deduplicate**: `seen_paths` ensures each file appears only once — first-seen entry wins (log a note on duplicates). **If an experiment has zero figures**, record it as empty and log a note — do not fail. Tier 2 should skip figure embedding for that question if no relevant figure exists. **Size guard**: Compute `total_figure_size_bytes` from the registry. If total exceeds 50 MB, warn the user that the deliverable will be large.
5.5. **Build data registry**: Collect non-figure data files from MANIFEST `outputs` field only (no supplementary glob — unlike figure_registry, data files must be explicitly listed in MANIFEST to avoid capturing intermediate/cache files):
   ```python
   SUPPORTED_DATA_EXTS = {'.csv', '.tsv', '.json', '.xlsx', '.xls',
                          '.yaml', '.yml', '.parquet', '.feather',
                          '.npy', '.npz', '.pkl', '.pickle'}
   data_registry = []
   seen_data_paths = set()
   for exp_id, exp in final_experiments.items():
       for output_path in exp.get('outputs', []):
           # Apply step 3 bare-filename resolution
           if os.sep not in output_path and '/' not in output_path:
               output_path = str(Path(exp['path']) / output_path)
           p = Path(output_path)
           if p.suffix.lower() in SUPPORTED_DATA_EXTS and not p.is_symlink() and str(p) not in seen_data_paths:
               seen_data_paths.add(str(p))
               data_registry.append({
                   "original_path": str(p),
                   "exp_id": exp_id.upper(),
                   "stem": p.stem,
                   "suffix": p.suffix,
                   "deliverable_name": f"{exp_id.upper()}_{p.stem}{p.suffix}",
                   "size_bytes": p.stat().st_size if p.exists() else 0
               })
   ```
   **Symlink guard**: Symlinks are excluded to avoid aliased duplicates. Note: figure_registry does not yet apply this guard — tracked for future update. **Size warning**: Compute `total_data_size_bytes` from the registry. If total exceeds 200 MB, warn the user that the deliverable will be large (but do not block — user explicitly allows large data files). **Zero-data edge case**: If no data files match the allowlist, `data_registry` is empty — this is valid. Phase 6.5 will skip creating the `data/` directory and GUIDE.md will omit the data section.
6. **Auto-detect F-number**: Scan MANIFEST for existing `f###` entries (case-insensitive). Next number = max + 1. Format as `F{NNN}` (e.g., F004).
7. **Check conversion scripts**: Look for `scripts/md_to_html.py` and `scripts/md_to_pdf.py`. If missing, generate them from embedded templates in Phase 5.
8. **Create output directory**: `data/F{NNN}/`
9. **Serialize computed data**: Save figure registry to `data/F{NNN}/figure_registry.json` (agents need this file as input). Save data registry to `data/F{NNN}/data_registry.json` (Phase 6.5 needs this for deliverable packaging). Save experiment summary to `data/F{NNN}/experiments.json` (experiment IDs, descriptions, findings, paths, superseded flags).

### Phase 1: Question Discovery (hybrid)

**Auto-derive candidate questions from MANIFEST metadata:**

Cluster the `description` and `findings` fields of all final experiments into 4-8 candidate questions using these heuristic categories:

| Arc Position | Category | Heuristic | Example Question |
|-------------|----------|-----------|------------------|
| **Claim** | Pipeline validation | Experiments with accuracy/performance metrics | "Does the pipeline work?" |
| **Claim** | Reliability & rigor | Experiments with CI, bootstrap, power analysis, sample size | "How reliable are the headline results?" |
| **Mechanism** | Mechanism / causality | Experiments with correlation/mediation analysis | "Why does it work?" |
| **Mechanism** | Domain decomposition | Experiments with sub-class/sub-target analysis | "Are classes internally coherent?" |
| **Boundary** | Negative results / failures | Experiments with poor metrics, failed approaches, or deprecated status | "What approaches failed and why?" |
| **Boundary** | Scalability / generalization | Experiments with cross-condition validation | "Does it generalize?" |
| **Practical** | Methodology comparison | Experiments comparing approaches (vs, comparison) | "Which method is best?" |
| **Practical** | Quantitative relationship | Experiments with continuous variable analysis (concentration, dose, time) | "Does the quantitative factor matter?" |

Categories in the same **Arc Position** should be merged into a single question when possible. The Arc column maps to the hierarchical ordering in Check 2 below.

**Validate question structure before presenting to user:**

After deriving candidate questions, run these 4 checks. Fix violations before presenting.

**Check 1 — MECE (Mutually Exclusive, Collectively Exhaustive)**
- **Mutual exclusivity**: If two questions share >50% of their assigned experiments *in either question* AND address the same aspect (e.g., both about "reliability"), merge them. An experiment may appear in multiple questions only if each question examines a **different aspect** of that experiment (e.g., Q1 uses E014's accuracy, Q4 uses E014's weighting effect). When this happens, annotate the aspect: `E014 (accuracy)` vs `E014 (weighting)`.
- **Exhaustiveness**: Every `status: final` experiment must be assigned to at least one question. If any experiment is unassigned, either create a new question or expand an existing one to include it. List unassigned experiments explicitly and resolve before proceeding.

**Check 2 — Hierarchical Ordering (Narrative Arc)**
Questions must follow a logical progression, not a flat list. Use this universal ordering framework:

```
1. Claim establishment  → "Does X work?" (headline result)
2. Mechanism            → "Why/how does it work?" (explanatory evidence)
3. Boundaries & failures → "Where does it fail?" (negative results, limitations)
4. Practical factors    → "What conditions matter for real-world use?" (scaling, cost, requirements)
```

Statistical rigor (CI, power analysis, sample size) should be **integrated into the relevant question** — typically Q1 (as a caveat on the claim) or Q4 (as a practical requirement) — rather than isolated as a standalone question. A standalone "Is the study adequately powered?" question risks retroactively undermining all prior questions.

Superseded experiments (e.g., E010 replaced by E017) should be noted as `E010→E017 (superseded)` in the relevant question, not listed separately.

**Check 3 — Conclusion Reachability**
Verify that answering ALL questions produces a coherent final conclusion. Draft a chain test table:

```
| Q# | One-sentence answer (hypothetical) | Connects to next? |
|----|------------------------------------|--------------------|
| Q1 | "[headline claim with key metric]" | → Q2 because...    |
| Q2 | "[mechanism explanation]"          | → Q3 because...    |
| ...| ...                                | ...                |
| QN | "[practical implication]"          | → Conclusion       |
| -- | "Therefore, [overall conclusion]"  | --                 |
```

- If any "Connects to next?" cell is blank or forced, reorder or reframe the questions.
- If the final conclusion cannot address "What should be done next?", add practical implications to the last question.
- This table is for internal validation only — do not include it in the output to the user.

**Check 4 — Question Count**
- Target: 4-6 questions. Fewer than 4 suggests over-merging. More than 6 suggests insufficient abstraction.
- If >6 after heuristic derivation, look for questions that are sub-aspects of a broader question and merge them.
- **Priority rule**: If an outlier experiment fits no existing question, exhaustiveness (Check 1) takes priority over count target. Create the extra question rather than leaving experiments unassigned. Annotate it as "Auxiliary" in the arc.

**Present validated questions to the user:**
- Print the questions as a numbered list **with the narrative arc label** (Claim / Mechanism / Boundary / Practical)
- Show the experiment assignment for each question (with aspect annotations if an experiment appears in multiple questions)
- Ask the user to confirm, edit, reorder, or add custom questions
- Wait for user response before proceeding
- Each question should be a single sentence ending with "?"

**Post-edit re-validation**: After the user confirms, edits, or reorders questions, re-run Checks 1 and 4:
- **Re-check exhaustiveness** (Check 1): If the user deleted a question, verify all its experiments are reassigned to remaining questions. List any orphaned experiments and ask the user how to handle them before proceeding.
- **Re-check count** (Check 4): If the user reduced below 4 or expanded above 6, accept their decision but log a note.
- If the user rejects all questions and requests a restart, return to the auto-derivation step with the user's feedback incorporated.

**Serialize**: Save confirmed questions to `data/F{NNN}/questions.json` (array of `{id, text, arc_label, experiment_ids}`). Agents 2 and 4 read this file.

### Phase 2: Decision Table Draft (lead agent, sequential)

For each confirmed question, extract an actionable decision:

```markdown
| Decision | Judgment | Evidence | Key Numbers |
|----------|----------|----------|-------------|
| D1: [topic] | [one-sentence verdict] | [which experiments support this] | [2-4 key metrics] |
```

**Rules:**
- Every number must trace back to a MANIFEST `findings` field
- Judgments use definitive language: "confirmed", "not supported", "conditional on..."
- Evidence cites experiment IDs (E001, E012, etc.)

**Serialize**: Save Decision Table to `data/F{NNN}/decision_table.md`. Agent 1 reads this file.

### Phase 3: Tier Generation (3+1 agents)

Spawn agents in two waves:
- **Wave 1** (parallel): Agents 1 (Tier 1), 2 (Tier 2), 3 (Tier 3) — fire simultaneously
- **Wave 2** (sequential): Agent 4 (Tier 0) — runs after Agent 1 completes and `tier1_decision_brief.md` exists on disk

Use `subagent_type="general-purpose"` for all agents:

#### Agent 1: Tier 1 — Decision Brief (sonnet, ~20-30 lines)

**Prompt template:**
```
You are writing Tier 1 (Decision Brief) of a multi-tier experiment synthesis report.

Input: MANIFEST.yaml final experiments, Decision Table from Phase 2 (`data/F{NNN}/decision_table.md`), Date: {today}, Experiment count: {EXPERIMENT_COUNT}, Language: {manifest_language}.

Output file: data/F{NNN}/tier1_decision_brief.md

Structure:
1. Title line with version (F{NNN} v1.0), date ({today}), experiment count ({EXPERIMENT_COUNT})
2. Executive Summary (1 paragraph, ~5 sentences):
   - What was done (system, method)
   - Headline result (best metric)
   - Key caveat (biggest limitation)
   - Actionable next step
3. Decision Table (from Phase 2, formatted as markdown table)

Constraints:
- NO experiment-by-experiment listing
- NO figures (Tier 1 is text-only for quick scanning)
- Every number must trace to MANIFEST `findings` or `notes` fields
- Maximum 30 lines total
- Write in {manifest_language}
- Term policy (Korean): Preserve English for technical terms — ALL-CAPS abbreviations (AUC, CI, ROC, GBM), method names (Lasso, Random Forest, XGBoost), units (uM, mg/kg), and terms appearing in English in MANIFEST `findings`/`notes`. Write surrounding prose in Korean. Never translate MANIFEST field values, experiment IDs, or code blocks.
- Use project-relative paths for any file references (e.g., `data/E001/figures/plot.png`)
```

#### Agent 2: Tier 2 — Evidence Narrative (opus, ~100-150 lines)

**Prompt template:**
```
You are writing Tier 2 (Evidence Narrative) of a multi-tier experiment synthesis report.

Input: MANIFEST.yaml final experiments (including `notes` fields — these contain critical context such as retroactive pipeline definitions, methodological caveats, and exclusion rationale), confirmed questions from Phase 1 (`data/F{NNN}/questions.json` — with narrative arc labels: Claim/Mechanism/Boundary/Practical and experiment assignments), figure registry (`data/F{NNN}/figure_registry.json`).

Output file: data/F{NNN}/tier2_evidence_narrative.md

Structure — for each question Q1..QN:
### Q{i}: [Question text]
**Verdict: [Bold one-sentence conclusion with key numbers in the first sentence.]**

[2-3 paragraphs of evidence narrative. Conclusion-first style — start each paragraph
with the finding, then explain methodology and context. Cite experiment IDs (E001, E012).]

![Figure caption](path/to/figure.png)
*Figure {i}. Description (experiment ID)*

**Implication**: [One sentence on what this means for the project]

---

Constraints:
- 1 figure per question where available (select most informative from registry); if no relevant figure exists for a question, skip the figure block and note "No figure available" in the Implication line
- Bold verdict opener for every question — first sentence states the conclusion
- Every number must trace to MANIFEST `findings` or `notes` fields (Phase 4 verification will reconcile cross-tier numeric consistency — do not attempt to match Tier 1 numbers at generation time since Tier 1 runs in parallel)
- Paragraphs are dense (no bullet lists in the narrative)
- Use experiment IDs consistently (E001, not "the first experiment")
- Follow the question order as a narrative arc: each question's section should
  connect to the next with a logical transition (e.g., "Having established that
  the pipeline works, the next question is why...")
- If an experiment appears in multiple questions, discuss only the aspect
  relevant to the current question (do not repeat the same analysis)
- Write in {manifest_language}
- Term policy (Korean): Preserve English for technical terms — ALL-CAPS abbreviations (AUC, CI, ROC, GBM), method names (Lasso, Random Forest, XGBoost), units (uM, mg/kg), and terms appearing in English in MANIFEST `findings`/`notes`. Write surrounding prose in Korean. Never translate MANIFEST field values, experiment IDs, or code blocks.
- Use project-relative paths for all figure references (e.g., `data/E001/figures/plot.png` — not absolute paths, not report-relative)
- If multiple formal hypothesis tests are cited across questions, note the total count and whether family-wise error rate correction (e.g., Holm, Bonferroni) was applied
- Scan for statistical language consistency: do not use "significantly" unless the cited p-value is < 0.05; use "practical improvement" or "non-significant trend" for p ≥ 0.05
```

#### Agent 3: Tier 3 — Technical Reference (haiku, ~60-100 lines)

**Prompt template:**
```
You are writing Tier 3 (Technical Reference) of a multi-tier experiment synthesis report.

Input: MANIFEST.yaml final experiments, figure registry (`data/F{NNN}/figure_registry.json`).

Output file: data/F{NNN}/tier3_technical_reference.md

Structure (all inside a single <details> block):

## Technical Reference
<details>
<summary>Expand: Pipeline spec, classification tables, cross-reference, figure gallery</summary>

## A. Pipeline Specification
[Exact step-by-step spec from canonical pipeline experiment, code-block format]

## B. Classification Table
[Table of classes/categories with counts and member lists]

## C. Experiment Cross-Reference Matrix
| Experiment | Status | Key Finding | Figures | Source | Related Experiments |
for each final experiment, add:
  Figures: link to figure files from registry (e.g., `[roc.png](data/E001/figures/roc.png)`)
  Source: link to experiment directory (e.g., `[E001](data/E001/)`)...

## D. Figure Gallery
[All figures from all experiments, organized by topic, with captions and experiment IDs]

</details>

Constraints:
- Everything inside <details> tags for collapsibility
- Pipeline spec in code block (not prose). If no canonical pipeline experiment exists, label Section A as "N/A — no canonical pipeline defined" rather than guessing
- Cross-reference matrix must list ALL final experiments (including superseded ones, marked as such)
- Figure paths must be valid (from registry)
- No narrative — this is reference material only
- Write in {manifest_language}
- Term policy (Korean): Preserve English for technical terms — ALL-CAPS abbreviations (AUC, CI, ROC, GBM), method names (Lasso, Random Forest, XGBoost), units (uM, mg/kg), and terms appearing in English in MANIFEST `findings`/`notes`. Write surrounding prose in Korean. Never translate MANIFEST field values, experiment IDs, or code blocks.
- Use project-relative paths for all figure and source references
```

#### Agent 4: Tier 0 — Plain Language Summary (sonnet, ~20-30 lines)

**Prompt template:**
```
You are writing Tier 0 (Plain Language Summary) of an experiment synthesis report.
Your job is to rewrite the Decision Brief (Tier 1) so that a non-specialist can
understand it — no jargon, no acronyms, no technical terms.

Input: Tier 1 Decision Brief (tier1_decision_brief.md), MANIFEST experiment descriptions.

Output file: data/F{NNN}/tier0_plain_language.md

Structure:
1. Title: "What We Found (Plain Language)"
2. One-paragraph summary using everyday language:
   - What the project tried to do (use analogies if helpful)
   - The main result in plain terms (keep the same numbers)
   - The biggest caveat in plain terms
   - What happens next
3. Decision Table rewritten with:
   - Column headers simplified (e.g., "Judgment" → "What we concluded")
   - Technical terms replaced with plain explanations
   - Parenthetical clarifications for any unavoidable technical terms

Constraints:
- Every number from Tier 1 must appear identically in Tier 0
- NO acronyms unless immediately defined on first use — for Korean output, apply `gloss_first`: preserve the English acronym as the primary form with a Korean parenthetical (e.g., "AUC(모델 성능 지표, 1.0이 최고) 0.92"); for English output, spell out on first use with acronym in parentheses
- NO assuming domain knowledge — explain as if to a smart generalist
- Keep the same structure and conclusions as Tier 1, only change the language
- Maximum 30 lines total
- Match the language of the MANIFEST content (Korean → Korean, English → English)
- Term policy (Korean): Use `gloss_first` — preserve English technical terms verbatim but define each on first use with a Korean parenthetical (e.g., "AUC(모델 성능 지표, 1.0이 최고) 0.92", "EC50(반수 효과 농도)"). After first definition, use the English term alone. Applies to ALL-CAPS abbreviations (AUC, CI, ROC), method names (Lasso, Random Forest), units (uM, mg/kg), and any term appearing in English in MANIFEST fields. Do NOT translate these terms to Korean — only add a parenthetical gloss on first use.
```

**Phase 3 Exit Criterion**: Before proceeding to Phase 4, verify ALL 4 tier files exist and are non-empty:
```python
# Pseudo-code — substitute actual F-number for {NNN}
tier_files = ['tier0_plain_language.md', 'tier1_decision_brief.md',
              'tier2_evidence_narrative.md', 'tier3_technical_reference.md']
for f in tier_files:
    p = Path(f'data/F{NNN}/{f}')
    if not p.exists() or p.stat().st_size == 0:
        print(f"MISSING or EMPTY: {f}")
    # Minimum quality checks:
    #   tier1 must contain a pipe table (|...|)
    #   tier2 must contain question headers (### Q)
    #   tier3 must contain <details>
    #   tier0 must contain its expected title
```
If any file is missing, re-spawn the failed agent (max 2 retries per agent) before proceeding. If Agent 1 (Tier 1) failed, Agent 4 (Tier 0) cannot run — fix Agent 1 first.

### Phase 4: Assembly + Verification

1. **Concatenate**: Read tier0, tier1, tier2, tier3 files → assemble into `F{NNN}_report.md`

   ```markdown
   # [Project Title]: [Report Subtitle]
   **Version**: F{NNN} v1.0 | **Date**: {today} | **Data**: {EXPERIMENT_COUNT} experiments ({range})

   ---

   {tier0 content — Plain Language Summary}

   ---

   {tier1 content — Executive Summary + Decision Table}

   ---

   {tier2 content — Evidence by Question}

   ---

   {tier3 content — Technical Reference in <details>}
   ```

2. **Cross-tier numeric verification** (detailed protocol):

   **Step 2a — Extract Tier 1 numbers**: Parse Decision Table cells using this regex:
   ```
   (?<!\d)[−\-+]?\d[\d,]*\.?\d*(?:[eE][−\-+]?\d+)?%?
   ```
   This handles: `90.5`, `86%`, `0.45`, `−0.220` (Unicode minus U+2212), `1e-30`, `1,234`. **Pre-process CI ranges** before extraction: replace `(\d+\.?\d*)\s*[-–−]\s*(\d+\.?\d*)` with two separate numbers to avoid misinterpreting range hyphens as negative signs (e.g., `0.81-0.89` → `0.81` and `0.89`, NOT `-0.89`). **Normalize commas**: strip `,` from comma-formatted numbers before comparison (`1,234` → `1234`).

   **Exclude** from matching using these filters:
   - Dates: skip numbers preceded by year context (e.g., `2024-01-15`)
   - Section/figure numbers: skip `Figure N`, `Q1`, `D1`, `E001`
   - Compound identifiers: skip numbers inside `[A-Z]{2,}\d+` patterns (EC50, IC50, LD50)
   - Contextual counts: skip "N-fold", "N features", "N variables", "N minutes", "N iterations", "N resamples"
   - Bootstrap/iteration counts: skip numbers in "N iterations/resamples" context
   - Korean numeral multipliers: skip numbers followed by Korean multiplier suffixes (만, 억, 조) — these are informal magnitude indicators, not statistical values

   **Step 2b — Search Tier 2**: For each extracted Tier 1 number, verify it appears **traceably** in Tier 2 text. Traceably means: exact match OR within rounding tolerance (e.g., `90.5%` matches `90.48%`). Handle percentage/decimal equivalence: `0.905` matches `90.5%`. Flag rounding differences > 5% relative for manual review.

   **Step 2c — Search Tier 0**: For each extracted Tier 1 number, verify it appears traceably in Tier 0 text (with plain-language context around technical terms — e.g., "AUC 0.92" → "scored 0.92 on AUC, a measure where 1.0 is perfect")

   **Step 2d — Cross-check MANIFEST**: For each number in Tier 1, verify it traces to a MANIFEST `findings` or `notes` field (or normalized equivalent: `result`, `conclusion`)

   **Step 2e — Report**: List any orphaned numbers (in Tier 1 but not in Tier 2, Tier 0, or MANIFEST)

   If verification fails, patch Tier 2 to include missing numbers before proceeding to Phase 5.

3. **Figure path verification**:
   - Extract all `![...](path)` references from the assembled report
   - Check each path exists on disk
   - **If a path is broken**: remove the figure reference from the report and log a warning. Do not block Phase 5 for broken figures, but report them in Phase 8 summary.

4. **Save all files** to `data/F{NNN}/`:
   - `F{NNN}_report.md` (assembled)
   - `tier0_plain_language.md`
   - `tier1_decision_brief.md`
   - `tier2_evidence_narrative.md`
   - `tier3_technical_reference.md`

### Phase 5: Conversion (HTML + PDF) — MANDATORY

> **⚠ DO NOT SKIP THIS PHASE.** The report is incomplete without HTML and PDF outputs.
> Print `">>> Phase 5: Converting to HTML + PDF..."` before starting.

**Step 5a: Ensure conversion scripts exist**
- If `scripts/md_to_html.py` exists → use it
- If `scripts/md_to_pdf.py` exists → use it
- If either is missing → **generate from embedded templates below** (create `scripts/` directory if needed)

**Step 5b: Check Python dependencies**
```bash
python -c "import markdown" 2>/dev/null || echo "WARNING: 'markdown' package not installed. Install with: pip install markdown"
python -c "import fitz" 2>/dev/null || echo "WARNING: 'PyMuPDF' package not installed. Install with: pip install PyMuPDF"
```
- If `markdown` is missing: proceed with HTML conversion anyway (script has regex fallback)
- If `fitz` is missing: the PDF script raises `ImportError` at runtime — no pre-intervention needed. Always run both scripts as shown in Step 5c. Continue to Phase 6

**Step 5c: Run conversions**
```bash
python scripts/md_to_html.py data/F{NNN}/F{NNN}_report.md &
python scripts/md_to_pdf.py data/F{NNN}/F{NNN}_report.md &
wait
```

**Step 5d: Verify outputs (MUST check before proceeding)**
```bash
ls -la data/F{NNN}/F{NNN}_report.html data/F{NNN}/F{NNN}_report.pdf 2>&1
```
- `F{NNN}_report.html` size > 0 → proceed
- `F{NNN}_report.pdf` size > 0 → proceed
- If HTML is missing or empty → **STOP and debug** (check script output for errors)
- **Content-quality spot-check**: If the source MD contains pipe-table syntax (`|...|`), verify `<table>` appears in the HTML output. If not, the fallback failed to convert tables — install `markdown` package and retry.
- If PDF is missing → log warning but continue (PDF requires PyMuPDF)

#### Embedded Conversion Script Templates

If `scripts/md_to_html.py` is missing, generate this generic version:

```python
#!/usr/bin/env python3
"""Convert Markdown reports to self-contained interactive HTML.

Usage:
    python scripts/md_to_html.py report.md
    python scripts/md_to_html.py report.md --output output.html
"""
import argparse, base64, re, sys
from pathlib import Path

PROJECT_ROOT = Path(__file__).parent.parent.resolve()

try:
    import markdown
    HAS_MARKDOWN = True
except ImportError:
    HAS_MARKDOWN = False

CSS = """<style>
* { box-sizing: border-box; }
body { max-width: 900px; margin: 0 auto; padding: 2rem; font-family: system-ui, sans-serif; line-height: 1.6; color: #333; }
h1 { border-bottom: 3px solid #2c3e50; padding-bottom: 0.5rem; color: #2c3e50; }
h2 { color: #2c3e50; border-bottom: 1px solid #eee; margin-top: 2rem; }
h3 { color: #34495e; margin-top: 1.5rem; }
table { border-collapse: collapse; width: 100%; margin: 1rem 0; font-size: 0.95rem; }
th { background: #2c3e50; color: white; padding: 0.5rem; text-align: left; }
td { border: 1px solid #ddd; padding: 0.5rem; }
tr:nth-child(even) { background: #f9f9f9; }
img { max-width: 100%; display: block; margin: 1rem auto; border: 1px solid #eee; border-radius: 4px; }
details { background: #f8f9fa; border: 1px solid #dee2e6; border-radius: 4px; padding: 1rem; margin: 1rem 0; }
summary { cursor: pointer; font-weight: bold; color: #2c3e50; }
code { background: #f5f5f5; padding: 0.2rem 0.4rem; border-radius: 3px; font-family: monospace; font-size: 0.9em; }
pre { background: #f5f5f5; padding: 1rem; border-radius: 4px; overflow-x: auto; }
pre code { background: transparent; padding: 0; }
blockquote { border-left: 4px solid #3498db; padding: 0.5rem 1rem; margin-left: 0; color: #555; background: #f8f9fa; }
.toc { position: sticky; top: 0; background: white; border-bottom: 2px solid #eee; padding: 1rem; z-index: 100; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
.toc ul { list-style: none; padding: 0; display: flex; flex-wrap: wrap; gap: 1rem; }
.toc a { color: #2c3e50; text-decoration: none; padding: 0.3rem 0.6rem; border-radius: 3px; }
.toc a:hover { background: #ecf0f1; }
.toc-h3 { margin-left: 1.5rem; font-size: 0.9em; }
</style>"""

def img_to_b64(p):
    try:
        mime = {'.png':'image/png','.jpg':'image/jpeg','.jpeg':'image/jpeg','.gif':'image/gif','.svg':'image/svg+xml'}.get(p.suffix.lower(),'image/png')
        return f"data:{mime};base64,{base64.b64encode(p.read_bytes()).decode()}"
    except Exception as e:
        print(f"Warning: {p}: {e}", file=sys.stderr); return str(p)

def process_images(content, md_path):
    def repl(m):
        img = PROJECT_ROOT / m.group(2)
        return f'![{m.group(1)}]({img_to_b64(img)})' if img.exists() else m.group(0)
    return re.sub(r'!\[([^\]]*)\]\(([^)\s]+)(?:\s+"[^"]*")?\)', repl, content)

def gen_toc(content):
    items = []
    def add_h2_id(m):
        t = m.group(1); sid = re.sub(r'[\s_]+', '-', re.sub(r'[^\w\s-]', '', t.lower()))
        items.append((2, sid, t)); return f'## <span id="{sid}">{t}</span>'
    def add_h3_id(m):
        t = m.group(1); sid = re.sub(r'[\s_]+', '-', re.sub(r'[^\w\s-]', '', t.lower()))
        items.append((3, sid, t)); return f'### <span id="{sid}">{t}</span>'
    content = re.sub(r'^## (.+)$', add_h2_id, content, flags=re.MULTILINE)
    content = re.sub(r'^### (.+)$', add_h3_id, content, flags=re.MULTILINE)
    if items:
        li = ''.join(f'<li class="toc-h{lvl}"><a href="#{s}">{t}</a></li>' for lvl,s,t in items)
        toc = f'<div class="toc"><h2>Contents</h2><ul>{li}</ul></div>'
    else:
        toc = ''
    return content, toc

def convert(md_path, out=None):
    content = md_path.read_text(encoding='utf-8')
    content = process_images(content, md_path)
    content, toc = gen_toc(content)
    if HAS_MARKDOWN:
        body = markdown.Markdown(extensions=['tables','fenced_code']).convert(content)
        body = re.sub(r'<img alt="([^"]*)" src="([^"]+)" ?/?>', r'<figure><img src="\2" alt="\1"><figcaption>\1</figcaption></figure>', body)
    else:
        # Regex-based markdown-to-HTML fallback (handles tables + code blocks)
        body = content
        # Fenced code blocks (``` ... ```)
        body = re.sub(r'```(\w*)\n(.*?)```', lambda m: f'<pre><code class="language-{m.group(1)}">{m.group(2).replace("<","&lt;").replace(">","&gt;")}</code></pre>', body, flags=re.DOTALL)
        # Pipe tables
        def table_repl(m):
            rows = [r.strip() for r in m.group(0).strip().split('\n') if r.strip()]
            if len(rows) < 2: return m.group(0)
            hdr = [c.strip() for c in rows[0].strip('|').split('|')]
            html = '<table><thead><tr>' + ''.join(f'<th>{c}</th>' for c in hdr) + '</tr></thead><tbody>'
            for row in rows[2:]:  # skip separator row
                cells = [c.strip() for c in row.strip('|').split('|')]
                html += '<tr>' + ''.join(f'<td>{c}</td>' for c in cells) + '</tr>'
            return html + '</tbody></table>'
        body = re.sub(r'(?:^\|.+\|$\n?){2,}', table_repl, body, flags=re.MULTILINE)
        body = re.sub(r'^### (.+)$', r'<h3>\1</h3>', body, flags=re.MULTILINE)
        body = re.sub(r'^## (.+)$', r'<h2>\1</h2>', body, flags=re.MULTILINE)
        body = re.sub(r'^# (.+)$', r'<h1>\1</h1>', body, flags=re.MULTILINE)
        body = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', body)
        body = re.sub(r'\*(.+?)\*', r'<em>\1</em>', body)
        body = re.sub(r'`([^`]+)`', r'<code>\1</code>', body)
        body = re.sub(r'^\d+\. (.+)$', r'<oli>\1</oli>', body, flags=re.MULTILINE)
        body = re.sub(r'^\- (.+)$', r'<uli>\1</uli>', body, flags=re.MULTILINE)
        body = re.sub(r'((?:<oli>.*</oli>\n?)+)', lambda m: '<ol>' + m.group(0).replace('<oli>','<li>').replace('</oli>','</li>') + '</ol>', body)
        body = re.sub(r'((?:<uli>.*</uli>\n?)+)', lambda m: '<ul>' + m.group(0).replace('<uli>','<li>').replace('</uli>','</li>') + '</ul>', body)
        # Paragraph wrapping: skip segments containing block-level elements
        BLOCK_TAGS = re.compile(r'<(?:h[1-6]|table|thead|tbody|tr|pre|ul|ol|details|figure|div|blockquote)')
        segments = re.split(r'\n{2,}', body)
        wrapped = []
        for seg in segments:
            seg = seg.strip()
            if not seg: continue
            wrapped.append(seg if BLOCK_TAGS.search(seg) else f'<p>{seg}</p>')
        body = '\n'.join(wrapped)
        print("Warning: 'markdown' package not installed. Using regex fallback.", file=sys.stderr)
        print("Install for better output: pip install markdown", file=sys.stderr)
    out = out or md_path.with_suffix('.html')
    out.write_text(f'<!DOCTYPE html><html><head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"><title>{md_path.stem}</title>{CSS}</head><body>{toc}{body}</body></html>', encoding='utf-8')
    return out

if __name__ == '__main__':
    ap = argparse.ArgumentParser(); ap.add_argument('input', type=Path); ap.add_argument('-o','--output', type=Path)
    a = ap.parse_args(); o = convert(a.input, a.output); print(f"OK: {o} ({o.stat().st_size/1024:.1f} KB)")
```

If `scripts/md_to_pdf.py` is missing, generate this generic version:

```python
#!/usr/bin/env python3
"""Convert Markdown reports to PDF using PyMuPDF Story API.

Usage:
    python scripts/md_to_pdf.py report.md
    python scripts/md_to_pdf.py report.md --output output.pdf
"""
import argparse, re, sys
from pathlib import Path

PROJECT_ROOT = Path(__file__).parent.parent.resolve()

PDF_CSS = """
body { font-family: sans-serif; font-size: 11pt; line-height: 1.5; color: #333; }
h1 { border-bottom: 2px solid #2c3e50; padding-bottom: 8pt; color: #2c3e50; font-size: 18pt; }
h2 { color: #2c3e50; font-size: 14pt; margin-top: 16pt; }
h3 { color: #34495e; font-size: 12pt; }
table { border-collapse: collapse; width: 100%; font-size: 9pt; margin: 12pt 0; }
th { background: #2c3e50; color: white; padding: 6pt 8pt; text-align: left; }
td { border: 1px solid #ddd; padding: 4pt 8pt; }
img { max-width: 80%; display: block; margin: 12pt auto; }
pre { background: #f5f5f5; border: 1px solid #ddd; padding: 8pt; font-family: monospace; font-size: 8pt; }
code { background: #f5f5f5; padding: 2pt 4pt; font-family: monospace; font-size: 9pt; }
pre code { background: none; padding: 0; }
blockquote { border-left: 3pt solid #2c3e50; padding-left: 12pt; color: #555; font-style: italic; }
ul, ol { margin: 8pt 0; padding-left: 20pt; }
"""

def md_to_html(md_text):
    try:
        import markdown
        html = markdown.Markdown(extensions=['tables','fenced_code']).convert(md_text)
    except ImportError:
        html = md_text
        # Pipe tables
        def table_repl(m):
            rows = [r.strip() for r in m.group(0).strip().split('\n') if r.strip()]
            if len(rows) < 2: return m.group(0)
            hdr = [c.strip() for c in rows[0].strip('|').split('|')]
            out = '<table><thead><tr>' + ''.join(f'<th>{c}</th>' for c in hdr) + '</tr></thead><tbody>'
            for row in rows[2:]:
                cells = [c.strip() for c in row.strip('|').split('|')]
                out += '<tr>' + ''.join(f'<td>{c}</td>' for c in cells) + '</tr>'
            return out + '</tbody></table>'
        html = re.sub(r'(?:^\|.+\|$\n?){2,}', table_repl, html, flags=re.MULTILINE)
        html = re.sub(r'^### (.+)$', r'<h3>\1</h3>', html, flags=re.MULTILINE)
        html = re.sub(r'^## (.+)$', r'<h2>\1</h2>', html, flags=re.MULTILINE)
        html = re.sub(r'^# (.+)$', r'<h1>\1</h1>', html, flags=re.MULTILINE)
        html = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', html)
    def fix_img(m):
        alt, src = m.group(1), m.group(2)
        if src.startswith(('http','data:')): return f'<img src="{src}" alt="{alt}"/>'
        p = (PROJECT_ROOT / src).resolve()
        return f'<img src="file://{p}" alt="{alt}"/>' if p.exists() else f'<img src="{src}" alt="{alt}"/>'
    html = re.sub(r'!\[([^\]]*)\]\(([^)]+)\)', fix_img, html)
    html = re.sub(r'<img([^>]*)src="([^"]+)"', lambda m: m.group(0) if m.group(2).startswith(('http','file://','data:')) else m.group(0).replace(f'src="{m.group(2)}"', f'src="file://{(PROJECT_ROOT/m.group(2)).resolve()}"') if (PROJECT_ROOT/m.group(2)).resolve().exists() else m.group(0), html)
    return html

def generate_pdf(md_path, output_path):
    import fitz
    if not hasattr(fitz, 'Story'):
        print(f"Error: PyMuPDF >= 1.21 required for Story API (found {fitz.version[0]})", file=sys.stderr)
        print("Upgrade: pip install --upgrade PyMuPDF", file=sys.stderr)
        sys.exit(1)
    body = md_to_html(md_path.read_text(encoding='utf-8'))
    doc = f'<!DOCTYPE html><html><head><meta charset="UTF-8"><style>{PDF_CSS}</style></head><body>{body}</body></html>'
    story = fitz.Story(html=doc)
    writer = fitz.DocumentWriter(str(output_path))
    pw, ph, m = 595, 842, 72
    pages = 0; more = True
    while more:
        pages += 1; dev = writer.begin_page(fitz.Rect(0,0,pw,ph))
        more, _ = story.place(fitz.Rect(m,m,pw-m,ph-m)); story.draw(dev); writer.end_page()
    writer.close()
    print(f"OK: {output_path} ({output_path.stat().st_size/1024:.1f} KB, {pages} pages)")

if __name__ == '__main__':
    ap = argparse.ArgumentParser(); ap.add_argument('input', type=Path); ap.add_argument('-o','--output', type=Path)
    a = ap.parse_args(); i = Path(a.input).resolve()
    if not i.exists(): print(f"Error: {i}", file=sys.stderr); sys.exit(1)
    o = Path(a.output).resolve() if a.output else i.with_suffix('.pdf'); o.parent.mkdir(parents=True, exist_ok=True)
    generate_pdf(i, o)
```

### Phase 6: Designer Review (CSS-extraction pattern) — Optional with `--fast`

After HTML conversion, invoke an agent to generate CSS improvements. **Skip this phase if the user passed `--fast` or `--no-design` flag** (these flags may appear anywhere in the user's invocation message, e.g., `/exp-report --fast`) — proceed directly to Phase 6.5 (deliverable packaging still runs). **Critical**: use the **CSS-extraction pattern** to avoid Base64 image corruption from agent line truncation:

1. Agent SHOULD read the HTML file to understand its structure (DOM, class names, element types)
2. Agent outputs **ONLY a `<style>` block** with CSS improvements — must NOT reproduce or rewrite any HTML content
3. Lead agent injects the style block into the HTML `<head>` section

**Invoke designer agent:**
```
Agent(subagent_type="general-purpose", model="sonnet", prompt="
You are a CSS specialist reviewing an HTML experiment report for readability.

CRITICAL CONSTRAINT: Do NOT rewrite or output the full HTML file.
Your output must be ONLY a <style> block containing CSS improvements.

Read the HTML file at: data/F{NNN}/F{NNN}_report.html
Analyze its structure, then output ONLY a <style>...</style> block that improves:

1. TYPOGRAPHY
   - Font stack: system-ui, -apple-system, sans-serif
   - Line-height >= 1.6 for body, tighter for headings
   - Font-size hierarchy: h1 > h2 > h3 clearly distinct
   - Paragraph max-width ~70ch for readability
   - Bold verdict openers: color accent or left border to visually pop

2. TABLE READABILITY
   - Alternate row colors, sticky header, cell padding >= 0.5rem
   - Compact font-size (0.9rem) with adequate whitespace
   - Horizontal scroll wrapper: .table-wrapper { overflow-x: auto; }

3. RESPONSIVE LAYOUT (@media max-width: 768px)
   - Images: max-width 100%
   - TOC: vertical stack instead of horizontal flex
   - Tables: horizontal scroll on narrow screens
   - Adequate touch targets for <details> summaries (min-height 44px)

4. FIGURE PRESENTATION
   - Figures centered with max-width 85%
   - Captions: italic, smaller font (0.85em), centered below figure
   - Consistent spacing (margin: 1.5rem auto)

Output format — ONLY this, nothing else:
<style>
/* Designer review CSS improvements */
...your CSS here...
</style>
")
```

**After receiving CSS output, inject it:**
```python
# Read the HTML, inject the new <style> block before </head>
html = Path('data/F{NNN}/F{NNN}_report.html').read_text()
html = html.replace('</head>', f'{designer_css}\n</head>')
Path('data/F{NNN}/F{NNN}_report.html').write_text(html)
```

**If oh-my-claudecode is installed**, prefer `subagent_type="oh-my-claudecode:designer"` with the same CSS-only constraint.

**After designer review, re-verify:**
- HTML file size > 0
- All Base64 images still intact (post-injection size should be >= pre-injection size, since CSS only adds bytes; a decrease indicates corruption)
- Content unchanged (spot-check: Decision Table numbers match Tier 1)

### Phase 6.5: Deliverable Packaging

Create a self-contained deliverable folder that can be shared as-is to stakeholders. This phase runs unconditionally (no flag needed) — the report is always intended for sharing. The deliverable bundles the report with all referenced figures and an interpretation guide.

**Distinction**: `data/F{NNN}/` is the workspace (authoritative, all tier files + assembled report). `data/F{NNN}/deliverable/` is a sharing snapshot — portable, self-contained, stakeholder-friendly.

**Step 6.5a: Clean up previous run (idempotency) and create deliverable structure**
```python
import shutil
deliverable_dir = Path(f'data/F{NNN}/deliverable')
if deliverable_dir.exists():
    shutil.rmtree(deliverable_dir)  # Remove stale files from previous run
zip_path = Path(f'data/F{NNN}/F{NNN}_deliverable.zip')
zip_path.unlink(missing_ok=True)

deliverable_dir.mkdir(parents=True)
Path(f'data/F{NNN}/deliverable/tiers').mkdir()
# figures/ directory created conditionally in Step 6.5b (only if registry is non-empty)
```

**Step 6.5b: Copy figures from registry**
```python
figure_map = {}  # {original_path: deliverable_name}
if figure_registry:
    Path(f'data/F{NNN}/deliverable/figures').mkdir(exist_ok=True)
    for entry in figure_registry:
        src = Path(entry['original_path'])
        dst_name = entry['deliverable_name']
        if str(src) in figure_map:
            continue  # deduplicate: first-seen experiment wins
        figure_map[str(src)] = dst_name
        try:
            shutil.copy2(str(src), f'data/F{NNN}/deliverable/figures/{dst_name}')
        except FileNotFoundError:
            print(f"WARNING: Figure missing at packaging time, skipping: {src}")
```
**Zero-figure edge case**: If `figure_registry` is empty, `figures/` directory is not created. GUIDE.md omits the figures/ row (see step 6.5g).

**Step 6.5c: Copy data files from registry**
```python
data_map = {}  # {original_path: deliverable_name}
data_file_count = 0
if data_registry:
    Path(f'data/F{NNN}/deliverable/data').mkdir(exist_ok=True)
    for entry in data_registry:
        src = Path(entry['original_path'])
        dst_name = entry['deliverable_name']
        if str(src) in data_map:
            continue  # deduplicate: first-seen experiment wins
        try:
            shutil.copy2(str(src), f'data/F{NNN}/deliverable/data/{dst_name}')
            data_map[str(src)] = dst_name  # only on success
        except FileNotFoundError:
            print(f"WARNING: Data file missing at packaging time, skipping: {src}")
    data_file_count = len(data_map)
    if data_file_count == 0:
        # All files were missing — remove empty data/ directory
        shutil.rmtree(Path(f'data/F{NNN}/deliverable/data'), ignore_errors=True)
```
**Zero-data edge case**: If `data_registry` is empty, `data/` directory is not created. GUIDE.md omits the data/ row and "Source Data Files" section (see step 6.5g).

**Step 6.5d: Rewrite MD paths and copy report**
```python
md_content = Path(f'data/F{NNN}/F{NNN}_report.md').read_text()
# Normalize path variants before replacement (./data/ vs data/)
for original_path, deliverable_name in figure_map.items():
    # Replace both normalized and ./ prefixed forms
    md_content = md_content.replace(original_path, f'figures/{deliverable_name}')
    md_content = md_content.replace(f'./{original_path}', f'figures/{deliverable_name}')
# Warn if any unrewritten project-relative figure paths remain (broad pattern)
remaining = re.findall(r'data/[Ee]\d+/[^\s)]+\.(?:png|jpg|jpeg|gif|svg)', md_content)
if remaining:
    print(f"WARNING: {len(remaining)} figure paths not rewritten: {remaining}")
Path(f'data/F{NNN}/deliverable/F{NNN}_report.md').write_text(md_content)
```

**Step 6.5e: Copy HTML and PDF** (HTML is self-contained provided Phase 4 removed broken figure refs before conversion; verify HTML size > MD size as a proxy for successful Base64 embedding)
```python
shutil.copy2(f'data/F{NNN}/F{NNN}_report.html', f'data/F{NNN}/deliverable/')
pdf_path = Path(f'data/F{NNN}/F{NNN}_report.pdf')
has_pdf = pdf_path.exists() and pdf_path.stat().st_size > 0
if has_pdf:
    shutil.copy2(str(pdf_path), f'data/F{NNN}/deliverable/')
```

**Step 6.5f: Copy tier source files verbatim** (reference copies — workspace-relative paths remain intact)
```python
for tf in Path(f'data/F{NNN}').glob('tier*.md'):
    shutil.copy2(str(tf), f'data/F{NNN}/deliverable/tiers/')
```
**Note**: Tier files in `deliverable/tiers/` contain workspace-relative paths (`data/E{NNN}/...`) that resolve only in the original project directory. This asymmetry is documented in GUIDE.md (see step 6.5g).

**Step 6.5g: Generate GUIDE.md**

Select template based on `manifest_language` (Korean or English). Populate from Phase 0-5 computed data. **Write to**: `data/F{NNN}/deliverable/GUIDE.md`.

**English template:**
```markdown
# {PROJECT_TITLE}: Experiment Synthesis Report {F_NUMBER}

**Generated**: {today} | **Experiments**: {EXPERIMENT_COUNT} ({RANGE}) | **Version**: {F_NUMBER} v1.0

This package contains the complete synthesis report and all supporting materials.
Open the HTML file for the best reading experience — it includes embedded images,
a navigation bar, and collapsible sections.

---

## How to Read This Report

| Your role | Start here | What you'll find |
|-----------|------------|------------------|
| Non-specialist | Open HTML, read first section | Jargon-free summary |
| Decision-maker | Open HTML, read Decision Table | Verdicts + key numbers |
| Scientist / reviewer | Open HTML, read Evidence Narrative | Question-by-question evidence with figures |
| Engineer / replicator | Open HTML, expand Technical Reference | Pipeline spec, cross-reference, figure gallery |

---

## Files in This Package

| File | Description |
|------|-------------|
| `{F_NUMBER}_report.html` | **Open this.** Self-contained report with embedded images. |
{%- if has_pdf %}
| `{F_NUMBER}_report.pdf` | Print/archive version. |
{%- endif %}
| `{F_NUMBER}_report.md` | Source Markdown with relative figure paths. |
{%- if figure_count > 0 %}
| `figures/` | All referenced figures ({FIGURE_COUNT} images). |
{%- endif %}
{%- if data_file_count > 0 %}
| `data/` | Source data files ({DATA_FILE_COUNT} files). |
{%- endif %}
| `tiers/` | Individual tier source files (workspace-relative paths — for reference only). |

---

## Source Experiments

| Experiment | Description | Question(s) |
|------------|-------------|-------------|
{EXPERIMENT_TABLE_ROWS}

---
{%- if figure_count > 0 %}

## Figure Index

| File | Source | Question |
|------|--------|----------|
{FIGURE_TABLE_ROWS}

---
{%- endif %}
{%- if data_file_count > 0 %}

## Source Data Files

| File | Experiment | Format | Description |
|------|------------|--------|-------------|
{DATA_TABLE_ROWS}

---
{%- endif %}

*Generated by exp-report plugin v{PLUGIN_VERSION} on {today}.*
```

**Korean template** (complete — all strings translated):
```markdown
# {PROJECT_TITLE}: 실험 통합 보고서 {F_NUMBER}

**생성일**: {today} | **실험**: {EXPERIMENT_COUNT}건 ({RANGE}) | **버전**: {F_NUMBER} v1.0

이 패키지는 통합 보고서와 모든 관련 자료를 포함합니다.
HTML 파일을 열면 이미지 임베딩, 목차 탐색, 접이식 섹션이 포함된 최적의 읽기 환경을 제공합니다.

---

## 이 보고서 읽는 법

| 역할 | 시작 위치 | 내용 |
|------|----------|------|
| 비전문가 | HTML 첫 섹션 | 전문용어 없는 요약 |
| 의사결정자 | HTML 결정표 | 판단 + 핵심 수치 |
| 연구자 / 검토자 | HTML 근거 서술 | 질문별 근거와 그림 |
| 엔지니어 / 재현자 | HTML 기술 참조 (접기) | 파이프라인 사양, 교차 참조, 그림 갤러리 |

---

## 파일 목록

| 파일 | 설명 |
|------|------|
| `{F_NUMBER}_report.html` | **이 파일을 여세요.** 이미지가 포함된 자체 완결형 보고서. |
{%- if has_pdf %}
| `{F_NUMBER}_report.pdf` | 인쇄/보관용 버전. |
{%- endif %}
| `{F_NUMBER}_report.md` | 상대 경로 그림 참조가 포함된 마크다운 원본. |
{%- if figure_count > 0 %}
| `figures/` | 참조된 모든 그림 ({FIGURE_COUNT}개). |
{%- endif %}
{%- if data_file_count > 0 %}
| `data/` | 원본 데이터 파일 ({DATA_FILE_COUNT}개). |
{%- endif %}
| `tiers/` | 개별 계층 원본 파일 (작업 디렉토리 기준 경로 — 참조용). |

---

## 원본 실험

| 실험 | 설명 | 질문 |
|------|------|------|
{EXPERIMENT_TABLE_ROWS}

---
{%- if figure_count > 0 %}

## 그림 목록

| 파일 | 출처 | 질문 |
|------|------|------|
{FIGURE_TABLE_ROWS}

---
{%- endif %}
{%- if data_file_count > 0 %}

## 원본 데이터 파일

| 파일 | 실험 | 형식 | 설명 |
|------|------|------|------|
{DATA_TABLE_ROWS}

---
{%- endif %}

*exp-report 플러그인 v{PLUGIN_VERSION}으로 {today}에 생성됨.*
```

**Conditional rows**: Use `has_pdf` (boolean from step 6.5e), `figure_count` (from `len(figure_map)`), and `data_file_count` (from `len(data_map)`, computed in step 6.5c) to conditionally include/exclude PDF, figures/, and data/ rows. The `{%- if ... %}` notation is pseudo-Jinja — implement as Python string conditionals.

**`{DATA_TABLE_ROWS}` generation** (parallel to `{FIGURE_TABLE_ROWS}`):
```python
data_table_rows = []
for entry in data_registry:
    if entry['original_path'] in data_map:  # only successfully copied files
        data_table_rows.append(
            f"| `{entry['deliverable_name']}` | {entry['exp_id']} "
            f"| {entry['suffix'].upper().lstrip('.')} "
            f"| {entry.get('description', '')} |"
        )
DATA_TABLE_ROWS = '\n'.join(data_table_rows)
```
The `description` field is sourced from the MANIFEST experiment's `description` field (already normalized in Phase 0 step 3). If unavailable, leave empty. The `Format` column displays the uppercase file extension (e.g., CSV, JSON, PARQUET).

**Write instruction**:
```python
guide_content = ...  # populated template
Path(f'data/F{NNN}/deliverable/GUIDE.md').write_text(guide_content, encoding='utf-8')
```

**Step 6.5h: Create ZIP archive** (with top-level wrapper directory so extraction is tidy)
```python
import zipfile
zip_path = Path(f'data/F{NNN}/F{NNN}_deliverable.zip')
wrapper = f'F{NNN}_deliverable'  # top-level directory inside ZIP
with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zf:
    for f in Path(f'data/F{NNN}/deliverable').rglob('*'):
        if f.is_file():
            arcname = f'{wrapper}/{f.relative_to(Path(f"data/F{NNN}/deliverable"))}'
            zf.write(f, arcname)
```
This ensures extracting the ZIP creates a single `F{NNN}_deliverable/` folder rather than spilling files into the current directory.

**Step 6.5i: Verify and compute stats**
```python
deliverable_files = list(Path(f'data/F{NNN}/deliverable').rglob('*'))
file_count = sum(1 for f in deliverable_files if f.is_file())
total_size_kb = sum(f.stat().st_size for f in deliverable_files if f.is_file()) / 1024
figure_count = len(figure_map)
total_data_size_kb = sum(
    Path(f'data/F{NNN}/deliverable/data/{name}').stat().st_size
    for name in data_map.values()
    if Path(f'data/F{NNN}/deliverable/data/{name}').exists()
) / 1024 if data_map else 0
zip_size_kb = zip_path.stat().st_size / 1024
# Store for Phase 8 summary
deliverable_stats = {
    'path': f'data/F{NNN}/deliverable/',
    'zip_path': f'data/F{NNN}/F{NNN}_deliverable.zip',
    'file_count': file_count, 'total_size_kb': total_size_kb,
    'figure_count': figure_count, 'data_file_count': data_file_count,
    'total_data_size_kb': total_data_size_kb, 'zip_size_kb': zip_size_kb
}
```

**`--fast` handling**: Phase 6 skip instruction routes through Phase 6.5 — when `--fast` is set, skip Phase 6 but still execute Phase 6.5.

### Phase 7: Bookkeeping

1. **Append to MANIFEST.yaml:**
   ```yaml
   f{nnn}:
     path: data/F{NNN}/
     script: "scientist agents (3 parallel + 1 sequential: sonnet x2 + opus x1 + haiku x1)"
     params:
       structure: "4-Tier (Plain Language + Decision Brief + Evidence Narrative + Technical Reference)"
       source: "MANIFEST final experiments, restructured"
     outputs:
       - data/F{NNN}/F{NNN}_report.md
       - data/F{NNN}/F{NNN}_report.html
       - data/F{NNN}/F{NNN}_report.pdf
       - data/F{NNN}/tier0_plain_language.md
       - data/F{NNN}/tier1_decision_brief.md
       - data/F{NNN}/tier2_evidence_narrative.md
       - data/F{NNN}/tier3_technical_reference.md
       - data/F{NNN}/deliverable/
       - data/F{NNN}/F{NNN}_deliverable.zip
     deliverable: data/F{NNN}/deliverable/
     deliverable_zip: data/F{NNN}/F{NNN}_deliverable.zip
     status: final
     description: "F{NNN} Integrated Report: 4-Tier synthesis of {EXPERIMENT_COUNT} experiments"
     findings: "[Extract the first 2 sentences of the Tier 1 Executive Summary paragraph verbatim, truncated at 200 characters if necessary (add '...' suffix). Do not paraphrase.]"
     notes: "[HTML size, PDF size/pages, line count, data: {data_file_count} files]"
   ```

2. **Append to experiment-log.md** (if present):
   ```markdown
   ## {date}: F{NNN} generated (4-Tier Report)
   - **Structure**: Plain Language + Decision Brief + Evidence Narrative + Technical Reference
   - **Source**: {EXPERIMENT_COUNT} final experiments ({range})
   - **Outputs**: MD + HTML + PDF
   - **Status**: final
   ```

### Phase 8: Summary Output

Print to user:

```
=== F{NNN} Report Generated ===

Tier 0 (Plain Language):    {lines} lines
Tier 1 (Decision Brief):    {lines} lines
Tier 2 (Evidence Narrative): {lines} lines, {count of ![...] references in tier2} figures
Tier 3 (Technical Reference): {lines} lines

Assembled: data/F{NNN}/F{NNN}_report.md ({total_lines} lines)
HTML:      data/F{NNN}/F{NNN}_report.html ({size} KB)
PDF:       data/F{NNN}/F{NNN}_report.pdf ({size} KB, {pages} pages)

Deliverable: data/F{NNN}/deliverable/ ({total_size_kb} KB, {file_count} files)
  Figures:       {figure_count} images
  Data:          {data_file_count} files ({total_data_size_kb} KB)
  ZIP:           data/F{NNN}/F{NNN}_deliverable.zip ({zip_size_kb} KB)
  Contents:      MD + HTML + PDF + figures/ + data/ + tiers/ + GUIDE.md

Designer:      CSS/layout review applied
Verification:  {pass/fail} — {details}
MANIFEST:      updated
experiment-log: updated
```

## Agent Dispatch Table

| Phase | Agent | Model | Input | Output | Max Lines |
|-------|-------|-------|-------|--------|-----------|
| 1 | Lead (you) | — | MANIFEST findings | Candidate questions | — |
| 2 | Lead (you) | — | Confirmed questions + MANIFEST | Decision table | 10-15 |
| 3a | agent-1 | sonnet | Decision table | tier1_decision_brief.md | 20-30 |
| 3b | agent-2 | opus | Questions + MANIFEST + figures | tier2_evidence_narrative.md | 100-150 |
| 3c | agent-3 | haiku | MANIFEST + figures | tier3_technical_reference.md | 60-100 |
| 3d | agent-4 | sonnet | Tier 1 + MANIFEST descriptions | tier0_plain_language.md | 20-30 |
| | | | *(sequential after 3a)* | | |
| 4 | Lead (you) | — | 4 tier files | F{NNN}_report.md | — |
| 5 | Bash (parallel) | — | MD file | HTML + PDF | — |
| 6 | designer | sonnet | HTML file | Improved HTML (CSS/layout only) | — |
| 6.5 | Lead (you) | — | Figure + data registries + report files | deliverable/ + ZIP + GUIDE.md | — |
| 7 | Lead (you) | — | All outputs | MANIFEST + log updated | — |
| 8 | Lead (you) | — | — | Summary printed | — |

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Correct Approach |
|-------------|--------------|------------------|
| Listing experiments sequentially in Tier 1 | Redundant with Tier 3; Tier 1 is for decisions | Organize by decision/question, not by experiment |
| Hallucinated numbers | Numbers not in MANIFEST are fabricated | Every number must trace to MANIFEST `findings` |
| Skipping verification | Numeric inconsistencies across tiers | Always run Phase 4 verification |
| Using opus for all tiers | Wasteful; Tiers 1 and 3 are mechanical formatting | Opus only for Tier 2 (cross-experiment synthesis) |
| Hardcoding project-specific terms | Makes skill non-reusable | Use MANIFEST content as-is; no hardcoded drug names, class names, etc. |
| Generating figures in the report | Report synthesizes existing figures, not new ones | Select from figure registry only |
| Skipping user confirmation of questions | May generate irrelevant questions | Always present questions and wait for user confirmation |
| Including non-final or synthesis (f###) experiments | Draft/deprecated experiments pollute the report; synthesis entries create circular dependencies | Phase 0 must filter `status: final` only AND exclude keys matching `/^f\d+/i` |
| Figure numbering mismatches across tiers | Tier 2 says "Figure 3" but Tier 3 gallery has different numbering | Use consistent figure IDs tied to experiment IDs (e.g., "E005-Fig1") |
| Using stale MANIFEST data | MANIFEST may have changed since last read | Re-read MANIFEST at Phase 0 start; do not cache across sessions |
| Flat question list without hierarchy (including siloing statistical rigor as standalone) | Readers see no logical progression; a standalone "adequately powered?" question retroactively undermines all prior claims | Order as narrative arc: Claim→Mechanism→Boundary→Practical; integrate CI/power into claim or practical question |
| Unassigned experiments | Experiments missing from all questions produce incomplete synthesis | Check exhaustiveness: every final experiment in at least one question |
| Stopping after Phase 4 (MD only) | User gets raw Markdown without HTML/PDF — defeats the plugin purpose | Phase 4 is complete ONLY when Phase 5 has executed and HTML exists on disk |
| Designer agent rewriting full HTML | Base64-encoded images get silently corrupted by line truncation | Use CSS-extraction pattern: agent outputs `<style>` block only |
| Embedding large Base64 images without size guard | Reports with 20+ experiments produce >10MB HTML that degrades browser performance | If total figure registry exceeds 10MB, warn the user and consider linking external images instead of embedding |
| Using "significantly" with p ≥ 0.05 | Incorrect statistical language misleads readers | Scan Tier 2 for "significantly" and verify adjacent p-value is < 0.05; use "practical improvement (p=X, n.s.)" otherwise |
| Translating technical terms to Korean | Readers lose the ability to search, reference, or compare with English-language literature; translated terms like "곡선하면적" are unrecognizable | Preserve English for ALL-CAPS abbreviations, method names, units, and MANIFEST verbatim values; add Korean gloss in parentheses on first use for Tier 0 only |

## Verification Checklist

Before declaring completion, verify **ALL** of the following. **If any REQUIRED item fails, the report is NOT complete — do not print the Phase 8 summary.**

- [ ] All Tier 1 Decision Table numbers appear traceably in Tier 2
- [ ] All Tier 1 Decision Table numbers appear traceably in Tier 0 (with plain-language context)
- [ ] All figure paths in the report exist on disk (broken paths removed with warning)
- [ ] Tier 2 figure count matches the number of questions (1 per question where figures are available)
- [ ] Tier 3 cross-reference matrix lists ALL final experiments (including superseded)
- [ ] Every `status: final` experiment is assigned to at least one question
- [ ] All 4 tier files exist on disk and are non-empty (tier0, tier1, tier2, tier3)
- [ ] **REQUIRED**: `F{NNN}_report.html` exists and size > 0 bytes (if missing, go back to Phase 5)
- [ ] PDF file size > 0 bytes (optional — warn if missing, do not block)
- [ ] MANIFEST.yaml updated with F{NNN} entry
- [ ] experiment-log.md updated (if present)
- [ ] Only experiments with `status: final` are cited (whitelist — reject any other status)
- [ ] No "significantly" language used alongside p ≥ 0.05 in Tier 2
- [ ] All Tier 1 numbers trace to a MANIFEST `findings` or `notes` field (no hallucinated numbers)
- [ ] GUIDE.md exists in `data/F{NNN}/deliverable/` with correct F-number, date, and experiment count
- [ ] `data_registry.json` exists at `data/F{NNN}/data_registry.json` (serialized in Phase 0 step 9)
- [ ] `deliverable/data/` contains `{data_file_count}` files matching data_registry (or absent if data_file_count == 0)
- [ ] GUIDE.md "Source Data Files" / "원본 데이터 파일" section present IFF data_file_count > 0
- [ ] No symlinks in data_registry entries (all entries pass `not Path.is_symlink()`)
- [ ] Korean reports: technical terms preserved in English (AUC not 곡선하면적, p-value not p값) with `gloss_first` applied in Tier 0
- [ ] Korean reports: MANIFEST `findings` field values appear verbatim (not translated)
- [ ] `F{NNN}_deliverable.zip` exists and size > 0 bytes

**FAIL-SAFE**: If you reach Phase 8 without HTML output, STOP. Go back to Phase 5 and generate it.
