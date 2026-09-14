# DAX Performance Investigation

I want you to investigate and diagnose a DAX performance issue and propose a fix, using the files in the **current Power BI project**.

Work through the following steps in order. Do not skip steps. 
Ask me any questions before proceding.

---

## Ground Rules

- Do **not** modify any `.tmdl` files.
- Do **not** modify the Power BI project.
- Do **not** modify the VPAX source file.
- You may extract the `.vpax` into a temporary working directory for analysis.
- You may create or modify files under the current project's `notes/` folder.
- Clearly distinguish between:
  - observed facts
  - reasonable inferences
  - hypotheses that require validation
- Do not claim that an optimization is proven faster until it has been re-tested in DAX Studio.
- Preserve the semantic intent of the original measure when proposing a rewrite.
- Do not guess when important evidence is unavailable. State the limitation clearly.
- Do not read the entire `/kb` library by default. Search for and read only the relevant material needed for the investigation.

---

# Project Structure

The current project should have a structure similar to:

```text
<BigQueryAdventureWorksDW>/
├──  *.Report
├── *.SemanticModel
├── /Diagnostics
└── /Notes
    
```

The shared DAX reference library is located outside the project under:

```text
/kb
```

Before beginning, read `GEMINI.md` in the project root and apply the project-specific DAX optimization principles and conventions.

Use **project-relative paths** wherever possible. Do not assume a specific project name such as `bigquerydemo`.

---

# STEP 1 — Read the Model

Find and read the relevant TMDL files in the current project's semantic model.

Typically these will be under:

```text
<project>.SemanticModel/definition/tables/
```

Identify the two measures being compared.

If the measures cannot be identified confidently from the model, naming, comments, query files, diagnostics, or surrounding context, stop and ask me to identify them rather than guessing.

Show the two DAX definitions side by side.

For each measure, identify:

- referenced tables
- referenced columns
- iterators
- row context
- context transitions
- `FILTER` operations
- `CALCULATE` / `CALCULATETABLE` usage
- virtual tables
- callbacks
- table expansion
- repeated evaluation
- other potentially expensive constructs

Also note any important differences between the two measures that may explain their different performance.

---

# STEP 2 — Parse the VPAX

Locate the `.vpax` file under:

```text
/Diagnostics
```

Extract it into a temporary working directory without modifying the original file.

Read the relevant structural metadata.

For tables and columns referenced by the two measures, extract where available:

- row count
- cardinality
- column size
- dictionary size
- encoding
- data type
- other storage-related metadata relevant to the diagnosis

Do not dump the entire VPAX into the report.

Extract only the information relevant to the two measures and the performance diagnosis.

Remember:

> VPAX describes model/storage characteristics. It does not by itself prove what happened during execution of a particular DAX query.

Use runtime evidence from DAX Studio to establish actual execution behavior.

---

# STEP 3 — Analyze Server Timings

Locate the Server Timings screenshot under:

```text
/Diagnostics
```

Read the relevant screenshot(s).

Extract the available Server Timings information for each measure, including:

- total duration
- Storage Engine (SE) duration
- Formula Engine (FE) duration
- number of SE queries
- other relevant timing information visible in the screenshot

If a value cannot be read reliably from the screenshot, explicitly say so rather than estimating it.

Create a small comparison table:

| Metric | Fast Measure | Slow Measure |
|---|---:|---:|
| Total duration | | |
| SE duration | | |
| FE duration | | |
| SE queries | | |

If multiple timing captures exist, determine which ones are relevant before drawing conclusions.

---

# STEP 4 — Diagnose the Performance Difference

Combine the evidence from:

1. DAX definitions
2. TMDL/model structure
3. VPAX metadata
4. DAX Studio Server Timings
5. Relevant material from `/kb`
6. Query plans or other diagnostic files under `/Diagnostics`, if available

Determine **why the slow measure is slower**.

Do not simply list possible causes.

Investigate specifically whether the problem is associated with:

- Storage Engine work
- Formula Engine work
- excessive FE callbacks
- row-context iteration
- context transition
- materialization of a large intermediate table
- excessive SE queries
- poor filter propagation
- high-cardinality columns
- inefficient virtual-table construction
- repeated evaluation
- unnecessary table expansion
- non-pushable expressions
- inefficient filtering
- excessive data movement between FE and SE
- other model or DAX characteristics

Identify the **actual mechanism** responsible for the performance difference whenever the available evidence allows it.

For example, do not merely say:

> "The iterator is slow."

Instead explain the mechanism, such as:

> "The iterator causes Formula Engine evaluation over a large number of rows, resulting in repeated Storage Engine requests."

Only make that conclusion if the available evidence supports it.

---

## Rank the Causes

Rank the likely causes by strength of evidence.

For each major conclusion, state:

**Evidence:**  
What was actually observed.

**Interpretation:**  
What that evidence means.

**Confidence:**  
High / Medium / Low.

Clearly distinguish:

- what we know
- what we infer
- what remains a hypothesis

Where useful, cross-reference relevant optimization principles from the PDFs and books in `/kb`.

Do not read entire books or slide decks unnecessarily. Search for relevant concepts first and read only the sections required to support the diagnosis.

---

# STEP 5 — Propose a Fix

Rewrite the slow measure into an optimized version.

Preserve the semantic intent of the original measure.

Then explain:

1. What changed.
2. Why the original pattern was expensive.
3. What execution mechanism caused the additional cost.
4. Why the new pattern should reduce FE or SE work.
5. Whether the rewrite changes materialization, iteration, callbacks, context transition, or filter propagation.
6. What trade-offs exist.
7. What semantic risks exist.
8. Whether the optimization depends on assumptions about the data/model.

Do not optimize merely because the new DAX looks cleaner.

Do not assume that replacing a function with another function automatically makes the measure faster.

The optimization must be tied to a specific performance mechanism.

Do not claim that the new measure is definitely faster.

Instead state:

- expected performance impact
- what evidence supports that expectation
- what must be measured to prove it

---

# STEP 6 — Define the Validation Test

Give me an exact DAX Studio test procedure.

Include:

- the exact measure/query to execute
- the DAX Studio settings to enable
- whether to clear the cache
- how many runs to perform
- whether to discard the first run
- which metrics to compare
- what result would constitute meaningful improvement

The key comparison should include:

- total duration
- FE duration
- SE duration
- SE query count

Where relevant, also compare:

- FE CPU
- SE CPU
- number of Storage Engine scans
- callbacks
- materialization
- query plan changes
- rows processed

Explain what result would indicate that the original diagnosis was wrong.

For example:

- If FE time does not materially decrease despite removing the suspected FE-heavy pattern, reconsider the diagnosis.
- If SE time increases substantially, investigate whether the rewrite shifted work from FE to SE.
- If total duration does not improve, do not claim the optimization was successful.
- If the semantic result changes, treat the rewrite as incorrect even if it is faster.

The final judgment must be based on both:

**semantic correctness + measured performance.**

---

# STEP 7 — Write the Investigation Report

Write the final report to:

```text
notes/dax-optimization-findings.md
```

Do not write to a root-level `/Notes` folder.

The `notes/` folder belongs to the **current project**.

The report should contain:

1. Executive Summary
2. Measures Compared
3. Original DAX
4. Model / TMDL Evidence
5. VPAX Evidence
6. Server Timings Evidence
7. Performance Comparison Table
8. Diagnosis
9. Ranked Causes
10. Evidence and Confidence
11. Proposed Optimized DAX
12. Explanation of the Optimization
13. Semantic Correctness Considerations
14. DAX Studio Validation Procedure
15. Expected Result
16. Remaining Uncertainties
17. Final Recommendation

Use concise tables where they make the evidence easier to understand.

Do not include irrelevant VPAX metadata or large dumps of model information.

---

# STEP 8 — Final Response

After writing the report, give me a concise summary containing:

### Finding

What is actually making the slow measure slower?

### Evidence

What are the strongest pieces of evidence supporting the diagnosis?

### Fix

What DAX rewrite do you recommend?

### Confidence

High / Medium / Low, with a brief explanation.

### Next Test

Tell me exactly what I should test next in DAX Studio.

Do not claim the optimization is successful until the validation test has been performed.

---

# Important Principles

Throughout the investigation:

- Start with evidence, not assumptions.
- Prefer runtime evidence over speculation.
- Treat Server Timings as evidence of actual query execution.
- Treat VPAX as structural/model evidence, not proof of runtime behavior.
- Understand whether work is being performed by the Formula Engine or Storage Engine.
- Look for the mechanism causing FE/SE work rather than blaming individual DAX functions.
- Consider cardinality and materialization.
- Consider callbacks and repeated Storage Engine requests.
- Consider context transition and row-context iteration.
- Consider whether filters can be pushed efficiently to the Storage Engine.
- Preserve semantic correctness before optimizing.
- Validate performance empirically.
- State uncertainty explicitly.
- Do not modify source model files unless explicitly instructed.
- Keep project-specific findings in the current project's `notes/` folder.
- Keep reusable/general DAX knowledge in `/kb`, not in project notes.
- Do not mix evidence or measures from different Power BI projects.