# DAX Performance Mindmap: `Total Sales Red` vs `Total Sales Red BAD`

This mindmap summarizes the architectural diagnosis, root causes, engine mechanics, and the validated solution for the `BigQueryAdventureWorksDW` model.

---

## 1. Mermaid Visual Mindmap

```mermaid
mindmap
  root((DAX Performance Diagnosis))
    Root Causes
      Fact Table Iteration
        SUMX over 60,398 rows
        Unnecessary row context
      Row-Level Context Transition
        CALCULATE inside iterator
        Converts 22 columns into filters
        Evaluated 60,398 times
      Procedural Row Filtering
        RELATED DimProduct Color inside IF
        Single-threaded FE evaluation
    Diagnostic Evidence
      Server Timings
        107 ms vs 4 ms Total Duration (26.8x slower)
        73 ms vs 1 ms SE Duration (73x slower)
        34 ms vs 3 ms FE Duration (11x slower)
        3 SE Queries vs 1 SE Query
      Materialization Metrics
        120,797 rows materialized vs 1 row
        11.5 MB uncompressed datacache vs 1 KB
      VPAX Structure
        FactInternetSales: 60,398 rows
        DimProduct: 606 rows (10 colors)
    Internal Engine Mechanics
      Storage Engine VertiPaq
        Cannot aggregate to scalar
        Pushed down to row grain
        2 full scans of 60,398 rows
      Formula Engine FE
        Overwhelmed by 11.5 MB datacache
        Row-by-row conditional evaluation
        Context transition execution
    Optimized Solution
      Code Pattern
        CALCULATE Base Measure with Filter
        KEEPFILTERS DimProduct Color = Red
      Engine Benefits
        Zero outer row context
        Zero context transition
        Single multi-threaded SE scan
        1 row (1 KB) returned in 1 ms
      Validation Protocol
        DAX Studio Clear Cache and Run
        5 warm/cold benchmark runs
        Verify scalar equivalence
```

---

## 2. Hierarchical Mindmap Outline

### 1. Root Causes (The Anti-Pattern)
- **Fact Table Iteration:**
  - `SUMX` loops across all 60,398 rows of `FactInternetSales`.
  - Violates the principle of filtering upstream on dimensions.
- **Row-Level Context Transition:**
  - `CALCULATE([Total Sales])` is evaluated in the row context of each fact row.
  - Automatically transforms all 22 columns of `FactInternetSales` into filter context.
  - Executed 60,398 consecutive times.
- **Procedural Row Filtering:**
  - Evaluates `IF ( RELATED(DimProduct[Color]) = "Red", ... )` row-by-row.
  - Forces the Formula Engine to inspect every row instead of letting the Storage Engine filter columnar vectors.

### 2. Diagnostic Evidence
- **Server Timings (`good query.png` vs `Bad query.png`):**
  - Total Duration: 4 ms vs 107 ms (**26.8x slower**).
  - Storage Engine Duration: 1 ms vs 73 ms (**73x slower**).
  - Formula Engine Duration: 3 ms vs 34 ms (**11.3x slower**).
  - Storage Engine Queries: 1 query vs 3 queries.
- **Materialization:**
  - Fast Measure: **1 row (1 KB)**.
  - Slow Measure: **120,797 rows (11,563 KB)** across two scans of 60,398 rows.
- **VPAX Structural Facts:**
  - `FactInternetSales` has exactly 60,398 rows (matching the scan row counts).
  - `DimProduct[Color]` has only 10 unique colors across 606 products.

### 3. Engine Execution Mechanics
- **Storage Engine (SE / VertiPaq):**
  - Unable to compute a simple `SUM` because the outer iterator demands row-level outputs.
  - Forces materialization of the entire fact table into datacache memory.
- **Formula Engine (FE):**
  - Must unpack 11.5 MB of uncompressed memory.
  - Evaluates 60,398 row transitions serially on a single thread.

### 4. Optimized Solution & Best Practices
- **Optimized DAX:**
  ```dax
  measure 'Total Sales Red' = 
  CALCULATE (
      [Total Sales],
      KEEPFILTERS ( DimProduct[Color] = "Red" )
  )
  ```
- **Execution Path:**
  - Injects `DimProduct[Color] = "Red"` into filter context.
  - Propagates automatically across the active 1-to-many relationship.
  - Storage Engine executes a single multi-threaded scan and returns 1 row (1 KB) in 1 ms.