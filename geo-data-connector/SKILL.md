---
name: geo-data-connector
description: SQL procedures, ingested data schema, and geospatial query patterns for Geo Data Connector (GDC) — Snowflake Native App. Use when questions mention GDC, Geo Data Connector, or GDC_CONFIG, GDC_TASK, GDC_CONNECTION, GDC_DISCOVER, GDC_TASK_INFO procedures.
tools:
  - snowflake_execute
  - snowflake_sql_execute
metadata:
  author: smartdatahub
  version: "1.4.0"
---

# Geo Data Connector — SQL Procedure Reference

Geo Data Connector provides five compound SQL procedures for managing geospatial data ingestion programmatically. These procedures are available with the **paid edition** only.

All procedures live in the `SETUP` schema of the Geo Data Connector app database.

## MANDATORY: Detect the GDC App Database Before Any Procedure Call

**CRITICAL: ALWAYS run database detection FIRST before calling ANY procedure. NEVER guess or assume the database name.** The database name is chosen by the consumer at install time — it varies per installation and cannot be predicted.

**If the user tells you which database to use** (e.g., "use MY_GDC_APP"), execute `USE DATABASE <name>;` with the **exact text the user provided** — do NOT interpret, map, or fuzzy-match their input to other database names. Proceed with procedure calls.

**Otherwise, auto-detect** by running:

```sql
SHOW APPLICATIONS;
SELECT "name", "source", "version"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "source" LIKE 'SDH_GDC_APP_PKG%';
```

Then follow this logic:

1. **One result:** Execute `USE DATABASE <name>;` where `<name>` is the value from the `"name"` column. Proceed with procedure calls.
2. **Multiple results:** List all results and ask the user which one to use. **STOP here and wait for the user's response.** Do NOT pick one yourself. Only after the user responds, execute `USE DATABASE <chosen_name>;` using the **exact text the user provided** — do NOT interpret, map, or fuzzy-match their response to the auto-detected list. The user may provide a database name that is not in the list (e.g., a dev/test database) — use it as-is.
3. **No results:** Tell the user no Geo Data Connector installation was found. **STOP here and ask the user for the database name.** Do NOT guess or proceed without it.

**NEVER execute a `CALL SETUP.*` statement until you have confirmed the database name — either from the user or from a single auto-detect result. NEVER skip this step.** Even if you think you know the database name from a previous conversation, run the detection query to verify.

### Calling Convention

```sql
CALL SETUP.<procedure>('<action>', <params>);
```

- `action` is a STRING identifying the operation.
- `params` is a VARIANT (JSON object) with action-specific parameters, or NULL when none are needed.
- Every call returns a JSON string: `{"ok": true, "action": "...", "data": {...}, "error": null}`.
- Use `PARSE_JSON()` to extract fields from the result.

### Output Format

All procedures support an optional third `format` parameter for tabular output:

| Call | Returns |
|------|---------|
| `CALL SETUP.GDC_TASK_INFO('list', NULL)` | JSON string |
| `CALL SETUP.GDC_TASK_INFO('list', NULL, 'table')` | Result set (tabular) |

**Always use the JSON format (2-arg) by default.** JSON is reliable across all procedures and easy to parse programmatically. Only use the table format (3-arg with `'table'`) when the user explicitly requests tabular output.

`sortorder` (where applicable): valid values are `asc` (default) or `desc`.

### Task Names

Task names are auto-generated hashes (e.g., `INGESTION_d29a11586bc041a42c7225f06f275e58`). You cannot guess or construct them. Always obtain task names from:
- `GDC_TASK_INFO('list', ...)` — returns `NAME` column
- `GDC_TASK('create', ...)` — returns `name` in the response data
- `GDC_TASK_INFO('get_by_source', ...)` — lookup by source identifiers

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

Readable settings: `ingestion-database`, `notification-email`, `notify-task-success`, `notify-task-failure`, `notify-idle-warning`, `notify-auto-suspend`, `notify-idle-hours`, `auto-suspend-enabled`, `auto-suspend-timeout`, `theme`, `app-edition`, `app-version`, `instance-id`. Of these, `app-edition`, `app-version`, and `instance-id` are read-only and cannot be changed via `set_setting`.

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
| `upsert` | `uri`, `type`, `name_service`, `description`, `dataset_count` | Create or update a connection |
| `delete` | `uri` | Delete a connection |

```sql
CALL SETUP.GDC_CONNECTION('count', NULL);
CALL SETUP.GDC_CONNECTION('list', PARSE_JSON('{"page": 1, "length": 10}'));
CALL SETUP.GDC_CONNECTION('upsert', PARSE_JSON('{"uri": "https://geo.example.com/wfs", "type": "WFS", "name_service": "Example WFS"}'));
CALL SETUP.GDC_CONNECTION('delete', PARSE_JSON('{"uri": "https://geo.example.com/wfs"}'));
```

## GDC_TASK_INFO — Task Information (Read-Only)

Role: APP_USER

| Action | Parameters | Description |
|--------|-----------|-------------|
| `count` | none | Count ingestion tasks |
| `list` | `page`, `length`, `sortkey`, `sortorder`, `filter` | List tasks with pagination. Valid `sortkey` values: `NAME`, `SOURCE_TYPE`, `SOURCE_URI`, `SOURCE_DATASET_NAME`, `INGESTION_DATABASE`, `INGESTION_STATUS`, `NEXT_INGESTION`, `ENABLED`, `SCHEDULE`, `CREATED`, `LAST_RUN`, `INGESTION_TARGET_TABLE`. Default: `NAME`. Optional `filter`: exact task name (case-insensitive) to return a single task. |
| `get` | `name` | Get task **configuration** (source URL, type, schedule, state). Does NOT include run history. |
| `get_by_source` | `source_uri`, `source_type`, `source_dataset_name` | Find a task by its data source |
| `runs` | `name` | Get **ingestion run history** — timestamps, status (succeeded/failed), duration. Use this when the user asks about task runs, ingestion status, or when the last ingestion happened. |
| `table_info` | `name` | Describe the ingested table (columns, row count, sample) |
| `metadata` | `name` | Get raster metadata (WMS, WMTS, WCS, OGC API Maps, OGC API Coverages) |
| `stage_file_info` | `name` | Get staged image file info (WMS, WMTS) |

**When the user asks about a task, call both `get` and `runs`** to show the full picture (configuration + execution history).

**`list` response columns:** `NAME`, `SOURCE_TYPE`, `SOURCE_URI`, `SOURCE_DATASET_NAME`, `INGESTION_DATABASE`, `INGESTION_STATUS`, `NEXT_INGESTION`, `ENABLED`, `SCHEDULE`, `CREATED`, `LAST_RUN`, `INGESTION_TARGET_TABLE`.

**`runs` response columns:** `NAME`, `START_TIME`, `END_TIME`, `STATUS`, `DURATION`, `TRIGGER_TYPE`, `ROWS_LOADED`.

**`table_info` response:** `table` (fully qualified name), `column_count`, `row_count`, `columns` (list of `{name, type}`), `sample` (up to 10 rows).

```sql
-- Count tasks
SELECT PARSE_JSON(SETUP.GDC_TASK_INFO('count', NULL)):data:count::INT AS task_count;

-- List tasks sorted by most recent run (tabular output)
CALL SETUP.GDC_TASK_INFO('list', PARSE_JSON('{"page": 1, "length": 10, "sortkey": "LAST_RUN", "sortorder": "desc"}'), 'table');

-- List tasks (JSON output for parsing)
CALL SETUP.GDC_TASK_INFO('list', PARSE_JSON('{"page": 1, "length": 10, "sortkey": "NAME"}'));

-- Get run history
CALL SETUP.GDC_TASK_INFO('runs', PARSE_JSON('{"name": "INGESTION_d29a11586bc041a42c7225f06f275e58"}'), 'table');

-- Describe ingested table
CALL SETUP.GDC_TASK_INFO('table_info', PARSE_JSON('{"name": "INGESTION_d29a11586bc041a42c7225f06f275e58"}'));
```

## GDC_TASK — Task Lifecycle Management

Role: APP_USER. Requires External Access Integration.

| Action | Parameters | Description |
|--------|-----------|-------------|
| `create` | `source_uri`, `source_type`, `source_table_name`, `schedule` (optional) | Create an ingestion task. Returns `{"name": "INGESTION_...", "enabled": true/false}`. |
| `delete` | `name`, `source_uri`, `source_type`, `source_table_name` | Delete a task |
| `suspend` | `name`, `source_uri`, `source_type`, `source_table_name` | Suspend a scheduled task |
| `resume` | `name`, `source_uri`, `source_type`, `source_table_name`, `schedule` | Resume a suspended task |
| `set_schedule` | `name`, `source_uri`, `source_type`, `source_table_name`, `schedule` | Change a task's schedule |
| `unset_schedule` | `name`, `source_uri`, `source_type`, `source_table_name` | Remove a task's schedule |
| `execute` | `name` | Run a task immediately |

**IMPORTANT — All actions except `create` and `execute` require ALL FOUR identifiers:** `name`, `source_uri`, `source_type`, and `source_table_name`. Passing only `name` will fail with MISSING_PARAMETER. Get these values from `GDC_TASK_INFO('list', ...)` first.

Valid `source_type` values: `WFS`, `WMS`, `WMTS`, `WCS`, `Features`, `Tiles`, `Maps`, `Coverages`. Must match what discovery returns (case-sensitive).

Valid `schedule` values: `Daily`, `Weekly`, or `Monthly` (case-insensitive).

```sql
-- Create a daily ingestion task
CALL SETUP.GDC_TASK('create', PARSE_JSON('{"source_uri": "https://geo.example.com/wfs", "source_type": "WFS", "source_table_name": "protected_areas", "schedule": "Daily"}'));
-- Response: {"ok": true, "action": "create", "data": {"name": "INGESTION_d29a...", "enabled": true}}
-- SAVE the returned name — you need it for all subsequent operations on this task.

-- Run a task now
CALL SETUP.GDC_TASK('execute', PARSE_JSON('{"name": "INGESTION_d29a11586bc041a42c7225f06f275e58"}'));

-- Suspend a task (requires all 4 identifiers)
CALL SETUP.GDC_TASK('suspend', PARSE_JSON('{"name": "INGESTION_d29a11586bc041a42c7225f06f275e58", "source_uri": "https://geo.example.com/wfs", "source_type": "WFS", "source_table_name": "protected_areas"}'));
```

**After successful task creation:** Confirm the task was created with its name, schedule, and enabled status. Then ask: "Would you like me to monitor the ingestion progress?" Do NOT ask "Would you like me to run it now?" — a created task with a schedule is already enabled and its first ingestion run starts automatically. The `execute` action should only be offered when the user explicitly asks to run immediately, when resuming a suspended task, or when re-ingesting to refresh existing data.

**Workflow for suspend/resume/delete:** Always call `GDC_TASK_INFO('list', ...)` first to get the task's `NAME`, `SOURCE_URI`, `SOURCE_TYPE`, and `SOURCE_DATASET_NAME`, then pass all four to the lifecycle action.

### Ingestion Task Statuses

| Status | Meaning | Task State |
|--------|---------|------------|
| **IN PROGRESS** | Ingestion is actively running | Enabled |
| **SUCCESS** | Last ingestion completed successfully | Enabled |
| **FAILED** | Last ingestion failed; task auto-disabled | Disabled |
| **INACTIVE** | Manually suspended by user | Disabled |

**Key behaviors:**
- **Auto-disable on failure:** When a task fails, the Connector automatically disables it to prevent repeated unsuccessful attempts. The user must manually re-enable it (via `resume`).
- **Retry logic:** Tasks include built-in retry logic — a task is retried multiple times before being marked as FAILED.
- **Schedule required:** A task created without a schedule is disabled and never runs. Always include `schedule` when creating tasks.
- **State transitions:** Create (with schedule) → IN PROGRESS → SUCCESS or FAILED. Suspend → INACTIVE. Resume from FAILED → resets to IN PROGRESS (new ingestion starts).

## GDC_DISCOVER — Data Source Discovery

Role: APP_USER. Requires External Access Integration.

| Action | Parameters | Description |
|--------|-----------|-------------|
| `navigate` | `url`, `page`, `limit`, `node_id`, `number`, `q` | Browse an OGC service tree |
| `search` | `q`, `limit`, `offset`, `bbox` | Search indexed datasets by keyword |

```sql
-- Browse a WFS service
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://geo.example.com/wfs?request=GetCapabilities", "page": 1, "limit": 20}'));

-- Search for datasets
CALL SETUP.GDC_DISCOVER('search', PARSE_JSON('{"q": "elevation data finland", "limit": 10}'));

-- Drill into a specific layer
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://geo.example.com/wfs", "node_id": "protected_areas"}'));
```

### Presenting Search Results

The `search` action returns `results` (array) and `total` (total matching count). Each result contains these fields:

| Field | Content | Example |
|-------|---------|---------|
| `title` | Dataset name | "UKContShelf BGS 1:1M Seabed Sediments" |
| `source` | Service name (human-readable) | "BGS Bedrock and Superficial geology" |
| `url` | Full service URL — extract domain for display | `ogc.bgs.ac.uk` from `https://ogc.bgs.ac.uk/cgi-bin/...` |
| `type` | Service type | WFS, WMS, WMTS, WCS |
| `score` | Relevance score (display-ready) | 0.54 |
| `score_breakdown` | Relevance signal breakdown (for reference) | `{"semantic": 0.54}` |
| `description` | Dataset description — show first sentence | "Seabed sediment data from..." |
| `discovered` | Human-readable relative time | "2 days ago", "today" |

**You MUST show exactly these 8 columns. Do NOT omit any column.**

| # | Column | Source field | Notes |
|---|--------|------------|-------|
| 1 | # | row number | |
| 2 | Title | `title` | |
| 3 | Score | `score` | Show as percentage (e.g., 0.54 → "54%") |
| 4 | Type | `type` | WFS/WMS/etc. |
| 5 | Service | `source` | human-readable service name |
| 6 | Domain | hostname from `url` | e.g., `ogc.bgs.ac.uk` |
| 7 | Discovered | `discovered` | already formatted, show as-is |
| 8 | Description | `description` | first sentence, truncate if long |

**After the table, always state:** "Showing N of M total results." If `total` > number shown, add: "Say 'show more' or refine your search to see additional results."

---

## Configuration Prerequisites

Before creating ingestion tasks, verify these prerequisites. If any check fails, task creation or ingestion will not work.

```sql
-- 1. Warehouse available?
SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_warehouse_status', NULL)):data:available::BOOLEAN AS warehouse_ok;

-- 2. External Access Integration configured?
SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_eai_status', NULL)):data:configured::BOOLEAN AS eai_ok;

-- 3. Ingestion target database set?
SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_setting', PARSE_JSON('{"key": "ingestion-database"}'))):data:value::STRING AS ingestion_db;
```

- **Warehouse = FALSE:** The Connector needs an active warehouse. The user must grant warehouse access via Snowsight.
- **EAI = FALSE:** External Access Integration is required for the Connector to reach external data sources. The user must grant this permission via Snowsight.
- **Ingestion database = NULL:** The user must set a target database name via `GDC_CONFIG('set_setting', ...)` or through the Connector's Settings view.

---

## Ingested Data Schema

All datasets ingested by Geo Data Connector follow a consistent schema. Understanding this schema is essential for writing correct queries.

### Supported Data Source Types

| Type | Sources | Output Artifacts |
|------|---------|-----------------|
| Vector | WFS, OGC API Features, OGC API Tiles | Single feature table with GeoJSON geometry + source attributes |
| Raster/Image | WMS, WMTS, WCS, OGC API Maps, OGC API Coverages | Pixel data table + `__metadata` companion table |
| Image sources | WMS, WMTS, OGC API Maps, OGC API Tiles | Additionally: Snowflake stage with downloadable georeferenced image files |
| Catalog | CSW, OGC API Records | Discovery and browsing only (not ingestible) |

### Target Table Naming

Ingested data is written to: `<ingestion_database>.<schema>.<table>`

- **Database**: The ingestion target database from application settings (see Configuration Prerequisites above)
- **Schema**: Auto-generated from the data source domain (e.g., `geo.stat.fi` → `FI_STAT`)
- **Table**: Derived from source type and dataset name with a unique suffix (e.g., `WFS__PROTECTED_AREAS_A1B2C3D4`)

To find a task's target table: use `GDC_TASK_INFO('table_info', ...)` which returns the table schema, row count, and sample data. The `GDC_TASK_INFO('get', ...)` response also includes the `INGESTION_DATABASE` field.

### Vector Tables (WFS, OGC API Features, OGC API Tiles)

Ingested vector feature tables contain source attributes alongside these system columns:

| Column | Type | Description |
|--------|------|-------------|
| `geom_geojson` | STRING | GeoJSON geometry (Point, LineString, Polygon, Multi*) |
| `h3_indices` | ARRAY | Pre-computed H3 hexagonal cell indices at fine resolution, covering the feature geometry |
| `h3_resolution` | INT | H3 resolution level used for `h3_indices` (automatically selected based on dataset extent) |
| `h3_indices_coarse` | ARRAY | Pre-computed H3 cell indices at coarse resolution for regional aggregation |
| `h3_resolution_coarse` | INT | H3 resolution level for coarse indices |

Source-specific attribute columns (e.g., `ROCK_TYPE`, `AREA_NAME`, `STATUS`) vary by dataset.

**Attribute-only tables:** Some datasets have no geometry. These tables omit `geom_geojson` entirely.

### Raster Pixel Tables (WMS, WMTS, WCS, OGC API Maps, OGC API Coverages)

All raster and image sources produce a pixel data table with one row per pixel:

| Column | Type | Description |
|--------|------|-------------|
| `pixel_id` | INT | Unique pixel identifier (row * width + col) |
| `x` | DECIMAL | Pixel center X coordinate in native CRS |
| `y` | DECIMAL | Pixel center Y coordinate in native CRS |
| `row` | INT | Pixel row index (0-based from top) |
| `col` | INT | Pixel column index (0-based from left) |
| `geom_geojson` | STRING | GeoJSON Point for pixel center in native CRS coordinates |
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
SELECT ST_AREA(TO_GEOGRAPHY(geom_geojson, TRUE)) / 1e6 AS area_sq_km
FROM my_db.provider.wfs__dataset;

-- TO_GEOMETRY for topological operations (planar)
SELECT ST_ISVALID(TO_GEOMETRY(geom_geojson, TRUE)) AS is_valid
FROM my_db.provider.wfs__dataset;
```

Always pass `TRUE` as the second argument to allow slightly invalid geometries without errors.

### Filtering Null Geometries

Always filter before spatial operations:

```sql
WHERE geom_geojson IS NOT NULL
  AND ARRAY_SIZE(PARSE_JSON(geom_geojson):coordinates) > 0
```

### Proximity Search

Find features within a distance of a point:

```sql
SELECT
    feature_name,
    ROUND(ST_DISTANCE(
        TO_GEOGRAPHY(geom_geojson, TRUE),
        ST_MAKEPOINT(24.94, 60.17)  -- longitude, latitude (Helsinki)
    ) / 1000, 1) AS distance_km
FROM my_db.provider.wfs__monitoring_stations
WHERE geom_geojson IS NOT NULL
  AND ST_DISTANCE(
        TO_GEOGRAPHY(geom_geojson, TRUE),
        ST_MAKEPOINT(24.94, 60.17)
    ) < 50000  -- 50 km in meters
ORDER BY distance_km;
```

### Area Calculation

```sql
SELECT
    region_name,
    ROUND(ST_AREA(TO_GEOGRAPHY(geom_geojson, TRUE)) / 1e6, 2) AS area_sq_km
FROM my_db.provider.wfs__administrative_regions
WHERE geom_geojson IS NOT NULL
ORDER BY area_sq_km DESC;
```

### H3 Spatial Join (Fast)

Use pre-computed H3 indices for spatial joins instead of expensive geometry operations:

```sql
-- Find which region each monitoring station falls in
SELECT DISTINCT
    stations.station_name,
    regions.region_name
FROM my_db.fi_syke.wfs__monitoring_stations AS stations,
    LATERAL FLATTEN(stations.h3_indices) AS s_h3
JOIN (
    SELECT region_name, h3.value::STRING AS h3_cell
    FROM my_db.fi_nls.wfs__administrative_regions,
        LATERAL FLATTEN(h3_indices) AS h3
) AS regions
    ON s_h3.value::STRING = regions.h3_cell;
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
WHERE geom_geojson IS NOT NULL
GROUP BY h3_cell
ORDER BY feature_count DESC;
```

### Cross-Dataset Spatial Join

Join two ingested datasets from different providers using H3 (no geometry computation needed):

```sql
-- Find geological formations under each protected area
SELECT
    pa.area_name,
    geo.rock_type,
    COUNT(*) AS overlap_cells
FROM my_db.fi_syke.wfs__protected_areas AS pa,
    LATERAL FLATTEN(pa.h3_indices) AS pa_h3
JOIN (
    SELECT rock_type, h3.value::STRING AS h3_cell
    FROM my_db.uk_bgs.wfs__geology_625k,
        LATERAL FLATTEN(h3_indices) AS h3
) AS geo
    ON pa_h3.value::STRING = geo.h3_cell
GROUP BY pa.area_name, geo.rock_type
ORDER BY pa.area_name, overlap_cells DESC;
```

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
        TO_GEOMETRY(regions.geom_geojson, TRUE)
    )
WHERE regions.geom_geojson IS NOT NULL
GROUP BY regions.region_name
ORDER BY pixel_count DESC;
```

### Geographic Extent Filter

Filter features within a geographic area:

```sql
SELECT *
FROM my_db.provider.wfs__features
WHERE geom_geojson IS NOT NULL
  AND ST_WITHIN(
    TO_GEOGRAPHY(geom_geojson, TRUE),
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
SELECT PARSE_JSON(SETUP.GDC_TASK_INFO('count', NULL)):data:count::INT;
```

### Check if a call succeeded
```sql
SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_warehouse_status', NULL)):ok::BOOLEAN;
```

### End-to-end workflow: discover, connect, ingest, analyze

```sql
-- 1. Search for datasets about a topic
CALL SETUP.GDC_DISCOVER('search', PARSE_JSON('{"q": "geological survey bedrock", "limit": 5}'));

-- 2. Browse the service to see available layers
CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "https://geo.example.com/wfs?request=GetCapabilities", "page": 1, "limit": 20}'));

-- 3. Save the connection
CALL SETUP.GDC_CONNECTION('upsert', PARSE_JSON('{"uri": "https://geo.example.com/wfs", "type": "WFS", "name_service": "Example WFS"}'));

-- 4. Create an ingestion task with a daily schedule
CALL SETUP.GDC_TASK('create', PARSE_JSON('{"source_uri": "https://geo.example.com/wfs", "source_type": "WFS", "source_table_name": "bedrock_geology", "schedule": "Daily"}'));

-- 5. Check task status (wait for ingestion to complete)
CALL SETUP.GDC_TASK_INFO('list', PARSE_JSON('{"page": 1, "length": 5}'));

-- 6. Inspect the ingested table schema
CALL SETUP.GDC_TASK_INFO('table_info', PARSE_JSON('{"name": "INGESTION_abc123def456"}'));

-- 7. Query the ingested data
SELECT rock_type,
       COUNT(*) AS feature_count,
       ROUND(SUM(ST_AREA(TO_GEOGRAPHY(geom_geojson, TRUE)) / 1e6), 1) AS total_area_sq_km
FROM my_db.provider.wfs__bedrock_geology
WHERE geom_geojson IS NOT NULL
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
CALL SETUP.GDC_TASK_INFO('get_by_source', PARSE_JSON('{"source_uri": "<url>", "source_type": "<type>", "source_dataset_name": "<layer>"}'));
```

Parse the result. If `ok` is true and `data` contains a task, go to Step 2. If no task exists, skip to Step 5.

**Step 2 — Assess the existing task:**

Collect the task name from Step 1, then run these queries:

```sql
-- Run history (last ingestion time, success/failure)
CALL SETUP.GDC_TASK_INFO('runs', PARSE_JSON('{"name": "<task_name>"}'));

-- Ingestion target table existence and row count
CALL SETUP.GDC_TASK_INFO('table_info', PARSE_JSON('{"name": "<task_name>"}'));
```

Note from the Step 1 result: `INGESTION_STATUS` (enabled/suspended), `SCHEDULE` (Daily/Weekly/Monthly/null), and `STATE` (started/succeeded/failed).

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

**Step 5 — Create new task (only when no existing task was found):**

**Always include a `schedule` parameter.** A task created without a schedule is disabled and will never run. **STOP and ask the user** for their preferred schedule (Daily, Weekly, or Monthly) before creating. Wait for the user's response. Default to `Daily` only if the user explicitly says they have no preference.

```sql
CALL SETUP.GDC_TASK('create', PARSE_JSON('{"source_uri": "<url>", "source_type": "<type>", "source_table_name": "<layer>", "schedule": "Daily"}'));
```

**Step 6 — After successful task creation:**

1. Confirm the task was created: show task name, schedule, and enabled status
2. Ask: "Would you like me to monitor the ingestion progress?"
3. Do NOT ask "Would you like me to run it now?" — a task with a schedule starts automatically
4. If the user wants monitoring: poll `GDC_TASK_INFO('runs', ...)` periodically and report when status changes from IN PROGRESS to SUCCESS or FAILED
5. When ingestion succeeds: show the target table name and offer to describe or query the ingested data

---

## Troubleshooting Failed Ingestions

When a user reports a failed ingestion task, follow this diagnostic procedure. **The API deliberately does not expose internal error details** — only the signals below are available.

### Available Diagnostic Signals

| Signal | How to Get | What It Tells You |
|--------|-----------|-------------------|
| Status = FAILED | `GDC_TASK_INFO('list')` or `get_by_source` | Task failed and was auto-disabled |
| ROWS_LOADED = 0 on failed run | `GDC_TASK_INFO('runs')` | Failure before any data retrieved — connectivity, permissions, or size limit |
| ROWS_LOADED > 0 on failed run | `GDC_TASK_INFO('runs')` | Partial data retrieved before failure — timeout or data quality issue |
| No successful runs ever | `GDC_TASK_INFO('runs')` full history | Likely persistent issue — incompatible dataset, size limit, or non-standard service |
| Previously succeeded, now failed | `GDC_TASK_INFO('runs')` full history | Likely transient issue — data source outage. Re-enabling usually works |

### Diagnostic Steps

1. **Get the task status:** `GDC_TASK_INFO('list')` or `GDC_TASK_INFO('get_by_source', ...)` — check `INGESTION_STATUS`
2. **Get run history:** `GDC_TASK_INFO('runs', ...)` — check ROWS_LOADED and the pattern of successes/failures
3. **Verify the data source is reachable:**
   ```sql
   CALL SETUP.GDC_DISCOVER('navigate', PARSE_JSON('{"url": "<source_url>", "page": 1, "limit": 5}'));
   ```
   If this fails or returns no datasets, the data source may be down or misconfigured.
4. **Check configuration prerequisites:**
   ```sql
   SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_warehouse_status', NULL)):data:available::BOOLEAN;
   SELECT PARSE_JSON(SETUP.GDC_CONFIG('get_eai_status', NULL)):data:configured::BOOLEAN;
   ```

### Common Causes and Recommendations

| Cause | Pattern | Recommendation |
|-------|---------|----------------|
| Data source temporarily unavailable | Previously succeeded, now failed | Re-enable the task (`resume`). Most failures resolve on the next run. |
| Network or timeout issues | ROWS_LOADED > 0, intermittent | Re-enable the task. Large or complex datasets may occasionally time out. |
| Dataset exceeds size limits (WCS) | Never succeeded, ROWS_LOADED = 0 | WCS coverages have a maximum size limit. Contact support for geographic subsetting options. |
| Non-standard service behavior | Never succeeded, ROWS_LOADED = 0 | The data source may list datasets that cannot actually be retrieved. Try a different dataset from the same source. |
| Application permissions revoked | All tasks failing | Verify Geo Data Connector permissions are intact in Snowsight (Catalog → Apps). |

### Important Limitations

- **Do NOT promise specific error messages** — the API does not expose internal error details
- **Do NOT suggest querying internal tables** like `services.task` or `services.task_run` — consumers cannot access these directly
- **Do NOT reference internal architecture** (pipeline stages, Lambda functions, infrastructure)
- When diagnosis is inconclusive, recommend the user contact the Geo Data Connector support team via the Snowflake Marketplace application page

---

## Updating This Skill

This skill is published automatically by CI/CD. The source of truth is `.cortex/skills/gdc.md` in the App repository (GeoDataConnector). Changes pushed to the App trigger the pipeline, which copies the file to the public GitHub repository.

| App branch | Skills repo branch | Audience |
|------------|--------------------|----------|
| `main` | `preview` | Development / testing |
| `rel-*` | `main` | Stable release (matches Marketplace version) |

To pull the latest version into your environment:

```
cortex skill update smartdatahub/gdc-snowflake-skills
```

**Source repository:** [github.com/smartdatahub/gdc-snowflake-skills](https://github.com/smartdatahub/gdc-snowflake-skills)
