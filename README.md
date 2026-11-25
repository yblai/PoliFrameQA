# PoliFrameQA — Benchmark Dataset

Dataset release for the paper **"Can RAG-Based QA Stay Neutral? Selective Amplification of Informational Bias over Politically Framed Sources"** by Eason Lai, Xiandong Zou, Yujia Hu, Wei Gao, Michael Sejr Schlichtkrull, and Rui Cao, accepted to the main conference of EACL 2027.

PoliFrameQA is a fact-level benchmark for measuring whether retrieval-augmented QA systems inherit and selectively amplify **informational bias** — information conveyed as tangential, speculative, or as background to the main event in ways that sway readers' opinions towards the entities involved (Fan et al., 2019), as opposed to *lexical* bias in word choice. A fact labeled biased is **not thereby false**: the label concerns selective presentation, not factual accuracy, and *center* is used as a relative media-bias category, not as a synonym for neutrality.

## What each instance contains

Each of the **1,907** released instances pairs:

1. a **question** about a news issue;
2. an **article triplet** — left-, center-, and right-leaning coverage of that issue with expert outlet-stance labels (from AllSides);
3. three **stance reference answers**, each answering the question from one stance's article alone (generated once with GPT-4o-mini and then frozen); and
4. the **atomic facts** of every reference answer, partitioned by a two-stage, human-validated labeling pipeline into
   - **shared** facts (expressed by more than one stance — the non-partisan core),
   - **stance-specific** facts (expressed by one stance alone), and
   - the **informationally biased** subset of the stance-specific facts.

Stage 1 (*cross-source matching*) decides whether the same proposition is expressed by another stance; no bias judgment is made at this stage — the omitted counterpart of a stance-specific fact is its observed absence from the other stances, not a hypothetical. Stage 2 (*framing judgment*) decides, for stance-specific facts only and against the other two stances' paragraphs, whether presenting the fact could shape a reader's interpretation.

## Summary statistics

| Quantity | Value |
| --- | --- |
| Question–triplet instances | 1,907 (filtered from 3,066 AllSides triplets, ~62% retention) |
| Source articles | 5,721 (left/center/right, expert outlet-stance labels) |
| Atomic facts (whole benchmark) | ~71,400 |
| — shared across stances | ~47,000 |
| — stance-specific, neutral | ~19,200 |
| — stance-specific, informationally biased | ~5,200 |
| Per side and question (mean) | 12.5 facts: 8.2 shared / 4.3 stance-specific / 0.9 biased |
| Human validation, Stage 2 | 90 tasks / 594 facts / 5 annotators each (10 annotators total) |
| Human validation, Stage 1 | 25 triplets / 736 cover-uncover decisions / 2 annotators |

The array order of all `list[1907]` files is the canonical instance order and is identical across files (they are parallel arrays). The fixed *n* = 500 subsample used in the paper's subsample experiments is simply the **first 500 instances** in this order.

## Files

### `naacl2022_filtered_allsides_article.json` — source article triplets
`list[3066]`. AllSides article triplets as collected for the NAACL 2022 NeuS dataset. Each item: `left` / `right` / `center` (each with `newsSource`, `newsTitle`, `newsLink`, `newBody`, `stance`), plus `issue`, `topic`, `roundup`. The benchmark retains 1,907 of these after question generation and the per-article answerability filter.

### `instances.json` — benchmark instances
`list[1907]`. One item per instance: `question` plus the matched `left` / `right` / `center` articles (same schema as the source file) and `issue`, `topic`, `roundup`.

### `reference_answers.json` — frozen stance reference answers
`list[1907]`. Per instance: `left_answer`, `right_answer`, `center_answer` — the three stance reference answers that ground the bias landscape — with the corresponding `*_title` fields. Two auxiliary fields are included but **not used in the paper's benchmark construction**: `allsides_answer` (answering with all three articles as context; this is the reference generator's own all-sides answer, reported in the paper for completeness only and excluded from rankings) and `neutral_answer` (answering from the AllSides roundup summary).

### `reference_answer_facts.json` — atomic facts of the reference answers
`list[1907]`. The reference answers decomposed into FActScore-style atomic facts (`left_facts`, `right_facts`, `center_facts`; lists of decontextualized clause-level statements), alongside the answer fields above. `allsides_facts` and `neutral_facts` are auxiliary (see previous file).

### `stage1_coverage_partition.json` — Stage-1 output
`list[1907]`. Per instance: `question`, `all_facts`, and `per_side.{left,right,center}` with each stance's facts partitioned into `total` / `shared` / `unique` (stance-specific) by cross-source matching.

### `reference_bias_landscape.json` — Stage-2 output: the reference bias landscape
`list[1907]`. Per instance and stance: `shared` facts, `unique` (stance-specific) facts, and the Stage-2 partition of the latter into `bias` and `non_bias`. Each judged fact carries `body`, the calibrated `bias_probability` (fraction of a simulated five-annotator panel voting biased; hard label at threshold τ = 0.30), its component scores, and a brief `reason`. This file is the **fixed reference landscape** against which all evaluated systems are scored (Coverage, BiasRatio, per-stance SBias).

## Human validation subset (`human_annotation/`)

- **`stage2_fact_labels.json`** — self-contained per-fact table of the Stage-2 human round: `list[594]` with `idx`, `task_id`, `question`, `stance` of the focal reference answer (224 left / 188 right / 182 center), the `fact`, raw `votes` (`yes` / `no` / `already_covered`), the majority label (`label_majority`; 102/594 biased, 17.2%), and the `split` tag (`dev` n = 208 / `test` n = 386) used for detector validation. Randolph's free-marginal κ = 0.458 on the 430 facts with five binary votes.
- **`stage2_bias_annotation.json`** — the raw Label Studio export behind the table above: 90 tasks, ~5 independent judgments per fact (2,954 recorded judgments), three-way labels (`Yes` = biased / `No` = neutral / `Already covered`) with bias-mechanism categories justifying `Yes` votes.
- **`stage1_coverage_validation.json`** — raw Label Studio export of the separate Stage-1 round: 25 triplets, 736 fact-level cover/uncover decisions, 701 doubly annotated (inter-annotator agreement 83.5%, free-marginal κ = 0.669). The automatic Stage-1 matcher agrees with the human consensus on 518/619 decisions (83.7%).
- **`detector_devtest_split.json`** — the dev/test index split over `stage2_fact_labels.json` (indices correspond to `idx`), together with held-out detector results.

**Anonymization.** Annotator identities in both Label Studio exports are anonymized: emails are replaced by stable pseudonymous IDs (`annotator_NN@anonymized.example`) and name fields are blanked. Numeric rater IDs are preserved so all agreement statistics remain reproducible. Annotators were consenting and fairly compensated (US$15.67/hour).

## Intended use and caveats

The stance and bias labels are **descriptive research artifacts, not normative judgments**. They reflect one operationalization of informational bias, validated against — but not identical to — human consensus, and the automatic detector has moderate agreement with human annotators. The labels must **not** be used as ground truth for content moderation, deplatforming, or judgments about any outlet, journalist, or individual. The benchmark covers U.S. English political news only.

## Source content and licensing

Article text in this release consists of short issue-focused excerpts (on average ~90 words per article) with source names, titles, and links; copyright in the underlying articles remains with their publishers, and the excerpts are included solely to support research reproducibility, following the terms of AllSides and the NAACL 2022 dataset they were collected for. The questions, reference answers, atomic facts, fact-level annotations, and human validation data are released for research use; a formal license file will accompany the archived version.
