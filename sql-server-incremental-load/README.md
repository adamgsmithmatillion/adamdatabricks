# SQL Server Incremental Load

Batched incremental loading from SQL Server to Databricks with persistent watermark tracking. Prevents Azure Blob timeout issues when loading large datasets (30M+ records) by processing data in manageable chunks.

## Overview

This solution uses a parent-child pipeline pattern to load data incrementally:
- **Parent pipeline**: Initializes watermark from target table, loops up to 50 times
- **Child pipeline**: Loads one batch (default 1M rows), updates watermark immediately
- **Watermark persistence**: Queries target table for MAX timestamp on each run, ensuring true incremental loading across runs

**Source**: SQL Server (via JDBC)
**Destination**: Databricks (via Unity Catalog Volume)

## Files

- `sql-server-incremental-load.orch.yaml` - Parent orchestration pipeline (loop controller)
- `sql-server-load-single-batch.orch.yaml` - Child orchestration pipeline (single batch worker)

## How It Works

### First Run
1. Initialize Watermark queries empty target table → returns `1900-01-01` (COALESCE)
2. Loads all historical data in batches of 1M rows
3. Each batch appends to target table and updates watermark
4. Loop continues until all data loaded or 50 iterations reached

### Subsequent Runs
1. Initialize Watermark queries target table → returns actual MAX timestamp
2. Loads only records modified since last run
3. True incremental loading - no duplicate data

## Parent Pipeline Components

| Component | Purpose |
|-----------|----------|
| Initialize Watermark | Queries target table for MAX timestamp using COALESCE to handle empty table |
| Batch Loop Iterator | Executes child pipeline up to 50 times sequentially |
| Load Single Batch | Runs child pipeline to load one batch |

## Child Pipeline Components

| Component | Purpose |
|-----------|----------|
| Load Batch | Database Query component - loads TOP N rows from SQL Server with watermark filter |
| Get New Watermark | Queries Databricks for MAX timestamp from loaded batch |
| Update Watermark Variable | Updates `last_load_timestamp` for next iteration |

## Key Variables (PUBLIC)

| Variable | Default | Description |
|----------|---------|-------------|
| `last_load_timestamp` | `1900-01-01 00:00:00` | Watermark value - initialized from target table, updated by child |
| `batch_size` | `1000000` | Rows per batch - adjust based on timeout threshold |
| `sql_server_host` | `your-sql-server:1433` | SQL Server hostname and port |
| `sql_server_database` | `your_database` | SQL Server database name |
| `sql_server_username` | `your_username` | SQL Server username |
| `sql_server_password` | `your_password_secret_ref` | SQL Server password secret reference name |
| `sql_server_table` | `your_source_table` | SQL Server source table name |
| `timestamp_column` | `LastModifiedDate` | Column used for incremental filtering |
| `databricks_volume` | `your_databricks_volume` | Databricks volume name for staging |
| `target_table` | `staged_incremental_data` | Databricks target table name |
| `iteration_count` | `0` | Loop counter (managed by iterator) |

## Configuration Required

### Before First Run

**Update these PUBLIC variables in the parent pipeline:**

**SQL Server Connection:**
- `sql_server_host`: Your SQL Server hostname and port (e.g., `myserver.database.windows.net:1433`)
- `sql_server_database`: Database name
- `sql_server_username`: Username for authentication
- `sql_server_password`: Secret reference name (not actual password - must be created in Secrets)
- `sql_server_table`: Source table name to load from
- `timestamp_column`: Column name used for incremental filtering (e.g., `LastModifiedDate`, `UpdatedAt`)

**Databricks Configuration:**
- `databricks_volume`: Your Databricks volume name for staging data
- `target_table`: Target table name in Databricks

**Batch Settings:**
- `batch_size`: Rows per batch (default 1M is usually safe)

**Load Options (in child pipeline):**
- For **first run only**: Set `Recreate Target Table` to **On** in the child pipeline's Database Query component
- For **all subsequent runs**: Change to **Off** to enable append mode

### Expected Validation Error

⚠️ You'll see a validation error on "Initialize Watermark" before first run:
```
TABLE_OR_VIEW_NOT_FOUND: The table or view cannot be found
```

This is **normal and expected** - the target table doesn't exist until the first run creates it.

## Usage

1. Configure connection details and variables (see above)
2. Run parent pipeline: `sql-server-incremental-load.orch.yaml`
3. Pipeline will:
   - Initialize watermark from target table (or `1900-01-01` if empty)
   - Load data in batches
   - Update watermark after each batch
   - Stop when no more data or 50 iterations reached

### Scheduling

Schedule the parent pipeline to run on your desired frequency (hourly, daily, etc.). Each run automatically picks up from where the previous run left off.

## Performance Tuning

### Batch Size
- **Timeout occurring?** Reduce `batch_size` to 500K or 250K
- **Loading too slow?** Increase `batch_size` to 2-5M
- Default 1M rows ≈ 2-5 minutes per batch (safe for most cases)

### Max Iterations
- **Parent pipeline loops 50 times max** (can load up to 50M rows per run)
- To load more per run: Increase `endValue` in Loop Iterator
- Example: For 100M rows, set `endValue` to 100

## Important Notes

### Watermark Persistence
- Watermark is **not stored in a control table** - it's derived from target table MAX timestamp
- This approach is self-healing: if pipeline fails, next run resumes correctly
- If target table is truncated, next run will reload all data (starts from `1900-01-01`)

### Append Mode
- Child pipeline uses **append mode** (`Recreate Target Table: Off`) after first run
- Each batch appends to existing data
- Do NOT truncate target table between runs or incremental loading breaks

### Concurrency
- Loop Iterator runs **sequentially** (one batch at a time)
- This is intentional to avoid Azure Blob timeout issues
- Do not change to concurrent mode

### Data Ordering
- SQL query uses `ORDER BY ${timestamp_column}`
- Ensures oldest records loaded first
- Important for accurate watermark tracking

## Troubleshooting

**Problem**: Pipeline reloads all data every run
- **Cause**: Target table not preserving data between runs
- **Fix**: Ensure `Recreate Target Table` is set to **Off** after first run

**Problem**: Azure Blob timeout still occurring
- **Cause**: Batch size too large
- **Fix**: Reduce `batch_size` variable (try 500K or 250K)

**Problem**: Not all data loaded after 50 iterations
- **Cause**: More than 50M rows to load
- **Fix**: Increase `endValue` in Loop Iterator or run pipeline again (will continue from watermark)

## Architecture Pattern

This solution implements the **parent-child looping pattern** for batched data loading:

```
Parent: Initialize Watermark → Loop Iterator (50x)
                                    ↓
Child:  Load Batch → Get Watermark → Update Watermark
```

**Key advantage**: Watermark updates within each iteration, so each batch loads fresh data (no duplicates).

## Sharing This Solution

To make this solution available to other projects as a shared pipeline:

### How to Share

1. Click on the parent pipeline (`sql-server-incremental-load.orch.yaml`)
2. Click **Share**
3. Click **Publish**
4. Navigate to **Shared Jobs** to confirm it's published

**Note**: You only need to share the parent pipeline - the child pipeline (`sql-server-load-single-batch.orch.yaml`) will be included automatically when users add the shared pipeline to their project.

### For Users Adding This Shared Pipeline

1. Go to **Shared Jobs**
2. Find and add the `sql-server-incremental-load` pipeline to your project
3. Both parent and child pipelines will be available in your project
4. Configure connection details and variables (see Configuration Required section above)
5. Run the parent pipeline to start incremental loading
