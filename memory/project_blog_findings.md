---
name: blog-findings
description: Findings, limitations, and observations from the Harness² test run, to be used when writing the research blog post
metadata:
  type: project
---

## Limitation: Missing system dependencies cause silent task failure

**Finding:** Tasks requiring Excel file processing (`.xlsx`) failed silently because `libreoffice` was not installed in the minimal Docker container. The framework correctly identified the problem and proposed a harness-level fix (an `excel_to_csv` skill), but could not overcome the missing system dependency.

**Specific task:** `computer_and_information_systems_managers/task1`

**What happened:** Base run failed to read `.xlsx` → domain probe added a skill to fix it → skill also failed (no libreoffice) → composed harness ran but produced no output → selection found no artifact → task unscored (not a zero, just missing).

**What makes this interesting for the blog:** The system correctly diagnosed and attempted to fix the problem at the harness level — that's the framework working as designed. The failure was environmental, not algorithmic. Fix: add `apt-get install -y libreoffice` to container setup.

**How to apply:** Flag as a limitation in the blog post under "running in a minimal Docker environment." Note that production use would require a more fully-provisioned execution environment.

---

## Observation: 0% weighted scores despite passing individual criteria

**Finding:** Several tasks scored 0 weighted points despite passing a meaningful number of individual criteria (e.g., computer_research_scientists/task1 passed 14/31 criteria but scored 0 weighted).

**Likely cause:** JobBench's rubric uses weighted scoring where certain mandatory criteria carry all the weight. Failing those collapses the total to zero even when many sub-criteria pass.

**How to apply:** When reporting scores in the blog post, report both `criteria_passed/criteria_count` and the weighted `frac` score. Do not rely on weighted score alone — it can be misleading without context.

---

## Observation: Harnessing-level memory was conservative (NO_ENTRY)

**Finding:** After task 1 (biostatisticians), the experience reflector wrote `NO_ENTRY` — it decided there wasn't enough unconfounded evidence to write a lesson to the improvement harness.

**Why:** Multiple changes happened simultaneously across probes, making it impossible to isolate cause and effect. The reflector is deliberately conservative.

**How to apply:** Note in the blog post that harnessing-level learning requires more tasks and cleaner signal to accumulate. With only 3 tasks in a domain, lessons may not propagate. This is expected behavior, not a bug.

---

## Observation: Refined harness produced more output than base

**Finding:** For biostatisticians/task1, the refined harness (`agg1`) produced 5 files vs. the base's 3 — including an additional written recommendation and a CSV of underlying data.

**What changed:** Domain probe added two new skills: `missing_data_handling` and `regulatory_guidance_citation`, triggered by explicit task requirements (FDA ICH E9 guidance, missing data assessment).

**How to apply:** Use as the concrete "before/after" example in the blog post. The harness learned to trigger specialist knowledge at the right moment.

---

## Observation: Task complexity ceiling — no improvement for biostatisticians/task2 with Gemini Flash, seq k=2

**Experiment:** `jb-oc-agg2` / `jb-oc-seq2`, seq mode, k=2, gemini-2.5-flash executor+improver+judge, Oct 26 2026.

**Result:** biostatisticians/task2 scored 0/58 weighted at every run — base1, base2, c1, c2 all failed. biostatisticians/task1 reached 20/72 (27.8%) but only matched the better base run (no net gain).

**Why task2 failed entirely:** Judge evaluated 7 rubrics, all failed:
- LOCF imputation not applied (30 subjects with missing Week 24 had no Week 12 carry-forward)
- Conditional power trivially computed as `1` instead of using actual Week 12 interim data
- SD discrepancy not flagged in Audit (protocol assumed 1.2%; observed differed; field left blank)
- No actual forest plot rendered — only a table; rubric required a visual with 95% CIs
- No clinical interpretation narrative for subgroup interactions — only p-values, no text
- RWE A1C testing proportions incorrect (computed tested/total where both were identical)
- Medication utilization mapping broken (GLP-1 = 0 in RWE despite relevant data)

**Root cause:** The task requires: LOCF imputation, forest plot image generation, narrative clinical interpretation, and downloading + processing the UCI Diabetes 130-US dataset (101k rows). Gemini Flash hit a capability ceiling — the harness improvements were directionally correct (added domain memory, merged task1 learnings) but couldn't overcome model execution limits on this task's complexity.

**What makes this interesting for the blog:** Demonstrates that Harness² learns and adapts the right things but cannot manufacture capability the underlying model doesn't have. The ceiling is the model, not the framework. Use as an honest limitation: "adaptation helps when the model can execute the task; it cannot substitute for missing model capability."

**How to apply:** Report task2 as a ceiling case. Contrast with task1 where partial improvement was visible. Note that running with a stronger model (e.g., Opus or Sonnet) on the same task would be a useful ablation.

---

## Limitation: No baseline scores yet (early runs only)

**Finding:** The early biostatisticians-only runs had no baseline. This was resolved in runs_full which ran `--bases` judging.

**How to apply:** Use runs_full base_judged.json as the baseline for comparison in the blog.

---

## Experiment: runs_full — Pro k=1 parallel, all 65 tasks (Oct 29–31 2026)

**Config:** gemini-2.5-pro executor + improver + judge, k=1 parallel mode, all 65 JobBench tasks, `runs_full/`.

**Overall result:** Avg base 6.6% → avg adapted 4.6% — **net -2.0 points**. Harness² hurt on average at k=1.

**Breakdown (52 graded tasks):**
- 2 tasks improved (producers/task1 +8pts, purchasing_agents/task1 +4.5pts)
- 3 tasks flat
- 15 tasks regressed — including dramatic drops: biostatisticians/task1 -50pts, data_entry_keyers/task2 -33pts
- 32 tasks had zero base score; 10 of these were unlocked by adaptation (best: social_science_research_assistants/task2 +23.5pts)

**Root cause of regressions — the k=1 verification gap:**
At k=1 there is only one base rollout. The harness editor diagnoses failures and adds scaffolding (skills, scripts, triggers) — but there is no second probe to verify the scaffolding works before it gets merged into the adapted harness. Two concrete examples:
- `biostatisticians/task1`: base scored 62% (5/8 rubrics). Adapted harness introduced a `biostat` skill that **failed to load** during the adapted run. Agent spent budget trying to invoke a broken skill → 4 previously-passing rubrics dropped to 0 → 12% final.
- `data_entry_keyers/task2`: base scored 37%. Adapted harness added a `data_reconciliation` skill with a corrected ICD-10 URL. The skill worked but over-structured the agent's output, causing it to follow the script instead of directly answering rubric questions → 0% final.

**Key insight:** k=1 is insufficient. The framework correctly diagnoses problems but the fix is applied without verification. k=2 adds a second Propose–Probe–Compose round that would catch broken skills before adoption. This explains why the paper's reported gains used higher k settings.

**What makes this interesting for the blog:** The regression pattern is itself a finding — it reveals the mechanism by which k matters. Not just "more k = better" but specifically: k=1 edits are unverified hypotheses; k=2+ edits are verified by a second probe. The biostatisticians/task1 case is a perfect illustration.

**Pending experiments:**
- `runs_full_k2`: Pro, k=2 parallel, all 65 tasks — tests the verification-gap hypothesis
- Flash k=1 parallel, all 65 tasks — isolates model contribution vs. k contribution

---

## Experiment log summary (to date)

| Run | Model | Mode | k | Tasks | Avg base | Avg adapted | Delta |
|---|---|---|---|---|---|---|---|
| runs/ (agg1) | Flash | parallel | 1 | 9 | ~14% | 15% | +1pt (small sample) |
| runs/ (seq2) | Flash | seq | 2 | 2 (biostats) | ~19% | 14% | -5pt |
| runs_pro/ | Pro | seq | 2 | 2 (biostats) | ~9% | 4% | -5pt |
| runs_full/ | Pro | parallel | 1 | 52 graded | 6.6% | 4.6% | -2pt |
| runs_full_k2/ | Pro | parallel | 2 | 65 | TBD | TBD | pending |
| Flash k=1 full | Flash | parallel | 1 | 65 | TBD | TBD | planned |
