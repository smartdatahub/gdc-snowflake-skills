# Geo Data Connector — Skills for AI Coding Assistants

Skills for [Geo Data Connector](https://app.snowflake.com/marketplace/listing/GZSYZ3BIGE/smart-data-hub-geo-data-connector), the Snowflake Native App for discovering and ingesting geospatial data from OGC web services into Snowflake tables.

These skills work with [Snowflake Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) and [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## Prerequisites

- Geo Data Connector installed in your Snowflake account (paid edition)
- SQL Procedures API feature enabled

## Installation

### Cortex Code CLI

```
/skill add https://github.com/smartdatahub/gdc-snowflake-skills.git
```

To update:

```
/skill sync
```

### Claude Code

```
/skill add https://github.com/smartdatahub/gdc-snowflake-skills.git
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `geo-data-connector` | SQL procedures, ingested data schema, and geospatial query patterns |

## What's Included

- **SQL Procedure Reference** — All 5 compound procedures (27 actions) with parameters and examples
- **Ingested Data Schema** — Vector tables, raster pixel tables, metadata tables, image stages
- **Geospatial Query Patterns** — Spatial joins, proximity search, H3 indexing, bounding box filters
- **Cortex AI Integration** — Translation, classification, extraction, and summarization examples

## Usage

Once installed, invoke the skill by name:

```
/geo-data-connector
```

Or let Cortex Code / Claude Code discover it automatically when you ask questions about Geo Data Connector procedures or ingested geospatial data.

## License

MIT
