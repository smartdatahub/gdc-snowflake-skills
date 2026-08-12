---
name: geo-data-connector
description: SQL procedures, ingested data schema, and geospatial query patterns for the Geo Data Connector (GDC) CoCo Skill — Snowflake Native App. Use when questions mention GDC, Geo Data Connector, or GDC_CONFIG, GDC_TASK, GDC_CONNECTION, GDC_DISCOVER procedures.
tools:
  - snowflake_execute
  - snowflake_sql_execute
metadata:
  author: smartdatahub
  version: "1.5.0"
---

# Geo Data Connector — CoCo Skill

Geo Data Connector provides four compound SQL procedures for managing geospatial data ingestion programmatically. These procedures are available with the **paid edition** only.

## Editions

Geo Data Connector is installed from the Snowflake Marketplace in two editions. This skill and the SQL procedures below require the **paid edition**. On the **Free edition**, these procedures are not installed — attempting to call them will result in a "procedure not found" or "not authorized" error.

If a user reports that a procedure call is failing with "unknown procedure", "not found", or a privilege error on `SETUP.GDC_TASK` / `GDC_CONFIG` / `GDC_CONNECTION` / `GDC_DISCOVER`, the most likely cause is that the Free edition is installed. In that case:

1. Confirm by asking the user to check **Settings** in the Geo Data Connector application — the **Ingestion row limit** field shows "10,000 rows" on the Free edition and "Unlimited" on the paid edition.
2. Explain that the SQL procedures are not available on the Free edition.
3. Direct the user to use the Geo Data Connector application directly for discovery, ingestion, and task management.
4. If they need the SQL procedures, advise them to upgrade to the paid edition via the Snowflake Marketplace.

Do **not** retry the procedure call or suggest workarounds — the procedures are not present on the Free edition and no SQL workaround exists.

The Free edition also differs in these ways (relevant to user questions, not to procedure behavior):
- Ingestion tasks: up to 10 active tasks
- Rows per ingestion run: up to 10,000 (datasets over this limit, or with unknown row counts, cannot be ingested)
- Ingestion runs per day: up to 10 (resets at midnight UTC; skipped runs show status SKIPPED)
- Ingestion schedule: On-demand only (Daily, Weekly, Monthly are not available)
- Authenticated connections: not available

All procedures live in the `SETUP` schema of the Geo Data Connector app database. The app database name varies per installation (e.g., SDH_GEO_DATA_CONNECTOR, MY_GDC_APP) — it is unknown until you detect or ask for it. **Calling any procedure without the correct database context WILL FAIL.**

## MANDATORY: Set Database Context Before ANY Procedure Call

**You MUST know the GDC app database name before calling ANY procedure. If the database name has not been established in this conversation, you MUST either auto-detect it or ASK the user — do NOT guess, do NOT attempt calls without it, do NOT skip this step.** The database name is chosen by the consumer at install time and cannot be predicted.

**If the user tells you which database to use** (e.g., "use MY_GDC_APP"), store that name and use it as the fully-qualified prefix for all procedure calls (e.g., `CALL MY_GDC_APP.SETUP.GDC_TASK(...)`). Use the **exact text the user provided** — do NOT interpret, map, or fuzzy-match their input to other database names.

**Otherwise, auto-detect** by running:

```sql
SHOW APPLICATIONS;
SELECT "name", "source", "version"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "source" LIKE 'SDH_GDC_APP_PKG%';
```

Then follow this logic:

1. **One result:** Store the `"name"` value and use it as the fully-qualified prefix for all procedure calls (e.g., `CALL <name>.SETUP.GDC_TASK(...)`). Proceed.
2. **Multiple results:** List all results and ask the user which one to use. **STOP here and wait for the user's response.** Do NOT pick one yourself. Only after the user responds, use the **exact text the user provided** as the database prefix — do NOT interpret, map, or fuzzy-match their response to the auto-detected list. The user may provide a database name that is not in the list (e.g., a dev/test database) — use it as-is.
3. **No results:** Tell the user no Geo Data Connector installation was found. **STOP here and ask the user for the database name.** Do NOT guess or proceed without it.

**NEVER execute a `CALL` statement against a `SETUP.*` procedure until you have confirmed the database name — either from the user or from a single auto-detect result. NEVER skip this step.** Even if you think you know the database name from a previous conversation, run the detection query to verify. Always use the database name as a fully-qualified prefix — do NOT rely on `USE DATABASE` persisting across tool calls.

### Calling Convention

**Always use fully-qualified names** to avoid database context issues. `USE DATABASE` does NOT persist across separate SQL tool calls — each call runs in its own session. Fully-qualified names work reliably every time:

```sql
CALL <gdc_app_database>.SETUP.<procedure>('<action>', <params>);
-- Example: CALL SDH_GEO_DATA_CONNECTOR.SETUP.GDC_TASK('list', NULL);
```

> **Note:** SQL examples below use `SETUP.<procedure>` for brevity. In practice, always prefix with the database name: `<gdc_app_database>.SETUP.<procedure>`.

- `action` is a STRING identifying the operation.
- `params` is a VARIANT (JSON object) with action-specific parameters, or NULL when none are needed.
- Every call returns a JSON string: `{"ok": true, "action": "...", "data": {...}, "error": null}`.
- Use `PARSE_JSON()` to extract fields from the result.

### Available Procedures

There are exactly **four** procedures. No other procedures exist — do NOT call any procedure not listed here:

| Procedure | Purpose |
|-----------|---------|
| `GDC_CONFIG` | Application configuration and settings |
| `GDC_CONNECTION` | Saved data source connections |
| `GDC_TASK` | Task lifecycle, status, run history, and table info |
| `GDC_DISCOVER` | Data source discovery and search |

### Output Format

All procedures support an optional third `format` parameter for tabular output:

| Call | Returns |
|------|---------|
| `CALL SETUP.GDC_TASK('list', NULL)` | JSON string |
| `CALL SETUP.GDC_TASK('list', NULL, 'table')` | Result set (tabular) |

**Always use the JSON format (2-arg) by default.** JSON is reliable across all procedures and easy to parse programmatically. Only use the table format (3-arg with `'table'`) when the user explicitly requests tabular output.

`sortorder` (where applicable): valid values are `asc` (default) or `desc`.

### Task Names

Task names are auto-generated hashes (e.g., `INGESTION_d29a11586bc041a42c7225f06f275e58`). You cannot guess or construct them. Always obtain task names from:
- `GDC_TASK('list', ...)` — returns `TASK_NAME` column
- `GDC_TASK('create', ...)` — returns `name` in the response data
- `GDC_TASK('get', ...)` — lookup by `name` or by `url`, `type`, and `dataset`

## GDC_CONFIG — Application Configuration

Role: APP_ADMIN

| Action | Parameters | Description |
|--------|-----------|-------------|
| `list_settings` | none | List user-facing configuration settings |
| `get_setting` | `key` | Get a specific setting by key |
| `set_setting` | `key`, `value` | Create or update a setting |
| `get_warehouse_status` | none | Check whether the warehouse is available (returns `{available: bool}`) |
| `get_eai_status` | none | Check whether External Access Integration is configured (returns `{configured: bool}`) |
| `get_auto_suspend_status` | none | Check auto-suspend configuration |

Readable settings: `ingestion-database`, `notification-email`, `notify-task-success`, `notify-task-failure`, `notify-idle-warning`, `notify-auto-suspend`, `notify-idle-hours`, `auto-suspend-enabled`, `auto-suspend-timeout`, `app-edition`, `app-version`, `instance-id`. Of these, `app-edition`, `app-version`, and `instance-id` are read-only and cannot be changed via `set_setting`.

```sql
CALL SETUP.GDC_CONFIG('list_settings', NULL);
CALL SETUP.GDC_CONFIG('get_setting', PARSE_JSON('{"key": "ingestion-database"}'));
CALL SETUP.GDC_CONFIG('set_setting', PARSE_JSON('{"key": "ingestion-database", "value": "MY_GDC_DATA"}'));
SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_warehouse_status', NULL)):data:available::BOOLEAN;
```

## GDC_CONNECTION — Saved Connections

Role: APP_USER

| Action | Parameters | Description |
|--------|-----------|-------------|
| `count` | none | Count saved connections |
| `list` | `page`, `length`, `sortkey`, `sortorder` | List connections with pagination. Valid `sortkey` values: `URI`, `TYPE`, `NAME_SERVICE`, `NAME`, `DATASET_COUNT`, `ACCESSED`, `CREATED`, `MODIFIED`. Default: `NAME`. |
| `upsert` | `uri`, `type`, `name_service`, `description`, `dataset_count`, `auth_method`, `auth_username`, `auth_password` | Create or update a connection. `auth_method`: `none` (default) or `basic`. `auth_username`/`auth_password` are write-only (stored encrypted, never returned by `list`). |
| `delete` | `uri` | Delete a connection |

```sql
CALL SETUP.GDC_CONNECTION('count', NULL);
CALL SETUP.GDC_CONNECTION('list', PARSE_JSON('{"page": 1, "length": 10}'));
CALL SETUP.GDC_CONNECTION('upsert', PARSE_JSON('{"uri": "https://geo.example.com/wfs", "type": "WFS", "name_service": "Example WFS"}'));
-- Connection with authentication (API key or username/password)
-- auth_method "basic" covers both API key and password-based authentication
CALL SETUP.GDC_CONNECTION('upsert', PARSE_JSON('{"uri": "https://geo.example.com/wfs", "type": "WFS", "name_service": "Example Authenticated WFS", "auth_method": "basic", "auth_username": "your-api-key"}'));
CALL SETUP.GDC_CONNECTION('delete', PARSE_JSON('{"uri": "https://geo.example.com/wfs"}'));
```

**Authenticated Connections**
- **Remove credentials:** call `upsert` with `auth_method: 'none'` — this clears stored credentials and reverts the connection to unauthenticated.
- **Partial update:** omit `auth_username` and/or `auth_password` from `upsert` to keep the current stored values unchanged.
- **Free edition restriction:** connections with `auth_method: 'basic'` are rejected with error code `FREE_TIER_RESTRICTED` on the Free edition. Authenticated connections require the paid edition.
- **`list` response:** returns `AUTH_METHOD` only — stored credentials are never returned (write-only storage).

## GDC_TASK — Task Lifecycle and Information

Role: APP_USER. Requires External Access Integration.

### Task Lifecycle Actions

| Action | Parameters | Description |
|--------|-----------|-------------|
| `create` | `url`, `type`, `dataset`, `schedule` (optional — omit for On-demand) | Create an ingestion task. Omitting `schedule` creates an On-demand task that runs once immediately. Returns `{"name": "INGESTION_...", "enabled": true}`. |
| `delete` | `name` OR (`url`, `type`, `dataset`) | Delete a task |
| `suspend` | `name` | Suspend a scheduled task |
| `resume` | `name`, `schedule` (optional) | Resume a suspended task |
| `set_schedule` | `name`, `schedule` | Change a task's schedule |
| `unset_schedule` | `name` | Remove a task's schedule |
| `execute` | `name` | Run a task immediately |

**Parameter names for `create`:** `url` (data source URL), `type` (service type, e.g. `WFS`), `dataset` (dataset name within the service).

**`delete` accepts either:** `name` alone, OR all three source identifiers (`url`, `type`, `dataset`).

Valid `type` values: `WFS`, `WMS`, `WMTS`, `WCS`, `OGC API Features`, `OGC API Tiles`, `OGC API Maps`, `OGC API Coverages`. Must match what discovery returns (case-sensitive).

Valid `schedule` values: `On-demand`, `Daily`, `Weekly`, or `Monthly` (case-insensitive). `On-demand` is the default — omitting `schedule` creates an On-demand task. On-demand tasks run once immediately on creation; use the `execute` action to run them again. On-demand tasks have no recurring schedule so `NEXT_RUN` is always "N/A".

A bare create with only `url`, `type`, and `dataset` is valid and creates an On-demand task that runs immediately:

```sql
-- Bare create (no schedule) → On-demand task, runs immediately (recommended for AI agents)
CALL SETUP.GDC_TASK('create', PARSE_JSON('{"url": "https://geo.example.com/wfs", "type": "WFS", "dataset": "protected_areas"}'));
-- Response: {"ok": true, "action": "create", "data": {"name": "INGESTION_d29a...", "enabled": true}}
-- SAVE the returned name — you need it for all subsequent operations on this task.

-- Create an on-demand ingestion task (explicit schedule; same as bare create)
CALL SETUP.GDC_TASK('create', PARSE_JSON('{"url": "https://geo.example.com/wfs", "type": "WFS", "dataset": "protected_areas", "schedule": "On-demand"}'));
-- Response: {"ok": true, "action": "create", "data": {"name": "INGESTION_d29a...", "enabled": true}}
-- SAVE the returned name — you need it for all subsequent operations on this task.

-- Create a daily ingestion task
CALL SETUP.GDC_TASK('create', PARSE_JSON('{"url": "https://geo.example.com/wfs", "type": "WFS", "dataset": "protected_areas", "schedule": "Daily"}'));
-- Response: {"ok": true, "action": "create", "data": {"name": "INGESTION_d29a...", "enabled": true}}
-- SAVE the returned name — you need it for all subsequent operations on this task.
```

`create` can also return `{"ok": false, "error": {"code": "...", "message": "..."}}` if the ingestion task could not be fully set up — a task record is still created (disabled, status FAILED) so you can inspect it via `get`/`list`, but no data will be retrieved until the underlying issue is resolved. Delete and recreate the task, or resume it once the issue clears:

```sql
CALL SETUP.GDC_TASK('create', PARSE_JSON('{"url": "https://geo.example.com/wfs", "type": "WFS", "dataset": "protected_areas", "schedule": "Daily"}'));
-- Response: {"ok": false, "action": "create", "data": {"name": "INGESTION_d29a...", "enabled": false}, "error": {"code": null, "message": "The Connector could not reach the data service. The ingestion task was created but is disabled — you can enable it once the service is available."}}

-- Run a task now
CALL SETUP.GDC_TASK('execute', PARSE_JSON('{"name": "INGESTION_d29a11586bc041a42c7225f06f275e58"}'));

-- Suspend a task
CALL SETUP.GDC_TASK('suspend', PARSE_JSON('{"name": "INGESTION_d29a11586bc041a42c7225f06f275e58"}'));

-- Delete by name
CALL SETUP.GDC_TASK('delete', PARSE_JSON('{"name": "INGESTION_d29a11586bc041a42c7225f06f275e58"}'));

-- Delete by source identifiers
CALL SETUP.GDC_TASK('delete', PARSE_JSON('{"url": "https://geo.example.com/wfs", "type": "WFS", "dataset": "protected_areas"}'));
```

`resume`, `suspend`, `delete`, `set_schedule`, and `unset_schedule` can also
return `{"ok": false, "error": {"code": "...", "message": "..."}}` (or, for
`resume`, `{"ok": true, "data": {"enabled": false}, "error": {...}}`) when
the data service could not be reached to confirm the change:

- **`resume` failure:** the task stays disabled (status FAILED), and
  `STATUS_NOTE`/`ERROR_MESSAGE` on the next `list`/`get` explains why. Try
  `resume` again once the data service is reachable.
- **`delete` failure:** the task is **not deleted** (kept so you can retry);
  call `delete` again once the data service is reachable.
- **`suspend`/`set_schedule`/`unset_schedule` failure:** the change still
  applies within the Connector itself; retry if the response includes an
  `error`.

**After successful task creation:** Confirm the task was created with its name, schedule, and enabled status. Then ask: "Would you like me to monitor the ingestion progress?" Do NOT ask "Would you like me to run it now?" — a created task with any schedule (including On-demand) is already enabled and its first ingestion run starts automatically on creation. The `execute` action should only be offered when the user explicitly asks to run again, when resuming a suspended task, or when re-ingesting to refresh existing data.

**Workflow for suspend/resume/delete:** Call `GDC_TASK('list', ...)` first to get the task's `TASK_NAME`, then pass it to the lifecycle action.

### Task Information Actions (Read-Only)

| Action | Parameters | Description |
|--------|-----------|-------------|
| `count` | none | Count ingestion tasks |
| `list` | `page`, `length`, `sortkey`, `sortorder`, `filter`, `status`, `enabled` | List tasks with pagination. Valid `sortkey` values: `TASK_NAME`, `SOURCE_TYPE`, `DATA_SOURCE`, `DATASET`, `TARGET_TABLE`, `STATUS`, `NEXT_RUN`, `ENABLED`, `SCHEDULE`, `CREATED`, `LAST_RUN`. Default: `TASK_NAME`. Optional `filter`: exact task name (case-insensitive). Optional `status`: `SUCCESS`, `FAILED`, `RETRYING`, `IN PROGRESS`, `INACTIVE`. Optional `enabled`: `true` or `false`. |
| `get` | `name` OR (`url`, `type`, `dataset`) | Get a single task (same columns as `list`). Does NOT include run history. |
| `runs` | `name` | Get **ingestion run history** — timestamps, status (succeeded/failed), duration. Use this when the user asks about task runs, ingestion status, or when the last ingestion happened. |
| `table_info` | `name` | Describe the ingested table (columns, row count, sample) |
| `metadata` | `name` | Get raster metadata (WMS, WMTS, WCS, OGC API Maps, OGC API Coverages) |
| `stage_file_info` | `name` | Get staged image file info (WMS, WMTS) |

**When the user asks about a task, call both `get` and `runs`** to show the full picture (configuration + execution history).

**`list`/`get` response columns:** `TASK_NAME`, `SOURCE_TYPE`, `DATA_SOURCE`, `DATASET`, `TARGET_TABLE`, `STATUS`, `STATUS_NOTE`, `NEXT_RUN`, `LAST_RUN`, `CREATED`, `ENABLED`, `SCHEDULE`.

- `DATA_SOURCE`: The title of the data source. Falls back to the URL if no title is available.
- `STATUS`: One of `SUCCESS`, `FAILED`, `RETRYING`, `IN PROGRESS`, or `INACTIVE`.
- `STATUS_NOTE`: Context for the current status. While ingestion is running, shows the current execution phase (e.g., "3/6 Retrieving") — or, if the data source is temporarily unavailable, a message indicating the Connector is waiting for it to become available. When the task has failed, contains the failure description. When retrying, shows the retry count and error (e.g., "Retry 1/2 — timeout"); the next-retry time is in `NEXT_RUN`. Null when not applicable.
- `NEXT_RUN`: The next scheduled run time (trigger time) with a relative suffix (e.g., "2026-04-08 06:00:00 (in 12 hours)"), or "N/A" for on-demand or disabled tasks.
- `TARGET_TABLE`: Fully qualified Snowflake table name (e.g., `MY_DB.SCHEMA.TABLE`), or "N/A" if ingestion has not completed.
- `ENABLED`: "Enabled" or "Disabled".
- `SCHEDULE`: "On-demand", "Daily", "Weekly", "Monthly", or "N/A". "On-demand" means the task runs once on creation and again only when triggered manually via `execute`.

**`runs` response columns:** `NAME`, `START_TIME`, `END_TIME`, `STATUS`, `STATUS_NOTE`, `ERROR_MESSAGE`, `DURATION`, `TRIGGER`, `ROWS_LOADED`.

- `STATUS`: Run outcome — `RUNNING`, `SUCCEEDED`, `FAILED`, `SKIPPED`, or `POSTPONED`. `SKIPPED` means the run was not attempted because the daily run limit (10 per day) was reached on the Free edition; the counter resets at midnight UTC. On the paid edition, `SKIPPED` does not appear. `POSTPONED` means the data source was temporarily unavailable when the run started; for scheduled tasks the Connector retries automatically (the interval is determined by data-source conditions and is not fixed), for on-demand tasks the task returns to inactive and can be run again manually. After the data source remains unavailable through repeated postponements, the task is automatically disabled and the STATUS becomes `FAILED`. The `STATUS_NOTE` column contains a brief explanation.
- `STATUS_NOTE`: Execution phase at the time of the status update (e.g., "3/6 Retrieving"). Null for completed runs.
- `ERROR_MESSAGE`: Failure description for failed runs. Null for succeeded runs.
- `DURATION`: Formatted duration string (e.g., "2m 30s").
- `TRIGGER`: How the run was started (e.g., "SCHEDULED", "MANUAL").

**`table_info` response:** `table` (fully qualified name), `column_count`, `row_count`, `columns` (list of `{name, type}`), `sample` (up to 10 rows).

```sql
-- Count tasks
SELECT PARSE_JSON(SETUP.GDC_TASK('count', NULL)):data:count::INT AS task_count;

-- List tasks sorted by most recent run (tabular output)
CALL SETUP.GDC_TASK('list', PARSE_JSON('{"page": 1, "length": 10, "sortkey": "LAST_RUN", "sortorder": "desc"}'), 'table');

-- List tasks (JSON output for parsing)
CALL SETUP.GDC_TASK('list', PARSE_JSON('{"page": 1, "length": 10, "sortkey": "TASK_NAME"}'));

-- Get a task by name
CALL SETUP.GDC_TASK('get', PARSE_JSON('{"name": "INGESTION_d29a11586bc041a42c7225f06f275e58"}'));

-- Get a task by source identifiers
CALL SETUP.GDC_TASK('get', PARSE_JSON('{"url": "https://geo.example.com/wfs", "type": "WFS", "dataset": "protected_areas"}'));

-- Get run history
CALL SETUP.GDC_TASK('runs', PARSE_JSON('{"name": "INGESTION_d29a11586bc041a42c7225f06f275e58"}'), 'table');

-- Describe ingested table
CALL SETUP.GDC_TASK('table_info', PARSE_JSON('{"name": "INGESTION_d29a11586bc041a42c7225f06f275e58"}'));
```

### Ingestion Task Statuses

| Status | Meaning | Task State |
|--------|---------|------------|
| **IN PROGRESS** | Ingestion is actively running. Check `STATUS_NOTE` for current phase (e.g., "3/6 Retrieving"). | Enabled |
| **SUCCESS** | Last ingestion completed successfully | Enabled |
| **RETRYING** | Ingestion failed but will automatically retry after a delay. Check `STATUS_NOTE` for retry details. | Enabled |
| **FAILED** | Ingestion failed and all retry attempts are exhausted, or the data source remained unavailable through repeated postponements; task auto-disabled. Check `STATUS_NOTE` for failure reason. | Disabled |
| **INACTIVE** | Manually suspended by user | Disabled |

**Key behaviors:**
- **Automatic retry on failure:** When a task fails, the Connector automatically retries it up to 2 times (configurable) with increasing delays before marking it as permanently FAILED.
- **Auto-disable after retries exhausted:** When all retry attempts are exhausted, the Connector disables the task. The user must manually re-enable it (via `resume`).
- **Schedule required:** A task created without a schedule is disabled and never runs. Always include `schedule` when creating tasks.
- **On-demand tasks:** Created with `schedule: "On-demand"` — runs once immediately on creation (status goes IN PROGRESS). `NEXT_RUN` is always "N/A". Re-run at any time using `execute`. On-demand tasks do not retry on failure; they go directly to FAILED.
- **State transitions:** Create (with schedule) → IN PROGRESS → SUCCESS or RETRYING → FAILED. Suspend → INACTIVE. Resume from FAILED or RETRYING → resets to IN PROGRESS (new ingestion starts). A task can also reach FAILED after repeated postponements while the data source stays unavailable, not only from pipeline/staging/loading failures.

## GDC_DISCOVER — Data Source Discovery

Role: APP_USER. Requires External Access Integration.

| Action | Parameters | Description |
|--------|-----------|-------------|
| `navigate` | `url`, `page`, `node_id`, `number`, `item_number`, `q` | Browse an OGC service tree (10 items per page) |
| `status` | `ticket_id` | Poll async discovery ticket until completed |
| `search` | `q`, `limit`, `offset`, `bbox`, `filters` | Search indexed datasets by keyword |

**URL credential restriction:** Do not include authentication credentials in the `url` parameter. URLs containing query parameters such as `apikey`, `token`, or `password` are rejected. Use `GDC_CONNECTION('upsert', ...)` with `auth_method` and `auth_username` to configure authentication for the connection instead.

```sql
-- Browse a WFS service (may return data directly or a ticket)
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://geo.example.com/wfs", "page": 1}'));

-- Browse a CSW service (URL with & query parameters works as-is in Snowsight)
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://www.geonorge.no/geonetwork/srv/nor/csw?service=CSW&request=GetCapabilities", "page": 1}'));

-- If the response contains type='ticket', poll until completed:
CALL SETUP.GDC_DISCOVER('status', PARSE_JSON('{"ticket_id": "abc-123"}'));
-- Repeat status calls until response has status='completed'

-- Search for datasets (returns results directly, no ticket)
CALL SETUP.GDC_DISCOVER('search', PARSE_JSON('{"q": "elevation data finland", "limit": 10}'));

-- Search with filters (country and/or service type)
CALL SETUP.GDC_DISCOVER('search', PARSE_JSON('{"q": "geology", "filters": {"country": "FI", "type": "WFS"}, "limit": 10}'));

-- Drill into a specific layer
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://geo.example.com/wfs", "node_id": "protected_areas"}'));

-- Drill-down by position: number selects the source, item_number selects an item within it.
-- Each item in the items array carries a 1-based ``number`` field you can use as item_number.
-- Level 1: items within source 1
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://geo.example.com/wfs", "number": 1}'));
-- Level 2 (dataset service such as WFS): selected dataset plus its column schema
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://geo.example.com/wfs", "number": 1, "item_number": 3}'));
-- Level 2 (catalog such as CSW): drills through the record and returns the
-- underlying data source (e.g. a WFS service) ready to navigate further
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://www.geonorge.no/geonetwork/srv/nor/csw?service=CSW&request=GetCapabilities", "number": 1, "item_number": 3}'));
```

**URLs with `&` characters:** URLs containing query parameters (e.g., `?service=CSW&request=GetCapabilities`) work as-is in Snowsight. If using a SQL client that interprets `&` as variable substitution, escape with `&&`:

```sql
-- Use && if your SQL client treats & as variable substitution
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://example.com/csw?service=CSW&&request=GetCapabilities"}'));
```

### Async Discovery Pattern

The `navigate` action uses an async pattern:
- **Immediate**: Returns discovery data directly (fast, ~1-2s)
- **Deferred**: Returns `{"type": "ticket", "ticket_id": "...", "status": "processing"}`. Poll with the `status` action until `status` is `completed` (typically 5-30s).

When polling, check the response:
- `status: 'completed'` — discovery data is in the response
- `status: 'processing'` — wait 2-4 seconds and poll again
- `status: 'failed'` — discovery failed, check `error` field. If `error_code` is `DISCOVERY_AUTH_FAILED`, the service requires authentication — configure credentials via the Connections page or `GDC_CONNECTION('upsert', ...)`

The `search` action always returns results directly (no ticket).

### Discovery Response Structure

Discovery responses contain one or more of the following keys, depending on what was found:

- `source` — a single data source with its details (appears at the service overview level)
- `sources` — an array of data sources (appears when a URL resolves to multiple services)
- `items` — an array of items within a source (see item kinds below). Each item has a 1-based `number` field for drill-down
- `item` — a single item's detail (appears at the item drill-down level)
- `columns` — an array of columns for the selected item
- `source_count`, `service_health`, `search_capabilities` — top-level metadata (only present when they have a value)
- `status_note` — present when a single-service drill-down could not fully load its datasets (see "Partial Single-Service Drill-down" below); absent on full results

**`source` object fields (service overview / single-source detail):**

| Field | Description |
|-------|-------------|
| `title` | Data source title |
| `description` | Data source description |
| `type` | Service type (e.g., `WFS`, `WMS`, `CSW`) |
| `url` | Service URL |
| `service_provider` | Name of the organization operating the service |
| `dataset_count` | Number of datasets available from this source (0 when source could not be fully loaded) |
| `data_source_count` | Number of nested data sources (catalog entries), when the source is a catalog |
| `status_note` | `null` when the source loaded normally; a user-facing message (e.g., `"Couldn't load - try again"`) when the source could not be fully loaded |
| `keywords` | List of descriptive terms provided in the service metadata, or `null` |
| `crs_epsg_code` | Default coordinate reference system reported by the service (e.g., `"EPSG:4326"`), or `null` |
| `other_crs` | List of all coordinate reference systems the service supports, or `null` |
| `data_formats` | List of output formats the service supports, or `null` |
| `fees` | Fee information as reported in the service metadata, or `null`. This text is provided by the data source owner — the user is responsible for reviewing any applicable conditions. |
| `access_restrictions` | Usage terms as reported in the service metadata, or `null`. This text is provided by the data source owner — the user is responsible for reviewing and complying with any restrictions. |

These service-detail fields (`keywords`, `crs_epsg_code`, `other_crs`, `data_formats`, `fees`, `access_restrictions`) appear on the single `source` object only, not on `sources` list items.

**Table format — listing datasets or catalog records.** In the `'table'` format, calling `navigate` with only a `url` returns **a single row describing the data source** (its title, type, url, dataset counts). It does **not** list the datasets or catalog records as rows. To get those as rows, add `"number": 1` (or the row number of the source you want to expand when the URL returned multiple sources):

```sql
-- Table format: just the service summary (one row)
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://example.com/wfs"}'), 'table');
-- Table format: list the service's datasets as rows
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://example.com/wfs", "number": 1}'), 'table');
```

In the `'json'` format, the same `{url}` call already returns both the source summary and the items list together, so `number` is only needed when the URL exposes multiple sources and you want to select one.

**Two item kinds.** Items are either **nested data sources** (when browsing a catalog such as CSW or OGC API Records) or **datasets** (when browsing a dataset-providing service such as WFS, WMS, WCS, WMTS, OGC API Features). The available fields and the `item_number` result differ:

| Item kind | Typical services | Fields | `item_number` result |
|-----------|------------------|--------|----------------------|
| Nested data source | CSW, OGC API Records | `number, title, description, type, url, service_provider, data_source_count, dataset_count, status_note` | The Connector drills through the record and returns the underlying data source as `{source, ...}` — typically a fully populated WFS/WMS service with its own `url`, ready to navigate further. Records that point at multiple services return `{sources: [...]}`. |
| Dataset | WFS, WMS, WCS, WMTS, OGC API Features/Maps/Tiles/Coverages | `number, name, title, description, type, source_type, url, bbox_wgs84, crs_epsg_code, crs_name, column_count, row_count, is_ingestible, ingestibility_note, column_schema_note, fit` | The selected dataset plus a `columns` list describing its column schema (`{item, columns, ...}`). `type` is always `dataset`; `source_type` names the protocol (`WFS`, `WMS`, `OGC API Features`, …). |

### Empty Discovery Responses

When `navigate` or `status` returns `completed` but the response contains no `source`, `sources`, or `items`, the Connector reached the URL but found no geospatial data sources. The response still includes a `service_health` object.

Common reasons for an empty response:
- **Incorrect URL** — The URL does not point to a valid OGC service.
- **Decommissioned service** — The service was previously available but has been taken offline or moved.
- **Unsupported service version** — The service uses an OGC protocol version the Connector does not support.
- **Network or access restrictions** — The service requires authentication or blocks external access.

Check `service_health.status`: `HEALTHY` or `DEGRADED` means the service was reachable but returned no datasets; `UNKNOWN` or `DOWN` suggests the service could not be probed or is unreachable.

### Service Health in Discovery Responses

Every discovery response includes a `service_health` object at the top level:

| Field | Description |
|-------|-------------|
| `status` | `HEALTHY`, `DEGRADED`, `DOWN`, `HALF_OPEN`, or `UNKNOWN` |
| `last_probe` | ISO 8601 timestamp of last health check (null if never checked) |
| `response_time_ms` | Response time of the most recent health check in ms (null if never checked) |
| `uptime_pct_7d` | 7-day uptime percentage (null if insufficient data) |
| `uptime_pct_30d` | 30-day uptime percentage (null if insufficient data) |
| `avg_response_ms_7d` | 7-day average response time in ms (null if insufficient data) |
| `availability` | `available`, `temporarily_unavailable`, or `permanently_unavailable` (null if never checked) |

Use this to warn users about unhealthy services before they attempt ingestion.

Health status reflects server reachability — not data quality or authentication validity. Services behind authentication may show `HEALTHY` because the server is reachable, even though it returns an access-denied response. `UNKNOWN` means the service has not been checked yet. The Connector checks services automatically at intervals appropriate to each service's status history.

### Dataset Ingestibility

Each dataset in a discovery response includes `is_ingestible` (true/false). When false, `ingestibility_note` provides a human-readable explanation: unsupported dataset type, invalid coordinate reference system, pixel count exceeding the limit, feature count exceeding the limit, file size exceeding the limit, or a temporarily unreachable ingestion URL. When true, the dataset can be used to create an ingestion task.

### Ingestion-Fit Advisory (`fit`)

Dataset items may also carry a read-only size advisory: `fit: {verdict, confidence, estimated_size, capacity_limit, assessed_at}`. `verdict` is one of `fits_full`, `never_fits_full`, or `n_a` today — treat the set as OPEN and interpret any unknown value as `n_a` (future versions add more verdicts). `estimated_size` is `{metric, value, unit}` (for example `feature_count`/`features` or `pixel_count`/`pixels`) or `null` when no estimate could be computed; `capacity_limit` is the ingestion capacity in the SAME unit as `estimated_size.unit`, or `null`. `confidence` (`exact`/`estimated`/`n_a`) qualifies the estimate and is independent of `row_count` formatting. The field may be absent — treat absence as "no advisory", never as an error. When `verdict` is `never_fits_full` on a dataset that is otherwise ingestible, warn the user BEFORE creating a task that the dataset's estimated size exceeds the ingestion capacity; the advisory is informational — the ingestion process itself remains the final authority.

### Partial Source Load

Nested data source items (returned when browsing a catalog such as CSW or OGC API Records) include a `status_note` field. When `null`, the source loaded normally. When non-null (for example, `"Couldn't load - try again"`), the Connector detected the data source but was unable to fully load it during this discovery request. In that case, `data_source_count` and `dataset_count` will be `0` even though the source exists.

When a source item has a non-null `status_note`, convey that the source could not be fully loaded and suggest retrying discovery for the same URL. Do not treat the source as empty or missing — it was detected but requires another attempt.

### Partial Single-Service Drill-down

When drilling into a specific service (by URL), the response may carry a top-level `status_note` field. When present, the Connector was unable to fully load that service's datasets during this request — the dataset list may be incomplete. Convey this to the user and suggest retrying discovery for the same URL with the same parameters. The `status_note` is a user-facing message (for example, `"Couldn't fully load this service's datasets — try again"`); use it as-is or paraphrase it naturally. Do not expose any internal status codes. When `status_note` is absent, the drill-down returned a complete dataset list.

### Dataset Schema Availability

Some datasets may include a `column_schema_note` field indicating the column schema could not be resolved during discovery — typically because the data source uses an external schema definition that is not publicly accessible. Ingestion may still succeed for these datasets.

### Presenting Search Results

The `search` action returns `results` (array), `total` (total matching count), and `query_info` (execution metadata). The `query_info` object includes `query` (the original search query) and `execution_time_ms` (query duration in milliseconds). When the query mentions a geographic region, the Connector automatically uses spatial context to improve relevance. Each result contains these fields:

| Field | Content | Example |
|-------|---------|---------|
| `title` | Dataset name | "UKContShelf BGS 1:1M Seabed Sediments" |
| `source` | Service name (human-readable) | "BGS Bedrock and Superficial geology" |
| `url` | Full service URL — extract domain for display | `ogc.bgs.ac.uk` from `https://ogc.bgs.ac.uk/cgi-bin/...` |
| `type` | Service type | WFS, WMS, WMTS, WCS |
| `score` | Relevance score (display-ready) | 0.54 |
| `description` | Dataset description — show first sentence | "Seabed sediment data from..." |
| `discovered` | Human-readable relative time | "2 days ago", "today" |

**You MUST show exactly these 7 columns. Do NOT omit any column.**

| # | Column | Source field | Notes |
|---|--------|------------|-------|
| 1 | # | row number | |
| 2 | Title | `title` | |
| 3 | Score | `score` | Show as percentage (e.g., 0.54 → "54%") |
| 4 | Type | `type` | WFS/WMS/etc. |
| 5 | Service | `source` | human-readable service name |
| 6 | Domain | hostname from `url` | e.g., `ogc.bgs.ac.uk` |
| 7 | Discovered | `discovered` | already formatted, show as-is |

**After the table, always state:** "Showing N of M total results in Xms." (use `query_info.execution_time_ms` for the duration). If `total` > number shown, add: "Say 'show more' or refine your search to see additional results."

The `filters` parameter accepts an object with optional `country` (ISO 2-letter code) and `type` (service type like WFS, WMS). Example: `{"country": "FI", "type": "WFS"}`. Use filters when the user asks for data from a specific country or service type.

---

## Configuration Prerequisites

Before creating ingestion tasks, verify these prerequisites. If any check fails, task creation or ingestion will not work.

```sql
-- 1. Warehouse available?
SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_warehouse_status', NULL)):data:available::BOOLEAN AS warehouse_ok;

-- 2. External Access Integration configured?
SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_eai_status', NULL)):data:configured::BOOLEAN AS eai_ok;

-- 3. Ingestion target database set?
SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_setting', PARSE_JSON('{"key": "ingestion-database"}'))):data:VALUE::STRING AS ingestion_db;
```

- **Warehouse = FALSE:** The Connector needs an active warehouse. The user must grant warehouse access via Snowsight.
- **EAI = FALSE:** External Access Integration is required for the Connector to reach external data sources. The user must grant this permission via Snowsight.
- **Ingestion database = NULL:** The user must set a target database name via `GDC_CONFIG('set_setting', ...)` or through the Connector's Settings view.

---

## Ingested Data Schema

All datasets ingested by Geo Data Connector follow a consistent schema. Understanding this schema is essential for writing correct queries.

**Why ingest through the Connector.** All datasets delivered by Geo Data Connector — vector and raster/image sources alike — arrive as standardized tables sharing the same `geom_wgs84` geometry column and precomputed H3 indexing (raster arrives as a pixel table — see Raster Pixel Tables below). Because every ingested dataset shares this same geometry and indexing, datasets from different sources combine directly with no coordinate system or spatial index reconciliation. This is why ingesting through the Connector is preferable to querying a data source directly: a direct query returns the source's own native, unindexed data, while the Connector's ingested tables are already standardized and ready to combine with each other.

### Supported Data Source Types

| Type | Sources | Output Artifacts |
|------|---------|-----------------|
| Vector | WFS, OGC API Features, OGC API Tiles | Single feature table with GeoJSON geometry + source attributes |
| Raster/Image | WMS, WMTS, WCS, OGC API Maps, OGC API Coverages | Pixel data table + `__metadata` companion table |
| Image sources | WMS, WMTS, OGC API Maps | Additionally: Snowflake stage with downloadable georeferenced image files |
| Catalog | CSW, OGC API Records | Discovery and browsing only (not ingestible) |

### Target Table Naming

Ingested data is written to: `<ingestion_database>.<schema>.<table>`

- **Database**: The ingestion target database from application settings (see Configuration Prerequisites above)
- **Schema**: Auto-generated from the data source domain (e.g., `geo.stat.fi` → `FI_STAT`)
- **Table**: Derived from source type and dataset name with a unique suffix (e.g., `WFS__PROTECTED_AREAS_A1B2C3D4`)

To find a task's target table: use `GDC_TASK('table_info', ...)` which returns the table schema, row count, and sample data. The `GDC_TASK('get', ...)` response also includes the `TARGET_TABLE` field.

### Vector Tables (WFS, OGC API Features, OGC API Tiles)

Ingested vector feature tables contain source attributes alongside these system columns:

| Column | Type | Description |
|--------|------|-------------|
| `geom_wgs84` | STRING | GeoJSON geometry (Point, LineString, Polygon, Multi*) |
| `h3_indices` | ARRAY | Pre-computed H3 hexagonal cell indices at fine resolution, covering the feature geometry |
| `h3_resolution` | INT | H3 resolution level used for `h3_indices` (automatically selected based on dataset extent) |
| `h3_indices_coarse` | ARRAY | Pre-computed H3 cell indices at coarse resolution for regional aggregation |
| `h3_resolution_coarse` | INT | H3 resolution level for coarse indices |

Source-specific attribute columns (e.g., `ROCK_TYPE`, `AREA_NAME`, `STATUS`) vary by dataset.

**Attribute-only tables:** Some datasets have no geometry. These tables omit `geom_wgs84` entirely.

### Raster Pixel Tables (WMS, WMTS, WCS, OGC API Maps, OGC API Coverages)

All raster and image sources produce a pixel data table with one row per pixel:

| Column | Type | Description |
|--------|------|-------------|
| `pixel_id` | INT | Unique pixel identifier (row * width + col) |
| `x` | DECIMAL | Pixel center X coordinate in native CRS |
| `y` | DECIMAL | Pixel center Y coordinate in native CRS |
| `row` | INT | Pixel row index (0-based from top) |
| `col` | INT | Pixel column index (0-based from left) |
| `geom_wgs84` | STRING | GeoJSON Point for pixel center in native CRS coordinates |
| `band_1`, `band_2`, ... | DECIMAL | Raster band values (one column per band in the source) |

### Metadata Tables

Every raster/image ingestion produces a companion metadata table named `<table_name>__metadata` with image properties including: `source_type`, `crs`, `width`, `height`, `bands`, `band_names`, `band_units`, `band_value_range`, `band_nodata`, `pixel_size_x`, `pixel_size_y`, `pixel_value_type`, `pixel_count`, `file_size`, `file_location`, and geographic extent coordinates in both native CRS and WGS84.

### Image Stages (WMS, WMTS)

WMS and WMTS ingestions also produce georeferenced image files in a Snowflake stage. WCS and OGC API Coverages deliver raw raster data and do not produce an image stage.

---

## Geospatial Query Patterns

### Converting Geometry

GDC stores geometry as GeoJSON strings. Convert to Snowflake spatial types for analysis:

```sql
-- TO_GEOGRAPHY for distance and area calculations (spherical, meters)
SELECT ST_AREA(TO_GEOGRAPHY(geom_wgs84, TRUE)) / 1e6 AS area_sq_km
FROM my_db.provider.wfs__dataset;

-- TO_GEOMETRY for topological operations (planar)
SELECT ST_ISVALID(TO_GEOMETRY(geom_wgs84, TRUE)) AS is_valid
FROM my_db.provider.wfs__dataset;
```

Always pass `TRUE` as the second argument to allow slightly invalid geometries without errors.

### Filtering Null Geometries

Always filter before spatial operations:

```sql
WHERE geom_wgs84 IS NOT NULL
  AND ARRAY_SIZE(PARSE_JSON(geom_wgs84):coordinates) > 0
```

### Proximity Search

Find features within a distance of a point:

```sql
SELECT
    feature_name,
    ROUND(ST_DISTANCE(
        TO_GEOGRAPHY(geom_wgs84, TRUE),
        ST_MAKEPOINT(24.94, 60.17)  -- longitude, latitude (Helsinki)
    ) / 1000, 1) AS distance_km
FROM my_db.provider.wfs__monitoring_stations
WHERE geom_wgs84 IS NOT NULL
  AND ST_DISTANCE(
        TO_GEOGRAPHY(geom_wgs84, TRUE),
        ST_MAKEPOINT(24.94, 60.17)
    ) < 50000  -- 50 km in meters
ORDER BY distance_km;
```

### Area Calculation

```sql
SELECT
    region_name,
    ROUND(ST_AREA(TO_GEOGRAPHY(geom_wgs84, TRUE)) / 1e6, 2) AS area_sq_km
FROM my_db.provider.wfs__administrative_regions
WHERE geom_wgs84 IS NOT NULL
ORDER BY area_sq_km DESC;
```

### H3 Spatial Join (Fast)

Use pre-computed H3 indices to pre-filter spatial joins instead of running geometry
operations on every row pair. **IMPORTANT — always normalize H3 cells to a common
resolution before joining across datasets:** each dataset carries its own H3 resolution
(`h3_resolution` differs per dataset), and cell arrays can contain mixed resolutions
(coverings are compacted). Direct equality on un-normalized cells silently loses matches.
Normalize with `H3_CELL_TO_PARENT(cell, <common_resolution>)`, using the coarsest
resolution present in either dataset's `h3_indices_coarse`
(`SELECT MIN(H3_GET_RESOLUTION(h3.value::BIGINT)) FROM t, LATERAL FLATTEN(h3_indices_coarse) h3`;
3 is a safe default), then verify candidates with an exact geometry predicate:

```sql
-- Find which region each monitoring station falls in (two-stage: H3 prefilter + exact verify)
WITH stations AS (
    SELECT station_name, geom_wgs84,
           H3_CELL_TO_PARENT(h3.value::BIGINT, 3) AS h3_norm
    FROM my_db.fi_syke.wfs__monitoring_stations, LATERAL FLATTEN(h3_indices_coarse) AS h3
), regions AS (
    SELECT region_name, geom_wgs84,
           H3_CELL_TO_PARENT(h3.value::BIGINT, 3) AS h3_norm
    FROM my_db.fi_nls.wfs__administrative_regions, LATERAL FLATTEN(h3_indices_coarse) AS h3
), candidates AS (
    SELECT DISTINCT s.station_name, s.geom_wgs84 AS sg, r.region_name, r.geom_wgs84 AS rg
    FROM stations s JOIN regions r ON s.h3_norm = r.h3_norm
)
SELECT station_name, region_name
FROM candidates
WHERE ST_WITHIN(TO_GEOGRAPHY(sg, TRUE), TO_GEOGRAPHY(rg, TRUE));
```

### H3 Regional Aggregation

Group features by coarse H3 cells for regional summaries:

```sql
SELECT
    h3.value::STRING AS h3_cell,
    COUNT(*) AS feature_count,
    ROUND(AVG(attribute_value), 2) AS avg_value
FROM my_db.provider.wfs__features,
    LATERAL FLATTEN(h3_indices_coarse) AS h3
WHERE geom_wgs84 IS NOT NULL
GROUP BY h3_cell
ORDER BY feature_count DESC;
```

### Cross-Dataset Spatial Join

Join two ingested datasets from different providers — same two-stage pattern
(normalized-H3 prefilter, then exact geometry). To roll values up into containing
areas WITHOUT double counting, test the delivered `geom_centroid` so each feature
counts toward exactly one area:

```sql
-- Total grid population per administrative region (each square counted once)
WITH squares AS (
    SELECT population, geom_centroid,
           H3_CELL_TO_PARENT(h3.value::BIGINT, 3) AS h3_norm
    FROM my_db.fi_stat.wfs__population_grid, LATERAL FLATTEN(h3_indices_coarse) AS h3
), regions AS (
    SELECT region_name, geom_wgs84,
           H3_CELL_TO_PARENT(h3.value::BIGINT, 3) AS h3_norm
    FROM my_db.fi_nls.wfs__administrative_regions, LATERAL FLATTEN(h3_indices_coarse) AS h3
), candidates AS (
    SELECT DISTINCT s.population, s.geom_centroid, r.region_name, r.geom_wgs84
    FROM squares s JOIN regions r ON s.h3_norm = r.h3_norm
)
SELECT region_name, SUM(population) AS total_population
FROM candidates
WHERE ST_CONTAINS(TO_GEOGRAPHY(geom_wgs84, TRUE), TO_GEOGRAPHY(geom_centroid, TRUE))
GROUP BY region_name
ORDER BY total_population DESC;
```

For a quick approximate overlap count (exploratory work), the H3 prefilter alone can
be used without the geometry stage — accept that features near cell boundaries may
match neighbouring cells, and note per-feature cell counts are not comparable across
datasets (resolutions differ).

### Raster-Vector Overlay

Join raster pixel data with vector boundaries:

```sql
SELECT
    regions.region_name,
    COUNT(*) AS pixel_count,
    ROUND(AVG(pixels.band_1), 2) AS avg_band_1
FROM my_db.provider.wms__landcover AS pixels
JOIN my_db.provider.wfs__administrative_regions AS regions
    ON ST_WITHIN(
        ST_MAKEPOINT(pixels.x, pixels.y),
        TO_GEOMETRY(regions.geom_wgs84, TRUE)
    )
WHERE regions.geom_wgs84 IS NOT NULL
GROUP BY regions.region_name
ORDER BY pixel_count DESC;
```

### Geographic Extent Filter

Filter features within a geographic area:

```sql
SELECT *
FROM my_db.provider.wfs__features
WHERE geom_wgs84 IS NOT NULL
  AND ST_WITHIN(
    TO_GEOGRAPHY(geom_wgs84, TRUE),
    TO_GEOGRAPHY('POLYGON((24.5 60.0, 25.5 60.0, 25.5 60.5, 24.5 60.5, 24.5 60.0))')
  );
```

---

## Cortex AI Functions with Geospatial Data

Snowflake Cortex AI functions can enrich, classify, translate, and summarize ingested geospatial data directly in SQL.

### Translate Feature Descriptions

European OGC services often have descriptions in Finnish, French, German, etc.:

```sql
SELECT
    dataset_name,
    description AS original,
    AI_TRANSLATE(description, 'fi', 'en') AS english_description
FROM my_db.fi_syke.wfs__monitoring_stations;
```

### Classify Features by Category

```sql
SELECT
    feature_name,
    feature_type,
    AI_CLASSIFY(
        feature_type || ': ' || COALESCE(description, ''),
        ['residential', 'commercial', 'industrial', 'agricultural',
         'forest', 'wetland', 'water body', 'transport infrastructure']
    ) AS land_use_category
FROM my_db.provider.wfs__topographic_features;
```

### Extract Structured Metadata

```sql
SELECT
    dataset_name,
    AI_EXTRACT(
        description,
        'Extract: spatial resolution, temporal coverage, update frequency, data provider'
    ) AS structured_metadata
FROM dataset_catalog;
```

### Summarize Features by Region

Combine H3 aggregation with AI summarization:

```sql
SELECT
    h3.value::STRING AS h3_cell,
    COUNT(*) AS feature_count,
    AI_AGG(
        description,
        'Summarize the types of features found in this area in one sentence'
    ) AS area_summary
FROM my_db.uk_bgs.wfs__geology_625k,
    LATERAL FLATTEN(h3_indices_coarse) AS h3
WHERE description IS NOT NULL
GROUP BY h3_cell
ORDER BY feature_count DESC
LIMIT 20;
```

### Sentiment on Status Descriptions

```sql
SELECT
    station_name,
    status_description,
    AI_SENTIMENT(status_description) AS condition_score
FROM my_db.fi_syke.wfs__environmental_stations
WHERE status_description IS NOT NULL;
```

---

## Common Patterns

### Parse a scalar result
```sql
SELECT PARSE_JSON(SETUP.GDC_TASK('count', NULL)):data:count::INT;
```

### Check if a call succeeded
```sql
SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_warehouse_status', NULL)):ok::BOOLEAN;
```

### End-to-end workflow: discover, connect, ingest, analyze (paid edition)

```sql
-- 1. Search for datasets about a topic
CALL SETUP.GDC_DISCOVER('search', PARSE_JSON('{"q": "geological survey bedrock", "limit": 5}'));

-- 2. Browse the service to see available layers
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://geo.example.com/wfs", "page": 1}'));

-- 3. Save the connection (with authentication if needed)
CALL SETUP.GDC_CONNECTION('upsert', PARSE_JSON('{"uri": "https://geo.example.com/wfs", "type": "WFS", "name_service": "Example WFS"}'));
-- For authenticated services, include credentials:
-- CALL SETUP.GDC_CONNECTION('upsert', PARSE_JSON('{"uri": "https://geo.example.com/wfs", "type": "WFS", "name_service": "Example WFS", "auth_method": "basic", "auth_username": "your-api-key"}'));

-- 4. Create an ingestion task with a daily schedule
CALL SETUP.GDC_TASK('create', PARSE_JSON('{"url": "https://geo.example.com/wfs", "type": "WFS", "dataset": "bedrock_geology", "schedule": "Daily"}'));

-- 5. Check task status (wait for ingestion to complete)
CALL SETUP.GDC_TASK('list', PARSE_JSON('{"page": 1, "length": 5}'));

-- 6. Inspect the ingested table schema
CALL SETUP.GDC_TASK('table_info', PARSE_JSON('{"name": "INGESTION_abc123def456"}'));

-- 7. Query the ingested data
SELECT rock_type,
       COUNT(*) AS feature_count,
       ROUND(SUM(ST_AREA(TO_GEOGRAPHY(geom_wgs84, TRUE)) / 1e6), 1) AS total_area_sq_km
FROM my_db.provider.wfs__bedrock_geology
WHERE geom_wgs84 IS NOT NULL
GROUP BY rock_type
ORDER BY total_area_sq_km DESC;

-- 8. Enrich with AI
SELECT rock_type,
       AI_CLASSIFY(rock_type, ['igneous', 'sedimentary', 'metamorphic']) AS classification,
       COUNT(*) AS features
FROM my_db.provider.wfs__bedrock_geology
GROUP BY rock_type
ORDER BY features DESC;
```

---

## Task Creation Workflow

Before creating an ingestion task, always check whether one already exists for the same dataset. This avoids duplicates and surfaces existing data the user can query immediately.

**Step 1 — Look up existing task by source:**

```sql
CALL SETUP.GDC_TASK('get', PARSE_JSON('{"url": "<url>", "type": "<type>", "dataset": "<layer>"}'));
```

Parse the result. If `ok` is true and `data` contains a task, go to Step 2. If no task exists, skip to Step 6.

**Step 2 — Assess the existing task:**

Collect the task name from Step 1, then run these queries:

```sql
-- Run history (last ingestion time, success/failure)
CALL SETUP.GDC_TASK('runs', PARSE_JSON('{"name": "<task_name>"}'));

-- Ingestion target table existence and row count
CALL SETUP.GDC_TASK('table_info', PARSE_JSON('{"name": "<task_name>"}'));
```

Note from the Step 1 result: `STATUS` (SUCCESS/FAILED/RETRYING/IN PROGRESS/INACTIVE), `SCHEDULE` (Daily/Weekly/Monthly/N/A), and `ENABLED` (Enabled/Disabled).

**Step 3 — Summarize findings to the user:**

Present a concise summary covering:
- Task name and current state (enabled/suspended)
- Schedule (if any)
- Last successful ingestion (date, or "never" if no successful runs)
- Ingestion target table: exists (with row count and column list) or does not exist

**Step 4 — STOP and let the user choose:**

Offer these options based on the findings:
- **If table exists with data:** "Query existing data" (show table name) or "Re-ingest to refresh data" (`execute`)
- **If task exists but never succeeded:** "Run ingestion now" (`execute`) or "Delete and recreate"
- **If task is suspended:** "Resume task" (`resume`) or "Delete and recreate"

Wait for the user's choice before taking any action.

**Step 5 — Verify dataset is ingestible:**

Before creating a task, confirm the dataset supports ingestion. Check the `is_ingestible` field from the discovery response (`GDC_DISCOVER('navigate')` result). If `is_ingestible` is false, inform the user with the reason from `ingestibility_note` and do not proceed with task creation.

**Step 6 — Create new task (only when no existing task was found):**

**Always include a `schedule` parameter.** A task created without a schedule is disabled and will never run. **STOP and ask the user** for their preferred schedule (On-demand, Daily, Weekly, or Monthly) before creating. Wait for the user's response. Default to `On-demand` only if the user explicitly says they have no preference. Note: if the installation is on the Free edition (see Editions section), only `On-demand` is supported — do not offer or create Daily, Weekly, or Monthly schedules for a Free edition user.

```sql
CALL SETUP.GDC_TASK('create', PARSE_JSON('{"url": "<url>", "type": "<type>", "dataset": "<layer>", "schedule": "Daily"}'));
```

**Step 7 — After successful task creation:**

1. Confirm the task was created: show task name, schedule, and enabled status
2. Ask: "Would you like me to monitor the ingestion progress?"
3. Do NOT ask "Would you like me to run it now?" — a task with a schedule starts automatically
4. If the user wants monitoring: poll `GDC_TASK('runs', ...)` periodically and report when status changes from IN PROGRESS to SUCCESS, RETRYING, or FAILED
5. When ingestion succeeds: show the target table name and offer to describe or query the ingested data

---

## Troubleshooting Failed Ingestions

When a user reports a failed ingestion task, follow this diagnostic procedure.

### Available Diagnostic Signals

| Signal | How to Get | What It Tells You |
|--------|-----------|-------------------|
| STATUS = RETRYING | `GDC_TASK('list')` | Task failed but will automatically retry |
| STATUS = FAILED | `GDC_TASK('list')` | Task failed after all retries, auto-disabled |
| STATUS_NOTE on list | `GDC_TASK('list')` | Context for current status — phase, failure description, or retry count |
| STATUS_NOTE on run | `GDC_TASK('runs')` | Execution phase at time of status update |
| ERROR_MESSAGE on failed run | `GDC_TASK('runs')` | Failure description when STATUS = FAILED |
| ROWS_LOADED = 0 on failed run | `GDC_TASK('runs')` | Failure before any data retrieved — connectivity, permissions, or size limit |
| ROWS_LOADED > 0 on failed run | `GDC_TASK('runs')` | Partial data retrieved before failure — timeout or data quality issue |
| No successful runs ever | `GDC_TASK('runs')` full history | Likely persistent issue — incompatible dataset, size limit, or non-standard service |
| Previously succeeded, now failed | `GDC_TASK('runs')` full history | Likely transient issue — data source outage. Re-enabling usually works |
| Run STATUS = SKIPPED | `GDC_TASK('runs')` | Free edition daily run limit (10/day) was reached; run was not attempted. Resets at midnight UTC. |

### Diagnostic Steps

1. **Get the task status and error message:** `GDC_TASK('list')` — check `STATUS` and `STATUS_NOTE`
2. **Get run history:** `GDC_TASK('runs', ...)` — check `ERROR_MESSAGE`, `STATUS_NOTE`, `ROWS_LOADED`, and the pattern of successes/failures
3. **Verify the data source is reachable:**
   ```sql
   CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "<source_url>", "page": 1}'));
   ```
   If this fails or returns no datasets, the data source may be down or misconfigured.
4. **Check configuration prerequisites:**
   ```sql
   SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_warehouse_status', NULL)):data:available::BOOLEAN;
   SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_eai_status', NULL)):data:configured::BOOLEAN;
   ```
5. **Check connection authentication (if the data source requires credentials):**
   ```sql
   CALL SETUP.GDC_CONNECTION('list', PARSE_JSON('{"page": 1, "length": 50}'));
   ```
   Verify that the connection for the data source URL has `AUTH_METHOD` set to `basic`. If not, update it with `GDC_CONNECTION('upsert', ...)` including `auth_method` and `auth_username`. Credentials are write-only — you cannot view stored credentials, only verify that `AUTH_METHOD` is set.

### Checking Service Health Before Ingestion

Before creating tasks for a data source, check its health status:

```sql
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://geo.example.com/wfs"}'));
```

Inspect the `service_health.status` field in the response. If `DOWN` or `DEGRADED`, warn the user that ingestion may fail or be slow.

### Common Causes and Recommendations

| Cause | Pattern | Recommendation |
|-------|---------|----------------|
| Data source temporarily unavailable | Previously succeeded, now failed | Re-enable the task (`resume`). Most failures resolve on the next run. |
| Network or timeout issues | ROWS_LOADED > 0, intermittent | Re-enable the task. Large or complex datasets may occasionally time out. |
| Dataset exceeds size limits (WCS) | Never succeeded, ROWS_LOADED = 0 | WCS coverages have a maximum size limit. Contact support for geographic subsetting options. |
| Non-standard service behavior | Never succeeded, ROWS_LOADED = 0 | The data source may list datasets that cannot actually be retrieved. Try a different dataset from the same source. |
| Application permissions revoked | All tasks failing | Verify Geo Data Connector permissions are intact in Snowsight (Catalog → Apps). |
| Authentication failure | Never succeeded, ROWS_LOADED = 0, source requires API key | Configure authentication via `GDC_CONNECTION('upsert', ...)` with `auth_method: "basic"` and `auth_username`. |
| Free edition daily limit reached | Runs show STATUS = SKIPPED | The 10 runs/day limit was reached. The counter resets at midnight UTC; the next run after reset will proceed normally. No action needed. |

### Important Limitations

- When diagnosis is inconclusive, recommend the user contact the Geo Data Connector support team via the Snowflake Marketplace application page
