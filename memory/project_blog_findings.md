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

## Limitation: No baseline scores yet

**Finding:** All scores in `judged.json` are for the refined harness (`agg1`). There are no baseline scores (agent without Harness²) to compare against.

**Impact:** Cannot claim numerical improvement without running `--k 0` baseline on the same tasks.

**How to apply:** Either run a baseline pass before writing the blog, or frame results as "absolute performance of the refined harness" rather than "improvement over baseline."
