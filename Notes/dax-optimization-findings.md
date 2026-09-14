# DAX Performance Investigation Report: `Total Sales Red` vs `Total Sales Red BAD`

---

## 1. Executive Summary

An investigation was conducted on the semantic model **BigQueryAdventureWorksDW** to diagnose the performance disparity between two measures calculating red product sales:
- **Fast Measure:** `Total Sales Red` (Total Duration: **4 ms**, SE Duration: **1 ms**, FE Duration: **3 ms**, SE Queries: **1**, Materialized Rows: **1**)
- **Slow Measure:** `Total Sales Red BAD` (Total Duration: **107 ms**, SE Duration: **73 ms**, FE Duration: **34 ms**, SE Queries: **3**, Materialized Rows: **120,797**)

The slow measure is **~27x slower in total elapsed duration**, incurs **73x more Storage Engine (SE) duration**, consumes **11x more Formula Engine (FE) duration**, and materializes **over 120,000 intermediate rows (~11.5 MB)** from the Storage Engine compared to a single 1-row (1 KB) cache in the fast measure.

The root cause is an **anti-pattern combining fact-table iteration (`SUMX` over all 60,398 rows of `FactInternetSales`) with row-level context transition (`CALCULATE([Total Sales])`) and procedural row filtering (`RELATED` inside `IF`)**. This forces VertiPaq to abandon bulk aggregation and instead generate un-aggregated, granular datacaches for all 60,398 fact rows twice, shifting massive data movement and evaluation onto the single-threaded Formula Engine.

Replacing the procedural row iterator with a set-based filter via `CALCULATE ( [Total Sales], DimProduct[Color] = "Red" )` enables the VertiPaq Storage Engine to push the color predicate directly across the relationship and compute the entire aggregation in a single multi-threaded scan returning 1 row.

---

## 2. Measures Compared

Both measures reside in the `FactInternetSales` table and aim to calculate sales filtered to products where `DimProduct[Color] = "Red"`.

- **Fast Baseline:** `[Total Sales Red]` — Uses `CALCULATE` with a boolean column filter predicate on `DimProduct[Color]`.
- **Problem Measure:** `[Total Sales Red BAD]` — Uses `SUMX` iterating across `FactInternetSales`, evaluating `RELATED(DimProduct[Color]) = "Red"` inside an `IF` condition, and invoking `CALCULATE([Total Sales])` for matching rows.

Both measures reference the underlying base measure:
```dax
measure 'Total Sales' = 
SUMX ( 
    FactInternetSales, 
    FactInternetSales[UnitPrice] * FactInternetSales[SalesAmount] 
)
```

---

## 3. Original DAX Definitions

### Side-by-Side Comparison

```dax
/* =========================================================================
   FAST MEASURE: Total Sales Red
   ========================================================================= */
measure 'Total Sales Red' = 
CALCULATE (
    [Total Sales],
    DimProduct[Color] = "Red"
)

/* =========================================================================
   SLOW MEASURE: Total Sales Red BAD
   ========================================================================= */
measure 'Total Sales Red BAD' = 
SUMX (
    FactInternetSales,
    IF (
        RELATED ( DimProduct[Color] ) = "Red",
        CALCULATE ( [Total Sales] )
    )
)
```

### Component Analysis

| Aspect | Fast Measure (`Total Sales Red`) | Slow Measure (`Total Sales Red BAD`) |
| :--- | :--- | :--- |
| **Outer Construct** | `CALCULATE` (Modifies Filter Context) | `SUMX` (Iterates Table in Row Context) |
| **Iterated Table** | None (Base measure iterates `FactInternetSales` under filtered context) | `FactInternetSales` (All 60,398 rows iterated explicitly) |
| **Row Context** | None at measure outer level | Active row context on `FactInternetSales` |
| **Context Transition** | None at measure outer level; none in filter argument | **Yes**: `CALCULATE([Total Sales])` invoked inside the row context of `FactInternetSales` |
| **Filter Method** | Set-based predicate (`DimProduct[Color] = "Red"`) pushed to filter context | Procedural check: `IF ( RELATED(DimProduct[Color]) = "Red", ... )` |
| **Relationship Traversal** | Automatic filter propagation: `DimProduct[ProductKey]` (1) -> `FactInternetSales[ProductKey]` (*) | Row-by-row lookup via `RELATED` across relationship |
| **Formula Engine Work** | Negligible (receives final scalar from SE) | High (evaluates `IF`, `RELATED`, and row transitions for 60,398 rows) |
| **Storage Engine Work** | 1 scan, aggregates directly in SE | 3 scans, materializes un-aggregated row-level datacaches |

---

## 4. Model & TMDL Evidence

From `definition/relationships.tmdl` and `definition/tables/`:

1. **Relationship Definition:**
   - **From:** `FactInternetSales[ProductKey]` (Many)
   - **To:** `DimProduct[ProductKey]` (One)
   - **State:** `IsActive = True`, `CrossFilteringBehavior = OneDirection`
   - **Join On Date Behavior:** `DateAndTime`
2. **Filter Propagation:**
   - Filters on `DimProduct` naturally propagate down to `FactInternetSales` via the active 1-to-many relationship.
   - A filter applied to `DimProduct[Color] = "Red"` restricts `DimProduct[ProductKey]`, which automatically restricts the visible rows in `FactInternetSales`.
3. **Table Grain & Schema:**
   - `FactInternetSales` is an import fact table containing 22 columns including high-cardinality transaction keys (`SalesOrderNumber`, `CustomerKey`, `OrderDateKey`, etc.).
   - `DimProduct` is a dimension table with 25 columns.

---

## 5. VPAX Structural Evidence

Extracted directly from `BigQueryAdventureWorks.vpax` (`DaxVpaView.json`):

### Table Statistics

| Table Name | Row Count | Total Size (Bytes) | Total Size (MB) | Column Count |
| :--- | ---:| ---:| ---:| ---:|
| **FactInternetSales** | 60,398 | 1,999,819 | ~1.91 MB | 22 |
| **DimProduct** | 606 | 1,333,363 | ~1.27 MB | 25 |
| **DimCustomer** | 18,484 | 3,684,236 | ~3.51 MB | 31 |

### Column Statistics (Referenced & Key Columns)

| Table[Column] | Cardinality | Total Size (Bytes) | Data Size (Bytes) | Dictionary Size | Type | Encoding |
| :--- | ---:| ---:| ---:| ---:| :--- | :--- |
| **DimProduct[Color]** | **10** | 17,708 | 336 | 17,276 | String | HASH |
| **DimProduct[ProductKey]** | **606** | 3,520 | 944 | 144 | Int64 | VALUE |
| **FactInternetSales[ProductKey]** | **158** | 38,872 | 32,592 | 5,000 | Int64 | HASH |
| **FactInternetSales[UnitPrice]** | **42** | 17,304 | 15,336 | 1,616 | Double | HASH |
| **FactInternetSales[SalesAmount]** | **42** | 17,304 | 15,336 | 1,616 | Double | HASH |
| **FactInternetSales[SalesOrderNumber]** | **27,659** | 1,174,927 | 120,928 | 832,719 | String | HASH |
| **FactInternetSales[CustomerKey]** | **18,484** | 195,016 | 120,928 | 144 | Int64 | VALUE |

### Structural Insights:
- `DimProduct[Color]` has a cardinality of only **10 distinct values**. Filtering on `"Red"` filters a tiny fraction of the dimension.
- `DimProduct` has only **606 rows**, while `FactInternetSales` has **60,398 rows**.
- Iterating `FactInternetSales` processes 60,398 rows, whereas filtering on `DimProduct[Color]` filters down to a small subset of 606 product keys that filter `FactInternetSales` via VertiPaq bitmapped indexes.

---

## 6. Server Timings Evidence

Extracted from DAX Studio diagnostic captures (`good query.png` and `Bad query.png`):

### Good Query (`Total Sales Red`) Server Timings
```dax
// DAX Query
EVALUATE
ROW(
    "Total_Sales_Red", 'FactInternetSales'[Total Sales Red]
)
```
- **Total Duration:** 4 ms
- **Formula Engine (FE) Duration:** 3 ms (75.0%)
- **Storage Engine (SE) Duration:** 1 ms (25.0%)
- **SE CPU:** 0 ms (x0.0)
- **SE Queries:** 1
- **SE Cache:** 0 (0.0%)
- **Storage Engine Scans:**
  - **Line 2 (Scan):** Duration: 1 ms, CPU: 0 ms, Par.: (none), **Rows: 1**, **KB: 1**
  - **Line 3 (ExecutionMetrics):** Duration: 0 ms, CPU: 0 ms

### Bad Query (`Total Sales Red BAD`) Server Timings
```dax
// DAX Query
EVALUATE
ROW(
    "Total_Sales_Red_BAD", 'FactInternetSales'[Total Sales Red BAD]
)
```
- **Total Duration:** 107 ms
- **Formula Engine (FE) Duration:** 34 ms (31.8%)
- **Storage Engine (SE) Duration:** 73 ms (68.2%)
- **SE CPU:** 78 ms (x1.1)
- **SE Queries:** 3
- **SE Cache:** 0 (0.0%)
- **Storage Engine Scans:**
  - **Line 2 (Scan):** Duration: 37 ms, CPU: 31 ms, Par.: x0.8, **Rows: 60,398**, **KB: 5,663** (xmSQL `SELECT...`)
  - **Line 4 (Scan):** Duration: 28 ms, CPU: 31 ms, Par.: x1.1, **Rows: 60,398**, **KB: 5,899** (xmSQL `WITH...`)
  - **Line 6 (Scan):** Duration: 8 ms, CPU: 16 ms, Par.: x2.0, **Rows: 1**, **KB: 1** (xmSQL `WITH...`)
  - **Line 7 (ExecutionMetrics):** Duration: 0 ms, CPU: 0 ms

---

## 7. Performance Comparison Table

| Metric | Fast Measure (`Total Sales Red`) | Slow Measure (`Total Sales Red BAD`) | Absolute Variance | Relative Variance |
| :--- | ---:| ---:| ---:| ---:|
| **Total Duration** | **4 ms** | **107 ms** | +103 ms | **26.8x slower** |
| **SE Duration** | **1 ms** | **73 ms** | +72 ms | **73.0x slower** |
| **FE Duration** | **3 ms** | **34 ms** | +31 ms | **11.3x slower** |
| **SE CPU** | **0 ms** | **78 ms** | +78 ms | N/A (inf.) |
| **SE Queries** | **1** | **3** | +2 queries | **3.0x queries** |
| **Total Rows Materialized** | **1 row** | **120,797 rows** | +120,796 rows | **120,797x more rows** |
| **Total Data Materialized** | **1 KB** | **11,563 KB** (~11.3 MB) | +11,562 KB | **11,563x more data** |

---

## 8. Diagnosis

The dramatic slowdown in `Total Sales Red BAD` is caused by the interaction of **four reinforcing anti-patterns**:

### A. Fact-Table Iteration with Row-Level Context Transition
`Total Sales Red BAD` iterates the fact table:
```dax
SUMX ( FactInternetSales, ... CALCULATE ( [Total Sales] ) )
```
When `CALCULATE` is evaluated inside the row context of `FactInternetSales`, it executes **context transition**. Context transition transforms the active row context—every single column value for that row across all 22 columns of `FactInternetSales`—into a filter context. 

Because `FactInternetSales` contains 60,398 rows, context transition is triggered **60,398 times**. As explained in *The Definitive Guide to DAX* (Chapter 20, p. 698-704), whenever context transition occurs inside an iterator, the Storage Engine cannot aggregate the result into a single scalar. It is forced to materialize datacaches down at the granularity of the iterated table.

### B. Massive Intermediate Materialization
Because the engine must provide row-level inputs to the Formula Engine to resolve the row-by-row `IF` and context-transitioned measure, the Storage Engine issues two separate full-table scans:
1. Scan 1: Materializes **60,398 rows (5,663 KB)**.
2. Scan 2: Materializes **60,398 rows (5,899 KB)**.

In total, **120,797 rows and over 11.5 MB of uncompressed memory** are transferred from the Storage Engine to the Formula Engine.

In contrast, `Total Sales Red` passes `DimProduct[Color] = "Red"` as a filter argument to `CALCULATE`. The Storage Engine evaluates `SUM(UnitPrice * SalesAmount)` filtered by `DimProduct[Color] = 'Red'` entirely inside VertiPaq, returning **1 row (1 KB)** in **1 ms**.

### C. Formula Engine Bottleneck (Iterative `IF` and `RELATED`)
An `IF` statement cannot be natively computed inside the VertiPaq Storage Engine (*The Definitive Guide to DAX*, p. 704). By embedding `IF ( RELATED(...) = "Red", ... )` inside `SUMX`, every row must be evaluated by the Formula Engine. The Formula Engine is single-threaded and significantly slower than the multi-threaded in-memory columnar Storage Engine. This causes FE duration to inflate from 3 ms to 34 ms.

### D. Reversing the Dimensional Filter Direction
Instead of applying a set filter on the dimension table (`DimProduct`) and letting the relationship filter the fact table, `Total Sales Red BAD` scans the entire fact table and uses `RELATED` to look up the color of each fact row. This inverts standard dimensional modeling execution.

---

## 9. Ranked Causes

1. **Unnecessary Context Transition over Fact Table (Primary Bottleneck):**
   - Calling `CALCULATE([Total Sales])` inside the row context of `FactInternetSales` forces row-level granularity evaluation across 60,398 rows.
2. **Massive Storage Engine Materialization (Direct Consequence):**
   - The SE must materialize two datacaches of 60,398 rows each (totaling 11.5 MB) rather than aggregating to 1 row (1 KB).
3. **Formula Engine Iteration & Procedural IF Evaluation:**
   - The FE is forced to iterate through 60,398 rows to evaluate the conditional branch and invoke measure evaluation.
4. **Row-by-Row Relationship Traversal via RELATED:**
   - Procedurally looking up `DimProduct[Color]` for each fact row rather than filtering `DimProduct` upstream.

---

## 10. Evidence and Confidence

| Level | Finding | Source / Evidence | Confidence |
| :--- | :--- | :--- | :--- |
| **Observed Fact** | `Total Sales Red` runs in 4 ms with 1 SE query, returning 1 row (1 KB). | DAX Studio capture: `good query.png` | **High** |
| **Observed Fact** | `Total Sales Red BAD` runs in 107 ms with 3 SE queries, materializing 60,398 rows twice (~11.5 MB). | DAX Studio capture: `Bad query.png` | **High** |
| **Observed Fact** | `FactInternetSales` contains exactly 60,398 rows; `DimProduct` contains 606 rows with 10 distinct colors. | VPAX extraction (`DaxVpaView.json`) | **High** |
| **Observed Fact** | Active 1-to-many relationship exists from `DimProduct[ProductKey]` to `FactInternetSales[ProductKey]`. | TMDL: `relationships.tmdl` | **High** |
| **Interpretation** | The 60,398 materialized rows in Lines 2 & 4 of `Bad query.png` correspond directly to the row count of `FactInternetSales`. | Match between VPAX row count (60,398) and SE rows returned (60,398) | **High** |
| **Interpretation** | `CALCULATE([Total Sales])` inside `SUMX(FactInternetSales, ...)` causes context transition that prevents SE aggregation. | *The Definitive Guide to DAX*, Ch. 7 & Ch. 20 | **High** |
| **Interpretation** | Set-based filtering via `CALCULATE` allows complete SE pushdown and aggregate elimination. | Server Timings line 2 in `good query.png` (1 row, 1 ms) | **High** |

---

## 11. Proposed Optimized DAX

The slow measure should be rewritten to adopt the set-based filter pattern established by the fast measure.

### Recommended Measure Definition

```dax
measure 'Total Sales Red' = 
CALCULATE (
    [Total Sales],
    KEEPFILTERS ( DimProduct[Color] = "Red" )
)
```

*(Note: If strict overwrite of any outer `DimProduct[Color]` filter is desired, omit `KEEPFILTERS`, exactly matching the existing fast measure):*

```dax
measure 'Total Sales Red' = 
CALCULATE (
    [Total Sales],
    DimProduct[Color] = "Red"
)
```

---

## 12. Explanation of the Optimization

1. **Elimination of Fact Table Iteration:**
   - The outer `SUMX(FactInternetSales, ...)` is completely removed. There is no row context on `FactInternetSales` created by the outer measure.
2. **Elimination of Context Transition:**
   - Because there is no row context active when `[Total Sales]` is evaluated, no context transition occurs. The engine does not have to create a 22-column filter context for every fact row.
3. **Elimination of Intermediate Materialization:**
   - VertiPaq can push the filter `DimProduct[Color] = "Red"` down to the Storage Engine. The Storage Engine joins `DimProduct` and `FactInternetSales` via `ProductKey`, filters the compressed column segments in memory, computes `SUM(UnitPrice * SalesAmount)` using multi-threaded vectorized SIMD instructions, and returns **a single aggregated value**. Materialization drops from 120,797 rows to 1 row.
4. **Elimination of Formula Engine Overhead:**
   - The Formula Engine no longer executes 60,398 iterations of `IF` and `RELATED`. FE duration drops from 34 ms to ~3 ms.
5. **Trade-offs:**
   - There are no negative trade-offs. The set-based pattern is idiomatic DAX, consumes vastly less memory, executes in a single SE scan, and scales linearly with data size.

---

## 13. Semantic Correctness Considerations

### Equivalence Analysis
- **Intended Business Logic:** Calculate the sum of `UnitPrice * SalesAmount` for all sales transactions where the purchased product is Red.
- **Risk in Bad Pattern (Row Duplication):** In `Total Sales Red BAD`, `CALCULATE([Total Sales])` executes context transition. If `FactInternetSales` had duplicate rows with identical values across all columns, context transition would filter to *all* matching rows in every iteration, producing **incorrect, inflated sales numbers**.
- **Equivalence in Standard Model:** Since `FactInternetSales` has unique transaction rows (or line numbers), `Total Sales Red BAD` produced the same scalar total as `Total Sales Red`, but at severe computational expense.
- **Filter Context Preservation:** Using `KEEPFILTERS(DimProduct[Color] = "Red")` ensures that if a user selects a specific color in a slicer (e.g., "Blue"), the measure returns `BLANK()` rather than overriding the slicer to force "Red". If standard overriding behavior is expected, `DimProduct[Color] = "Red"` without `KEEPFILTERS` is identical to the current `Total Sales Red`.

---

## 14. DAX Studio Validation Procedure

To validate the optimization empirically, perform the following test in DAX Studio:

### Test Query Script
```dax
/* =========================================================================
   DAX Studio Performance Validation Script
   Model: BigQueryAdventureWorksDW
   ========================================================================= */

// Query 1: Benchmark the Slow Pattern
EVALUATE
ROW (
    "Benchmark_Slow",
    SUMX (
        FactInternetSales,
        IF (
            RELATED ( DimProduct[Color] ) = "Red",
            CALCULATE ( [Total Sales] )
        )
    )
)

// Query 2: Benchmark the Optimized Pattern
EVALUATE
ROW (
    "Benchmark_Optimized",
    CALCULATE (
        [Total Sales],
        KEEPFILTERS ( DimProduct[Color] = "Red" )
    )
)
```

### Execution Settings
1. Connect DAX Studio to the running Power BI Desktop instance or workspace XMLA endpoint.
2. In the **Home** ribbon, toggle on **Server Timings**.
3. In the Server Timings options, check **xmSQL** and **Cache**.
4. Set execution mode to **Clear Cache and Run** (under the Run drop-down) before each test run to eliminate warm-cache distortion.
5. Execute each query independently **5 times** using "Clear Cache and Run". Discard run 1 (cold start compilation) and average runs 2 through 5.

### What to Compare
- **Total Duration (ms):** Expect reduction from ~100+ ms to < 10 ms.
- **Storage Engine Duration (ms):** Expect reduction from ~70+ ms to <= 2 ms.
- **Formula Engine Duration (ms):** Expect reduction from ~30+ ms to <= 5 ms.
- **Storage Engine Queries:** Expect reduction from 3 queries to 1 query.
- **Rows Materialized:** Expect reduction from 120,797 rows to 1 row.
- **Data Size (KB):** Expect reduction from ~11.5 MB to 1 KB.

### Criteria Indicating Diagnosis Was Valid or Invalid
- **Diagnosis Valid:** If SE queries drop to 1, rows drop to 1, and total duration drops below 15 ms.
- **Diagnosis Invalid:** If FE duration remains high or multiple SE scans returning 60,000+ rows persist (which would indicate that `[Total Sales]` itself contains an unpushable construct, which is refuted by `good query.png`).

---

## 15. Expected Result

| Metric | Baseline Slow Measure | Expected Optimized Result | Target Improvement |
| :--- | ---:| ---:| :--- |
| **Total Duration** | 107 ms | **< 10 ms** | **> 90% reduction** |
| **SE Duration** | 73 ms | **1 - 2 ms** | **> 97% reduction** |
| **FE Duration** | 34 ms | **2 - 5 ms** | **> 85% reduction** |
| **SE Queries** | 3 | **1** | **66% reduction** |
| **Rows Materialized** | 120,797 | **1** | **99.999% reduction** |
| **Memory Footprint** | ~11.5 MB | **1 KB** | **Eliminated datacache bloat** |

---

## 16. Remaining Uncertainties

1. **DirectQuery / Dual Mode Impact:**
   - The current model partition is set to `mode: import` with a Google BigQuery source. In import mode, VertiPaq resolves the single SE query entirely in memory. If this model were switched to DirectQuery, the bad pattern would be catastrophic, generating thousands of SQL queries or failing entirely due to query complexity limits.
2. **Base Measure Logic:**
   - `[Total Sales]` multiplies `UnitPrice * SalesAmount`. In AdventureWorksDW, `SalesAmount` typically already accounts for quantity and unit price (`SalesAmount = OrderQuantity * UnitPrice`). While this does not affect the relative performance diagnosis between the two measures, the business logic of `[Total Sales]` should be verified by the model author.

---

## 17. Final Recommendation

1. **Deprecate `Total Sales Red BAD`:** Remove the measure and update any report visuals referencing it (specifically Visual `f10e6d1a2ac4d2c4dc77` on page `8d760254e47a1d182ce3`) to use `Total Sales Red`.
2. **Adopt Set-Based Filter Modifiers:** Replace any similar `SUMX ( Fact, IF ( RELATED (...) = "...", CALCULATE (...) ) )` patterns across the semantic model with `CALCULATE ( [Measure], Dimension[Column] = "..." )`.
3. **Never Invoke Context Transition on Fact Tables:** Avoid calling measures or `CALCULATE` inside iterators traversing fact tables unless explicitly calculating at a specific grouped grain where unique keys are guaranteed and cardinality is minimal.