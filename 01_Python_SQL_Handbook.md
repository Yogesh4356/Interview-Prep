# Deloitte Interview Handbook 1 --- Python + SQL

## AI / ML Engineer II \| Interview + Production Focus

> **Goal:** Build interview-ready Python and SQL skills for a 3--6 year
> AI/ML Engineer role, with emphasis on clean code, reasoning about
> complexity, data manipulation, and production ML systems.

------------------------------------------------------------------------

# 1. How to use this handbook

For this Deloitte JD, Python and SQL are the foundation of the ML
lifecycle:

``` text
Business problem → Data extraction (SQL) → Data processing (Python/Pandas/PySpark)
→ Feature engineering → Model training → Evaluation → Deployment → Monitoring
```

For every topic, be able to answer: **what is it, why use it,
complexity/trade-off, and production connection.**

# 2. Python data structures

### List

Ordered, mutable collection. Indexing is O(1), membership is O(n),
append is amortized O(1).

### Tuple

Ordered, immutable collection. Useful for fixed records and hashable
composite keys when elements are hashable.

### Set

Unique values with average O(1) membership/insertion/deletion. Very
useful for duplicate detection.

### Dictionary

Key-value hash map with average O(1) lookup/insertion/deletion. Common
for feature maps, JSON payloads and counters.

``` python
feature_map = {"age": 31, "income": 85000, "tenure": 4}
```

**Production connection:** choose data structures based on access
pattern, memory, and readability rather than habit.

# 3. Mutability and copying

``` python
a = [1, 2, 3]
b = a
b.append(4)       # a also changes
```

Use `a.copy()` for a shallow copy and `copy.deepcopy()` for recursive
copying of nested objects.

**Production connection:** accidental mutation can corrupt shared
preprocessing state, configuration, caches, or request state.

# 4. Functions and interfaces

Prefer small, testable functions with explicit inputs/outputs.

``` python
def normalize_income(income: float) -> float:
    return income / 1000
```

Use type hints for important interfaces. They improve readability,
static analysis and IDE support, but are not runtime type enforcement by
themselves.

# 5. Iterators and generators

Generators produce values lazily:

``` python
def read_rows(rows):
    for row in rows:
        yield row
```

They are useful for large files, streams and batch pipelines because
they avoid materializing everything in memory.

# 6. Exception handling and logging

``` python
try:
    result = model.predict(features)
except ValueError:
    logger.exception("Invalid model input")
    raise
```

Avoid swallowing exceptions with `except Exception: pass`. Use
structured logging instead of `print()` in services.

Production logs should answer **what, when, where, request/job ID and
duration** without leaking secrets or sensitive document contents.

# 7. OOP essentials

Know class/object, constructor, encapsulation, inheritance, abstraction
and especially composition.

A production ML service might look like:

``` text
InferenceService
 ├── Preprocessor
 ├── Model
 └── Postprocessor
```

Composition often makes components easier to test and replace.

# 8. Decorators and context managers

Decorators wrap behavior such as timing, logging, authorization or
caching. Context managers guarantee resource cleanup:

``` python
with open("data.txt") as f:
    text = f.read()
```

# 9. Big-O and interview patterns

Know the complexity of common structures and patterns. A hashmap
solution to Two Sum is O(n) average time and O(n) space, versus O(n²)
brute force.

High-yield patterns:

-   hashmap/frequency map
-   two pointers
-   sliding window
-   stack
-   binary search
-   BFS/DFS basics
-   heap/top-K
-   prefix/suffix

Do not memorize 100 solutions. Learn the pattern and explain why it
reduces complexity.

# 10. NumPy and Pandas

Know NumPy arrays, shape, dtype, broadcasting, axes, vectorization and
matrix multiplication.

Know Pandas:

``` python
df.head()
df.info()
df.isna().sum()
df.groupby("segment")["amount"].mean()
df.merge(other, on="id")
```

**Production connection:** Pandas is excellent for moderate in-memory
datasets; it is not a distributed processing engine. Move to Spark when
the workload requires distributed processing.

# 11. SQL fundamentals

Typical ML data flow:

``` text
Operational DB / warehouse → SQL extraction → training dataset → feature engineering → model
```

Know SELECT, WHERE, GROUP BY, HAVING, ORDER BY, joins, CTEs, subqueries,
window functions, NULLs and aggregations.

Logical query processing is conceptually:

``` text
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

# 12. JOINs

-   **INNER JOIN:** only matches
-   **LEFT JOIN:** every left row plus matching right rows
-   **RIGHT JOIN:** every right row plus matching left rows
-   **FULL OUTER JOIN:** rows from both sides

The join choice is a data-correctness decision. Wrong joins can create
duplicate rows, inflated features or leakage.

# 13. Window functions

``` sql
SELECT customer_id, transaction_date, amount,
       ROW_NUMBER() OVER (
         PARTITION BY customer_id
         ORDER BY transaction_date DESC
       ) AS rn
FROM transactions;
```

Know `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`, and rolling
aggregates.

**Production ML connection:** window functions are excellent for
time-aware features, but always enforce an **as-of timestamp** so future
information cannot enter the feature.

# 14. CTEs

``` sql
WITH customer_totals AS (
    SELECT customer_id, SUM(amount) AS total
    FROM transactions
    GROUP BY customer_id
)
SELECT * FROM customer_totals WHERE total > 100000;
```

Use CTEs to make feature-generation logic auditable and readable.

# 15. NULL and duplicate handling

Use `IS NULL`, not `= NULL`. Use `COALESCE` deliberately.

Find duplicates:

``` sql
SELECT customer_id, COUNT(*)
FROM customers
GROUP BY customer_id
HAVING COUNT(*) > 1;
```

Data quality decisions here directly affect model behavior.

# 16. Top-N per group

``` sql
WITH ranked AS (
    SELECT customer_id, transaction_id, amount,
           ROW_NUMBER() OVER (
             PARTITION BY customer_id ORDER BY amount DESC
           ) AS rn
    FROM transactions
)
SELECT * FROM ranked WHERE rn <= 3;
```

# 17. SQL and data leakage

For a prediction at January 31, every feature must use information
available by January 31.

``` text
Prediction timestamp = Jan 31
Features = data available by Jan 31
Label = future outcome
```

**Excellent interview line:**

> "For ML feature SQL, I define an as-of date and ensure every feature
> only uses records available before that timestamp."

# 18. Python + SQL together

``` python
query = """
SELECT customer_id, age, income, total_spend
FROM customer_features
WHERE snapshot_date = :snapshot_date
"""

df = pd.read_sql(query, connection, params={"snapshot_date": snapshot_date})
```

Production improvements:

-   parameterized SQL
-   connection pooling
-   secrets management
-   schema validation
-   data-quality checks
-   retries for transient failures
-   select only required columns
-   batch large datasets

# 19. High-yield Python questions

### List vs tuple?

List is mutable; tuple is immutable.

### Set vs dictionary?

Set stores unique values; dictionary maps keys to values.

### Why generators?

Lazy evaluation reduces memory usage.

### Deep vs shallow copy?

Shallow copies the outer object while nested references can remain
shared; deep copy recursively copies nested objects.

### Why type hints?

Readability, tooling and static analysis.

# 20. High-yield SQL questions

### WHERE vs HAVING?

WHERE filters rows before aggregation; HAVING filters groups after
aggregation.

### ROW_NUMBER vs RANK?

ROW_NUMBER is unique/sequential; RANK assigns ties the same rank and
leaves gaps.

### Latest row per customer?

Use `ROW_NUMBER()` partitioned by customer and ordered by timestamp
descending.

### How do you prevent leakage?

Use explicit prediction/as-of timestamps and validate feature
availability.

# 21. Production checklist

-   [ ] Small, testable functions
-   [ ] Type hints for important interfaces
-   [ ] Intentional exception handling
-   [ ] Structured logs
-   [ ] No secrets in source
-   [ ] Parameterized SQL
-   [ ] Join/data-grain validation
-   [ ] NULL behavior understood
-   [ ] Leakage checks
-   [ ] Unit/integration tests
-   [ ] Realistic performance tests
-   [ ] Monitoring

# 22. 15 coding problems to practice

1.  Two Sum --- hashmap
2.  Contains Duplicate --- set
3.  Valid Anagram --- frequency map
4.  Longest Substring Without Repeating Characters --- sliding window
5.  Maximum Subarray --- Kadane
6.  Merge Intervals --- sorting/merge
7.  Valid Parentheses --- stack
8.  Binary Search --- invariant
9.  Top K Frequent Elements --- heap/bucket
10. Product of Array Except Self --- prefix/suffix
11. Best Time to Buy/Sell Stock --- running minimum
12. Reverse Linked List --- pointers
13. Linked List Cycle --- fast/slow pointers
14. Kth Largest --- heap/quickselect
15. Number of Islands --- BFS/DFS

# 23. Final mental model

For coding:

``` text
Correctness → Complexity → Edge cases → Readability → Tests → Production impact
```

For SQL:

``` text
Business definition → Data grain → Joins → Time/as-of logic → Aggregation → Leakage → Performance
```
