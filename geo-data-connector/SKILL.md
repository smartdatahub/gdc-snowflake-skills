---
name: geo-data-connector
description: SQL procedures, ingested data schema, and geospatial query patterns for Geo Data Connector (GDC) — Snowflake Native App. Use when questions mention GDC, Geo Data Connector, or GDC_CONFIG, GDC_TASK, GDC_CONNECTION, GDC_DISCOVER, GDC_TASK_INFO procedures.
tools:
  - snowflake_execute
  - snowflake_sql_execute
---

# Geo Data Connector — SQL Procedure Reference

Geo Data Connector provides five compound SQL procedures for managing geospatial data ingestion programmatically. These procedures are available with the **paid edition** only.

All procedures live in the `SETUP` schema of the Geo Data Connector app database.

### Finding the GDC Database

The database name is chosen by the consumer at install time and may differ from the default. Before calling any procedure, run this discovery query to find it:

```sql
SHOW APPLICATIONS;
SELECT "name", "source", "version"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "source" LIKE 'SDH_GDC_APP_PKG%';
```

- **One result:** Use that `name` as the database — `USE DATABASE <name>;`
- **Multiple results:** Ask the user which installation to use.
- **No results:** The app is not installed or the role lacks access. Ask the user for the database name.

Once the database is set, all procedure calls use `CALL SETUP.<procedure>(...)` without further qualification.

### Calling Convention

```sql
CALL SETUP.<procedure>('<action>', <params>);
```

- `action` is a STRING identifying the operation.
- `params` is a VARIANT (JSON object) with action-specific parameters, or NULL when none are needed.
- Every call returns a JSON string: `{"ok": true, "action": "...", "data": {...}, "error": null}`.
- Use `PARSE_JSON()` to extract fields from the result.

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
| `list` | `page`, `length`, `sortkey`, `sortorder` | List connections with pagination |
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
| `list` | `page`, `length`, `sortkey`, `sortorder` | List tasks with pagination |
| `get` | `name` | Get a task by its internal name (e.g. INGESTION_abc123) |
| `get_by_source` | `source_uri`, `source_type`, `source_dataset_name` | Find a task by its data source |
| `runs` | `name` | Get ingestion run history for a task |
| `table_info` | `name` | Describe the ingested table (columns, row count, sample) |
| `metadata` | `name` | Get raster metadata (WMS, WMTS, WCS, OGC API Maps, OGC API Coverages) |
| `stage_file_info` | `name` | Get staged image file info (WMS, WMTS) |

```sql
-- Count tasks
SELECT PARSE_JSON(SETUP.GDC_TASK_INFO('count', NULL)):data:count::INT AS task_count;

-- List tasks sorted by name
CALL SETUP.GDC_TASK_INFO('list', PARSE_JSON('{"page": 1, "length": 10, "sortkey": "NAME"}'));

-- Get run history
CALL SETUP.GDC_TASK_INFO('runs', PARSE_JSON('{"name": "INGESTION_abc123def456"}'));

-- Describe ingested table
CALL SETUP.GDC_TASK_INFO('table_info', PARSE_JSON('{"name": "INGESTION_abc123def456"}'));
```

## GDC_TASK — Task Lifecycle Management

Role: APP_USER. Requires External Access Integration.

| Action | Parameters | Description |
|--------|-----------|-------------|
| `create` | `source_uri`, `source_type`, `source_table_name`, `schedule` (optional) | Create an ingestion task |
| `delete` | `name`, `source_uri`, `source_type`, `source_table_name` | Delete a task |
| `suspend` | `name`, `source_uri`, `source_type`, `source_table_name` | Suspend a scheduled task |
| `resume` | `name`, `source_uri`, `source_type`, `source_table_name`, `schedule` | Resume a suspended task |
| `set_schedule` | `name`, `source_uri`, `source_type`, `source_table_name`, `schedule` | Change a task's schedule |
| `unset_schedule` | `name`, `source_uri`, `source_type`, `source_table_name` | Remove a task's schedule |
| `execute` | `name` | Run a task immediately |

Schedules: `Daily`, `Weekly`, or `Monthly`.

```sql
-- Create a daily ingestion task
CALL SETUP.GDC_TASK('create', PARSE_JSON('{"source_uri": "https://geo.example.com/wfs", "source_type": "WFS", "source_table_name": "protected_areas", "schedule": "Daily"}'));

-- Run a task now
CALL SETUP.GDC_TASK('execute', PARSE_JSON('{"name": "INGESTION_abc123def456"}'));

-- Suspend a task
CALL SETUP.GDC_TASK('suspend', PARSE_JSON('{"name": "INGESTION_abc123def456", "source_uri": "https://geo.example.com/wfs", "source_type": "WFS", "source_table_name": "protected_areas"}'));
```

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

---

## Ingested Data Schema

All datasets ingested by Geo Data Connector follow a consistent schema. Understanding this schema is essential for writing correct queries.

### Supported Data Source Types

| Type | Sources | Output |
|------|---------|--------|
| Vector | WFS, OGC API Features, OGC API Tiles | Feature table with GeoJSON geometry and attributes |
| Raster/Image | WMS, WMTS, WCS, OGC API Maps, OGC API Coverages | Pixel data table + metadata table |
| Catalog | CSW, OGC API Records | Discovery and browsing only (not ingestible) |

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

Every raster/image ingestion produces a companion metadata table named `<table_name>__metadata` with image properties including: `source_type`, `crs`, `width`, `height`, `bands`, `band_names`, `band_units`, `band_value_range`, `band_nodata`, `pixel_size_x`, `pixel_size_y`, `pixel_value_type`, `pixel_count`, `file_size`, `file_location`, and bounding box coordinates in both native CRS and WGS84.

### Image Stages (WMS, WMTS)

WMS and WMTS ingestions also produce Cloud Optimized GeoTIFF (COG) files in a Snowflake stage. WCS and OGC API Coverages deliver raw raster data and do not produce an image stage.

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

### Bounding Box Filter

Filter features within a geographic extent:

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
