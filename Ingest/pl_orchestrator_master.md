# 🔄 Orchestrator Master Pipeline — Complete Documentation
> **Pipeline Name:** `pl_orchestrator_master`
> **Child/Worker Pipeline:** `pl_executor_worker`
> **Linked Service:** `ecomwarehouse`
> **Dataset:** `warehouse`
> **Last Updated:** 2026-04-26

---

## 📌 Table of Contents
1. [Big Picture — What is this pipeline?](#1-big-picture--what-is-this-pipeline)
2. [Why Does This Pattern Exist?](#2-why-does-this-pattern-exist)
3. [Parameters — What Goes IN](#3-parameters--what-goes-in)
4. [Variables — Internal Memory](#4-variables--internal-memory)
5. [Database Schemas Used](#5-database-schemas-used)
6. [Complete Flow — Bird's Eye View](#6-complete-flow--birds-eye-view)
7. [Activity 1 — ORC Start Log](#7-activity-1--orc-start-log)
8. [Activity 2 — Get ORC Domain Name](#8-activity-2--get-orc-domain-name)
9. [Activity 3 — Look for Dependency Level](#9-activity-3--look-for-dependency-level)
10. [Activity 4 — Loop for Each Dependency Level](#10-activity-4--loop-for-each-dependency-level)
11. [Activity 4a — ORC If Condition](#11-activity-4a--orc-if-condition)
12. [Activity 4b — ForEach ORC Set Error Variable](#12-activity-4b--foreach-orc-set-error-variable)
13. [Activity 4c — ForEach Error Handling Script](#13-activity-4c--foreach-error-handling-script)
14. [Activity 4d — ForEach Fail Pipeline](#14-activity-4d--foreach-fail-pipeline)
15. [Activity 5 — ORC Pipeline End Log](#15-activity-5--orc-pipeline-end-log)
16. [Failure Path — Full Chain](#16-failure-path--full-chain)
17. [Status Codes Reference](#17-status-codes-reference)
18. [All Database Tables Reference](#18-all-database-tables-reference)
19. [Variable Flow Walkthrough](#19-variable-flow-walkthrough)
20. [How to Deploy in ADF](#20-how-to-deploy-in-adf)

---

## 1. Big Picture — What is this pipeline?

Think of this pipeline as a **Traffic Controller** at an airport.

```
Without this pipeline:
  Every data table loads whenever it wants
  Orders load before Customers exist → ERROR!
  Payments load before Orders exist  → ERROR!

With this pipeline:
  Traffic Controller decides WHO goes first
  Customers load first (Level 1)
  Products load next  (Level 2)
  Orders load after   (Level 3) ← waits for Level 1 + 2!
  Payments load last  (Level 4) ← waits for Level 3!
```

This pipeline is the **MASTER** that:
- Knows the order everything should run in
- Calls the **Worker pipeline** (`pl_executor_worker`) for each batch
- Tracks everything in log tables
- Handles errors gracefully

---

## 2. Why Does This Pattern Exist?

### The Problem
```
E-Commerce Database has these tables:
  customers    ← no dependency
  products     ← no dependency
  orders       ← NEEDS customers + products
  order_items  ← NEEDS orders
  payments     ← NEEDS orders
  reviews      ← NEEDS customers + products
```

If you load `orders` before `customers` → **Foreign Key Violation Error!**

### The Solution — Dependency Levels
```
Level 1 → customers, products      (load these FIRST, parallel)
Level 2 → orders, reviews          (load after Level 1 done)
Level 3 → order_items, payments    (load after Level 2 done)
```

### How the Config Table Controls This
```sql
-- retailflow.pipeline_config stores this:
domain_name    dependency_level    table_name      skip_failure
Orders         1                   customers       0
Orders         1                   products        0
Orders         2                   orders          0
Orders         2                   reviews         1   ← skip if fails
Orders         3                   order_items     0
Orders         3                   payments        0
```

- `skip_failure = 0` → if this level fails, **STOP** everything
- `skip_failure = 1` → if this level fails, **IGNORE** and continue

---

## 3. Parameters — What Goes IN

When someone triggers this pipeline, they pass these 3 values:

```json
{
    "entity_batch_id": "BATCH_20260426_001",
    "environment": "DEV",
    "domain_name": "Orders"
}
```

| Parameter | Type | Default | What it means |
|---|---|---|---|
| `entity_batch_id` | String | "0" | Unique ID for this run. Like a ticket number. Used to track this specific execution in logs |
| `environment` | String | "DEV" | Which environment: DEV / UAT / PROD |
| `domain_name` | String | (required) | Which group of tables to load. Example: "Orders", "Products", "Customers" |

### What is entity_batch_id?
```
Think of it like an order number at a restaurant.
Every time you trigger the pipeline → new batch ID.
All log entries for this run share the same batch_id.
You can search logs by batch_id to see everything that happened.

Example:
BATCH_20260426_001  ← Morning run
BATCH_20260426_002  ← Afternoon run
BATCH_20260426_003  ← Re-run after failure
```

---

## 4. Variables — Internal Memory

Variables are like **sticky notes** the pipeline uses while running.
They are reset every time the pipeline starts.

| Variable | Type | Default | Purpose |
|---|---|---|---|
| `FailPipeline` | String | "0" | Controls whether to continue or stop the loop |
| `IFErrorMessage` | Array | empty | Collects all error messages inside the IF condition |
| `ForEachErrorMessage` | String | empty | Final combined error from the ForEach loop |
| `ORCErrorMessage` | String | empty | Top level error message for the whole pipeline |

### FailPipeline — The Most Important Variable
```
"0" = Everything is fine, keep running  ✅
"1" = Something failed, STOP all loops  ❌

Start of pipeline:
  FailPipeline = "0"  (default)

Worker pipeline fails AND skip_failure = 0:
  FailPipeline = "1"  → next level will NOT execute!

Worker pipeline fails AND skip_failure = 1:
  FailPipeline stays "0"  → next level STILL executes!

Next ForEach iteration checks:
  FailPipeline = "0"  → IF TRUE  → run worker ✅
  FailPipeline = "1"  → IF FALSE → skip + fail ❌
```

### IFErrorMessage — The Error Collector
```
This is an ARRAY (list).
Every error gets APPENDED (added) to this list.

Example after 2 failures:
IFErrorMessage = [
  "Worker failed: Connection timeout at level 2",
  "Worker failed: Table not found at level 3"
]

At the end, join() combines them:
"Worker failed: Connection timeout at level 2, Worker failed: Table not found at level 3"
```

---

## 5. Database Schemas Used

This pipeline uses 2 schemas in `ecomdatawarehouse`:

### retailflow schema — Configuration (READ ONLY by pipeline)
```
retailflow.pipeline_domain         ← list of domains (Orders, Products etc.)
retailflow.pipeline_config         ← dependency levels per domain
retailflow.pipeline_workspace_params ← worker pipeline item_id
```

### audit schema — Logging (WRITTEN by pipeline)
```
audit.orc_batch_log        ← one row per batch per domain
audit.orc_execution_log    ← detailed execution tracking
audit.orc_error_log        ← error details when things fail
```

---

## 6. Complete Flow — Bird's Eye View

```
═══════════════════════════════════════════════════════
                    HAPPY PATH (Success)
═══════════════════════════════════════════════════════

TRIGGER
  ↓
[1] ORC start log          ← Script: log start, status='R'
  ↓ SUCCESS
[2] Get ORC Domain Name    ← Lookup: get domain_id
  ↓ SUCCESS
[3] Look for dependency    ← Lookup: get all pending levels
    level                    (ordered 1, 2, 3...)
  ↓ SUCCESS
[4] Loop for each          ← ForEach (SEQUENTIAL):
    dependency level          for each level...
  │
  ├─ Level 1:
  │    IF FailPipeline="0" → TRUE
  │    Execute pl_executor_worker (level 1)
  │    Worker succeeds ✅ → continue
  │
  ├─ Level 2:
  │    IF FailPipeline="0" → TRUE
  │    Execute pl_executor_worker (level 2)
  │    Worker succeeds ✅ → continue
  │
  └─ Level 3:
       IF FailPipeline="0" → TRUE
       Execute pl_executor_worker (level 3)
       Worker succeeds ✅ → done!
  ↓ ALL LEVELS DONE
[5] ORC Pipeline end log   ← Script: status='S', log end time
  ↓
PIPELINE COMPLETE ✅


═══════════════════════════════════════════════════════
               FAILURE PATH (Something breaks)
═══════════════════════════════════════════════════════

Any activity above fails
  ↓
[6] Set ORC Error Variable  ← SetVariable: find which activity failed
  ↓
[7] Generic Error Handling  ← Script: log error to audit.orc_error_log
    Script
  ↓
[8] Update ORC pipeline     ← Script: status='F' in logs (5 retries)
    fail log
  ↓
[9] Fail Pipeline           ← Fail: mark ADF run as FAILED ❌
```

---

## 7. Activity 1 — ORC Start Log

| Property | Value |
|---|---|
| Type | Script |
| Linked Service | `ecomwarehouse` |
| Runs after | Nothing (first activity) |
| On failure | Triggers failure path |

### What this activity does — step by step:

**Step 1: Find the domain_id**
```sql
DECLARE @domain_id INT;

SELECT DISTINCT @domain_id = domain_id
FROM retailflow.pipeline_domain AS d
WHERE d.is_active = 1
  AND d.domain_name = @domain_name;
-- @domain_name comes from pipeline parameter
-- Example: 'Orders' → domain_id = 3
```

**Step 2: Check if batch already started (avoid duplicates)**
```sql
IF NOT EXISTS (
    SELECT 1
    FROM [audit].[orc_batch_log]
    WHERE batch_id = @entity_batch_id
      AND domain_name = @domain_name
)
-- This check is important for RE-RUNS!
-- If pipeline crashed and you re-run with same batch_id
-- → It won't create a duplicate batch log entry
```

**Step 3: Insert batch log entry**
```sql
INSERT INTO [audit].[orc_batch_log] (
    batch_id,
    domain_name,
    status,           -- 'R' = Running
    start_date_time,  -- right now
    end_date_time     -- NULL (not done yet)
)
VALUES (
    @entity_batch_id,
    @domain_name,
    'R',
    SYSDATETIME(),
    NULL
);
```

**Step 4: Insert execution log entry**
```sql
INSERT INTO audit.orc_execution_log (
    workspace_id,
    pipeline_run_id,      -- ADF's unique run ID
    pipeline_id,
    pipeline_name,        -- 'pl_orchestrator_master'
    pipeline_trigger_time,
    batch_id,
    domain_id,
    status,               -- 'R' = Running
    inserted_on,
    pipeline_type
)
```

**Step 5: Return domain_id**
```sql
SELECT @domain_id;
-- This output is used by the next activity!
```

### Why the duplicate check matters:
```
Scenario: Pipeline crashes at Level 3, you re-run it.
  Same batch_id used → IF NOT EXISTS → skips insert
  No duplicate batch log → clean audit trail ✅

Without this check:
  Re-run → duplicate batch log entry → messy logs ❌
```

---

## 8. Activity 2 — Get ORC Domain Name

| Property | Value |
|---|---|
| Type | Lookup |
| Dataset | `warehouse` |
| Source type | AzureSqlSource |
| Runs after | ORC start log (SUCCESS) |
| firstRowOnly | TRUE (default) |
| Retry | 2 times, 30 sec interval |

### What this activity does:
```
Fetches domain_id and domain_name from config table
for the domain_name that was passed as a parameter
```

### Query:
```sql
SELECT domain_id, domain_name
FROM retailflow.pipeline_domain
WHERE is_active = 1
  AND domain_name = '@{pipeline().parameters.domain_name}'
-- Example result:
-- domain_id = 3, domain_name = 'Orders'
```

### How the output is used later:
```
activity('Get ORC Domain Name').output.firstRow.domain_id
→ Used in: Look for dependency level query
→ Used in: Execute Worker pipeline parameters
→ Used in: Set ORC Error Variable check
```

### Why is this a separate activity from Start Log?
```
Start Log returns domain_id via SELECT at the end.
But Lookup output is easier to reference in ADF expressions.
Lookup gives: activity('Get ORC Domain Name').output.firstRow.domain_id
Script output: harder to reference in subsequent activities.
So both are used — Start Log for writing, Lookup for reading.
```

---

## 9. Activity 3 — Look for Dependency Level

| Property | Value |
|---|---|
| Type | Lookup |
| Dataset | `warehouse` |
| Source type | AzureSqlSource |
| Runs after | Get ORC Domain Name (SUCCESS) |
| firstRowOnly | **FALSE** — returns ALL rows! |
| Retry | 2 times, 30 sec interval |

### What this activity does:
```
Fetches ALL pending dependency levels for this domain
that have NOT yet been successfully completed in this batch.
Results are ordered ASC (1, 2, 3...) so loop runs in order.
```

### Full Query with explanation:
```sql
SELECT DISTINCT
    pc.dependency_level,    -- the level number (1, 2, 3...)
    pc.skip_failure,        -- 0=stop on fail, 1=ignore fail
    wp.item_id              -- ADF item ID of worker pipeline

FROM retailflow.pipeline_config AS pc

INNER JOIN retailflow.pipeline_domain AS pd
    ON pd.domain_id = pc.domain_id
    -- join to get domain info

LEFT JOIN (
    -- get the worker pipeline's ADF item_id
    -- this is how ADF knows WHICH pipeline to call
    SELECT item_id
    FROM retailflow.pipeline_workspace_params
    WHERE item_type = 'DataPipeline'
      AND item_name = 'pl_executor_worker'
) wp ON 1 = 1
-- LEFT JOIN ON 1=1 means: attach item_id to EVERY row

WHERE pc.is_active = 1
  AND pd.is_active = 1
  AND pd.domain_id = '@{activity('Get ORC Domain Name').output.firstRow.domain_id}'

  AND pc.dependency_level NOT IN (
      -- THE KEY PART: skip already completed levels!
      SELECT DISTINCT dependency_level
      FROM audit.orc_execution_log
      WHERE domain_id = '@{activity('Get ORC Domain Name').output.firstRow.domain_id}'
        AND status = 'S'                              -- only SUCCESSFUL ones
        AND batch_id = '@{pipeline().parameters.entity_batch_id}'
        AND dependency_level IS NOT NULL
  )

ORDER BY pc.dependency_level;   -- MUST be ordered!
```

### Example output:
```json
{
  "count": 3,
  "value": [
    {"dependency_level": 1, "skip_failure": 0, "item_id": "abc-123"},
    {"dependency_level": 2, "skip_failure": 1, "item_id": "abc-123"},
    {"dependency_level": 3, "skip_failure": 0, "item_id": "abc-123"}
  ]
}
```

### The RESUMABLE pipeline concept:
```
First run — all 3 levels pending:
  Query returns: Level 1, Level 2, Level 3

Pipeline crashes at Level 3.
You fix the issue and RE-RUN with same batch_id.

Second run — Level 1 and 2 already done (status='S'):
  NOT IN clause removes Level 1 and Level 2
  Query returns: ONLY Level 3

Pipeline resumes from where it left off! ✅
No need to re-process already done data!
```

---

## 10. Activity 4 — Loop for Each Dependency Level

| Property | Value |
|---|---|
| Type | ForEach |
| isSequential | **TRUE** — one at a time! |
| Items | `@activity('Look for dependency level').output.value` |
| Runs after | Look for dependency level (SUCCESS) |

### Why Sequential = TRUE is critical:
```
isSequential = TRUE:
  Level 1 → WAIT until done → Level 2 → WAIT → Level 3
  Safe! Level 2 always has Level 1's data ready ✅

isSequential = FALSE (parallel):
  Level 1, 2, 3 all start at same time!
  Level 3 tries to load Orders before Customers exist!
  FOREIGN KEY VIOLATION ERROR ❌
```

### What happens in each iteration:
```
Iteration 1: item() = {dependency_level: 1, skip_failure: 0, item_id: "abc"}
  → Checks FailPipeline → "0" → TRUE → runs worker for Level 1
  → Worker succeeds → moves to iteration 2

Iteration 2: item() = {dependency_level: 2, skip_failure: 1, item_id: "abc"}
  → Checks FailPipeline → "0" → TRUE → runs worker for Level 2
  → Worker FAILS → skip_failure=1 → FailPipeline stays "0"
  → Appends error to IFErrorMessage
  → Fails this iteration
  → ForEach sets error variable → logs error → fails ForEach

If skip_failure was 0:
  → FailPipeline = "1"
  → Iteration 3: Checks FailPipeline → "1" → FALSE → skips!
```

### How item() works:
```
item() gives you the current row from the Lookup output.

Inside the ForEach, you access:
  item().dependency_level  → 1, 2, or 3
  item().skip_failure      → 0 or 1
  item().item_id           → ADF pipeline item ID
```

---

## 11. Activity 4a — ORC If Condition

| Property | Value |
|---|---|
| Type | IfCondition |
| Runs after | Nothing (first in ForEach) |
| Expression | `@equals(variables('FailPipeline'),'0')` |

### The condition explained:
```
Check: Is FailPipeline equal to "0"?

YES (TRUE path)  → Previous level succeeded → Run worker pipeline
NO  (FALSE path) → Previous level FAILED   → Skip this level, fail out
```

---

### TRUE PATH — Execute Worker Pipeline

| Property | Value |
|---|---|
| Type | ExecutePipeline |
| Pipeline called | `pl_executor_worker` |
| waitOnCompletion | TRUE — waits for worker to finish |

#### Parameters passed to worker:
```json
{
    "entity_batch_id": "@pipeline().parameters.entity_batch_id",
    "dependency_level": "@item().dependency_level",
    "domain_id": "@activity('Get ORC Domain Name').output.firstRow.domain_id",
    "item_id": "@item().item_id"
}
```

#### What each parameter tells the worker:
```
entity_batch_id  → which batch this belongs to (for logging)
dependency_level → which level to process (1, 2, or 3)
domain_id        → which domain (e.g. domain_id = 3 for Orders)
item_id          → ADF pipeline ID to execute
```

#### After worker runs:

**If worker SUCCEEDS:**
```
Nothing extra happens.
ForEach moves to next iteration automatically. ✅
```

**If worker FAILS:**
```
→ True Set variable for FailPipeline runs
   Expression: @if(equals(item().skip_failure,1),'0','1')
   
   skip_failure = 1 → FailPipeline = '0' (ignore, keep going)
   skip_failure = 0 → FailPipeline = '1' (stop everything!)

→ Append Error variable true for ORC invoke worker runs
   Appends worker error message to IFErrorMessage array:
   @if(
     equals(activity('Execute Worker pipeline').status,'Failed'),
     activity('Execute Worker pipeline').error.message,
     'No Activity Failed'
   )

→ True Fail ORC Invoke Worker runs
   Officially fails this ForEach iteration
   Message: 'Loop failed for dependency Level: 2 | <error>'
   ErrorCode: 'DEP_LEVEL_FAIL_2'
```

---

### FALSE PATH — Previous Level Failed, Skip This One

This runs when `FailPipeline = "1"` (a previous level failed and skip_failure was 0).

#### Step 1: False set variable for fail pipeline
```
Sets FailPipeline = "1"
(makes sure it stays 1, not changed accidentally)
```

#### Step 2: ORC append error skip message
```
Appends to IFErrorMessage array:
'Loop skipped for dependency level: 3 due to previous failure'
```

#### Step 3: Fail IF false ORC skip level
```
Fails this iteration with:
Message:   'Dependency level 3 skipped. Errors: <all errors joined>'
ErrorCode: 'DEP_LEVEL_SKIP_3'
```

---

## 12. Activity 4b — ForEach ORC Set Error Variable

| Property | Value |
|---|---|
| Type | SetVariable |
| Variable set | `ForEachErrorMessage` |
| Runs after | ORC If condition (FAILED) |

### Expression:
```
@if(
    equals(activity('ORC If condition for dependency execution').status, 'Failed'),
    concat(
        join(variables('IFErrorMessage'), ', '),
        ' | ',
        activity('ORC If condition for dependency execution').error.message
    ),
    'No Activity Failed'
)
```

### What this does:
```
Takes all individual error messages from IFErrorMessage array
Joins them with ', ' separator
Adds the IF condition's own error message at the end
Stores the complete error in ForEachErrorMessage

Example result:
"Worker failed: timeout, Worker failed: table missing | IF condition failed"
```

---

## 13. Activity 4c — ForEach Error Handling Script

| Property | Value |
|---|---|
| Type | Script |
| Linked Service | `ecomwarehouse` |
| Runs after | ForEach ORC set Error Variable (COMPLETED) |

### What this does:
```
Logs the error details into the database
So you can query the error table to see what went wrong
```

### SQL:
```sql
INSERT INTO audit.orc_error_log (
    workspace_id,
    pipeline_run_id,
    pipeline_id,
    pipeline_name,
    pipeline_trigger_time,
    batch_id,
    inserted_on,
    pipeline_type,
    error_description,  -- the full error message
    dependency_level    -- WHICH level failed (very useful!)
)
VALUES (
    @workspace_id,
    @pipeline_run_id,
    @pipeline_id,
    @pipeline_name,
    @pipeline_trigger_time,
    @entity_batch_id,
    GETDATE(),
    @pipeline_type,
    @error_description,
    @dependency_level   -- e.g. 2 (level 2 failed)
)
```

### Why dependency_level in error log is important:
```
You can query:
SELECT * FROM audit.orc_error_log
WHERE batch_id = 'BATCH_20260426_001'

Result shows:
batch_id              dependency_level  error_description
BATCH_20260426_001    2                 Worker failed: timeout on orders table

→ Instantly you know: Level 2 failed, orders table had timeout!
```

---

## 14. Activity 4d — ForEach Fail Pipeline

| Property | Value |
|---|---|
| Type | Fail |
| errorCode | `ORC_FOREACH_FAIL` |
| Message | `@variables('ForEachErrorMessage')` |
| Runs after | ForEach Error Handling Script (COMPLETED) |

### What this does:
```
Officially marks the ForEach loop as FAILED in ADF.
This failure bubbles up to the outer pipeline.
The outer pipeline sees ForEach failed → triggers failure path.
```

---

## 15. Activity 5 — ORC Pipeline End Log

| Property | Value |
|---|---|
| Type | Script |
| Linked Service | `ecomwarehouse` |
| Runs after | Loop for each dependency level (SUCCESS ONLY) |
| Retry | 2 times, 30 sec interval |

### What this does:
```
All levels completed successfully!
Now update the log tables to show SUCCESS.
Has built-in retry logic (5 attempts) in case of DB deadlock.
```

### Full SQL with retry:
```sql
DECLARE @DomainName NVARCHAR(100);
DECLARE @MaxRetries INT = 5;
DECLARE @Attempt INT = 1;

-- First get the domain name
SELECT @DomainName = domain_name
FROM retailflow.pipeline_domain
WHERE is_active = 1
  AND domain_id = @domain_id;

-- Try to update up to 5 times
WHILE @Attempt <= @MaxRetries
BEGIN
    BEGIN TRY
        -- Mark execution log as SUCCESS
        UPDATE audit.orc_execution_log
        SET status = 'S',           -- S = Success
            completed_on = GETDATE()
        WHERE pipeline_run_id = @pipeline_run_id
          AND domain_id = @domain_id;

        -- Mark batch log as SUCCESS
        UPDATE [audit].[orc_batch_log]
        SET status = 'S',
            end_date_time = SYSDATETIME()
        WHERE batch_id = @entity_batch_id
          AND domain_name = @DomainName;

        BREAK;  -- EXIT loop, we succeeded!

    END TRY
    BEGIN CATCH
        -- Update failed (maybe deadlock), wait and retry
        PRINT CONCAT('Attempt ', @Attempt, ' failed: ', ERROR_MESSAGE());
        WAITFOR DELAY '00:00:02';  -- wait 2 seconds
    END CATCH

    SET @Attempt += 1;
END

-- If all 5 attempts failed, throw error
IF @Attempt > @MaxRetries
    THROW 51000, 'End log update failed after maximum retries.', 1;
```

### Why the retry logic?
```
In enterprise databases, UPDATE statements can hit DEADLOCKS.
A deadlock happens when two queries block each other.
SQL Server kills one of them → that query fails.

Without retry: pipeline fails just because of a deadlock!
With retry: wait 2 seconds → try again → usually succeeds ✅

5 retries × 2 second wait = 10 seconds max before giving up.
```

---

## 16. Failure Path — Full Chain

This chain runs when ANY main activity fails.

### What triggers the failure path?

The `Set ORC Error Variable` activity has TWO dependsOn conditions:
```
dependsOn:
  1. ORC Pipeline end log   → FAILED or SKIPPED
  2. Loop for each dep level → FAILED or SKIPPED
```

So it triggers when:
- The ForEach loop fails (any level failed)
- The End Log script fails
- Either gets skipped due to earlier failure

---

### Step 1: Set ORC Error Variable

| Property | Value |
|---|---|
| Type | SetVariable |
| Variable | `ORCErrorMessage` |

#### Expression — checks each activity in order:
```
@if(
    equals(activity('ORC start log').status, 'Failed'),
    activity('ORC start log').error.message,          ← Start Log failed?

    if(
        equals(activity('Get ORC Domain Name').status, 'Failed'),
        activity('Get ORC Domain Name').error.message, ← Domain lookup failed?

        if(
            equals(activity('Look for dependency level').status, 'Failed'),
            activity('Look for dependency level').error.message, ← Dep level lookup failed?

            if(
                equals(activity('Loop for each dependency level').status, 'Failed'),
                activity('Loop for each dependency level').error.message, ← ForEach failed?

                if(
                    equals(activity('ORC Pipeline end log').status, 'Failed'),
                    activity('ORC Pipeline end log').error.message, ← End log failed?

                    'No Activity Failed'   ← fallback (shouldn't happen)
                )
            )
        )
    )
)
```

#### How this works — like nested if-else:
```
Check ORC start log first.
  Failed? → use its error message. DONE.
  Not failed? → check next one.

Check Get ORC Domain Name.
  Failed? → use its error message. DONE.
  Not failed? → check next one.

...and so on until we find the failed activity.

Result stored in ORCErrorMessage variable.
Used later by Fail Pipeline activity.
```

---

### Step 2: Generic Error Handling Script

| Property | Value |
|---|---|
| Type | Script |
| Linked Service | `ecomwarehouse` |
| Runs after | Set ORC Error Variable (COMPLETED) |

```sql
-- Logs the top-level pipeline error
INSERT INTO audit.orc_error_log (
    workspace_id,
    pipeline_run_id,
    pipeline_id,
    pipeline_name,
    pipeline_trigger_time,
    batch_id,
    inserted_on,
    pipeline_type,
    error_description,  -- @ORCErrorMessage (which activity failed + why)
    dependency_level    -- NULL (top level error, not level specific)
)
```

---

### Step 3: Update ORC Pipeline Fail Log

| Property | Value |
|---|---|
| Type | Script |
| Linked Service | `ecomwarehouse` |
| Runs after | Generic Error Handling Script (COMPLETED) |

```sql
-- With 5 retry attempts (same pattern as end log)
-- Updates BOTH log tables to status = 'F' (Failed)

UPDATE audit.orc_execution_log
SET status = 'F',
    completed_on = GETDATE()
WHERE pipeline_run_id = @pipeline_run_id
  AND domain_id = @domain_id;

UPDATE audit.orc_batch_log
SET status = 'F',
    end_date_time = SYSDATETIME()
WHERE batch_id = @entity_batch_id
  AND domain_name = @domain_name;
```

---

### Step 4: Fail Pipeline

| Property | Value |
|---|---|
| Type | Fail |
| errorCode | `ORC_MASTER_FAIL` |
| Message | `@variables('ORCErrorMessage')` |
| Runs after | Update ORC pipeline fail log (COMPLETED) |

```
This is the final nail.
Officially marks the ADF pipeline run as FAILED.
The error message shown in ADF monitor = ORCErrorMessage.
Anyone checking ADF will see exactly which activity failed and why.
```

---

## 17. Status Codes Reference

| Code | Meaning | Set where |
|---|---|---|
| `R` | Running | ORC Start Log (on start) |
| `S` | Success | ORC Pipeline End Log (on success) |
| `F` | Failed | Update ORC Pipeline Fail Log (on failure) |

---

## 18. All Database Tables Reference

### retailflow.pipeline_domain
```sql
-- Stores all domains (groups of related tables)
domain_id      INT           -- unique ID
domain_name    VARCHAR(100)  -- 'Orders', 'Products', 'Customers'
is_active      BIT           -- 1=active, 0=disabled
```

### retailflow.pipeline_config
```sql
-- Stores which tables belong to which domain + level
config_id          INT
domain_id          INT       -- FK to pipeline_domain
dependency_level   INT       -- 1, 2, 3...
skip_failure       BIT       -- 0=stop on fail, 1=ignore fail
is_active          BIT
```

### retailflow.pipeline_workspace_params
```sql
-- Stores ADF pipeline item IDs
item_id        VARCHAR(200)  -- ADF pipeline item ID
item_type      VARCHAR(50)   -- 'DataPipeline'
item_name      VARCHAR(100)  -- 'pl_executor_worker'
```

### audit.orc_batch_log
```sql
-- One row per batch per domain
batch_id         VARCHAR(100)  -- 'BATCH_20260426_001'
domain_name      VARCHAR(100)  -- 'Orders'
status           CHAR(1)       -- 'R', 'S', 'F'
start_date_time  DATETIME2     -- when batch started
end_date_time    DATETIME2     -- when batch ended (NULL if running)
```

### audit.orc_execution_log
```sql
-- Detailed tracking per pipeline run
workspace_id           VARCHAR(200)
pipeline_run_id        VARCHAR(200)  -- ADF unique run ID
pipeline_id            VARCHAR(200)
pipeline_name          VARCHAR(200)
pipeline_trigger_time  DATETIME2
batch_id               VARCHAR(100)
domain_id              INT
status                 CHAR(1)       -- 'R', 'S', 'F'
inserted_on            DATETIME2
completed_on           DATETIME2     -- NULL until done
pipeline_type          VARCHAR(50)
dependency_level       INT           -- which level (for resumability check)
```

### audit.orc_error_log
```sql
-- Error details when something fails
workspace_id           VARCHAR(200)
pipeline_run_id        VARCHAR(200)
pipeline_id            VARCHAR(200)
pipeline_name          VARCHAR(200)
pipeline_trigger_time  DATETIME2
batch_id               VARCHAR(100)
inserted_on            DATETIME2
pipeline_type          VARCHAR(50)
error_description      VARCHAR(MAX)  -- full error message
dependency_level       INT           -- which level failed (NULL for top-level)
```

---

## 19. Variable Flow Walkthrough

### Scenario: Level 2 fails, skip_failure = 0

```
PIPELINE START
  FailPipeline    = "0"   (default)
  IFErrorMessage  = []    (empty array)
  ForEachErrorMessage = ""
  ORCErrorMessage     = ""

══ ForEach Iteration 1 (Level 1) ══
  IF check: FailPipeline = "0" → TRUE
  Execute Worker → SUCCESS ✅
  FailPipeline    = "0"   (unchanged)
  IFErrorMessage  = []    (unchanged)

══ ForEach Iteration 2 (Level 2) ══
  IF check: FailPipeline = "0" → TRUE
  Execute Worker → FAILED ❌
  
  True Set variable for FailPipeline:
    skip_failure = 0 → FailPipeline = "1"
  
  Append Error variable:
    IFErrorMessage = ["Worker failed: DB connection lost"]
  
  True Fail ORC Invoke Worker:
    Fails with: "Loop failed for dependency Level: 2 | Worker failed: DB connection lost"
    errorCode: "DEP_LEVEL_FAIL_2"

  IF condition now FAILED → ForEach sees failure

  ForEach ORC set Error Variable:
    ForEachErrorMessage = "Worker failed: DB connection lost | IF condition failed at level 2"

  ForEach Error Handling Script:
    INSERT into audit.orc_error_log
    error_description = "Worker failed: DB..."
    dependency_level  = 2

  ForEach Fail pipeline:
    Fails with ForEachErrorMessage
    errorCode: "ORC_FOREACH_FAIL"

══ ForEach Iteration 3 (Level 3) ══
  IF check: FailPipeline = "1" → FALSE
  
  False set variable for fail pipeline:
    FailPipeline = "1" (stays 1)
  
  ORC append error skip message:
    IFErrorMessage = [
      "Worker failed: DB connection lost",
      "Loop skipped for dependency level: 3 due to previous failure"
    ]
  
  Fail IF false ORC skip level:
    Fails with: "Dependency level 3 skipped. Errors: ..."
    errorCode: "DEP_LEVEL_SKIP_3"

FOREACH NOW FULLY FAILED
  ↓
Set ORC Error Variable:
  Checks all activities → ForEach failed
  ORCErrorMessage = "Loop for each dependency level error message"

Generic Error Handling Script:
  INSERT into audit.orc_error_log
  dependency_level = NULL (top level)

Update ORC pipeline fail log:
  orc_execution_log → status = 'F'
  orc_batch_log → status = 'F'

Fail Pipeline:
  ADF run marked FAILED ❌
  Message = ORCErrorMessage
```

---

## 20. How to Deploy in ADF

### Step 1: Create Linked Service
```
ADF Studio
→ Manage tab
→ Linked Services
→ + New
→ Azure SQL Database
→ Name: ecomwarehouse
→ Server: your-server.database.windows.net
→ Database: ecomdatawarehouse
→ Auth: SQL Authentication
→ Username + Password
→ Test Connection → Create
```

### Step 2: Create Dataset
```
ADF Studio
→ Author tab
→ Datasets
→ + New Dataset
→ Azure SQL Database
→ Name: warehouse
→ Linked Service: ecomwarehouse
→ Table: (leave blank, queries used directly)
→ OK
```

### Step 3: Import Pipeline JSON
```
ADF Studio
→ Author tab
→ Pipelines
→ Click ··· (three dots)
→ Import from JSON
→ Upload: pl_orchestrator_master.json
→ Publish All
```

### Step 4: Create Config Tables
```sql
-- Run in SSMS on ecomdatawarehouse:
-- 1. Create retailflow schema + tables
-- 2. Create audit schema + tables
-- 3. Insert domain config data
-- See /database/ folder for full SQL scripts
```

### Step 5: Test Run
```
ADF Studio
→ pl_orchestrator_master
→ Debug
→ Parameters:
    entity_batch_id: BATCH_TEST_001
    environment:     DEV
    domain_name:     Orders
→ OK
→ Watch Monitor tab for results
```

### Step 6: Check Results in SSMS
```sql
-- Check batch log
SELECT * FROM audit.orc_batch_log
WHERE batch_id = 'BATCH_TEST_001';

-- Check execution log
SELECT * FROM audit.orc_execution_log
WHERE batch_id = 'BATCH_TEST_001';

-- Check errors (if any)
SELECT * FROM audit.orc_error_log
WHERE batch_id = 'BATCH_TEST_001';
```

---

## Quick Cheat Sheet

```
Pipeline:   pl_orchestrator_master
Worker:     pl_executor_worker
Linked Svc: ecomwarehouse
Dataset:    warehouse

Parameters:
  entity_batch_id → unique run ID (like ticket number)
  environment     → DEV / UAT / PROD
  domain_name     → which group to load (e.g. Orders)

Variables:
  FailPipeline        → "0"=go, "1"=stop
  IFErrorMessage      → array of errors from IF path
  ForEachErrorMessage → combined error from ForEach
  ORCErrorMessage     → top level error message

Status codes:
  R = Running
  S = Success
  F = Failed

Dependency levels:
  Level 1 → no dependency (load first)
  Level 2 → depends on level 1
  Level 3 → depends on level 2
  ...sequential, never parallel!

skip_failure:
  0 = if this level fails → STOP all next levels
  1 = if this level fails → IGNORE and continue next level

Retry logic:
  End log   → 5 retries, 2 sec wait
  Fail log  → 5 retries, 2 sec wait
  Lookups   → 2 ADF retries, 30 sec interval
```

---
