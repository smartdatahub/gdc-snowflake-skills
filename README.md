# Geo Data Connector — Skills for AI Coding Assistants

Skills for <a href="https://www.smartdatahub.io/gdc" target="_blank">Geo Data Connector</a>, the Snowflake Native App for discovering and ingesting geospatial data from OGC web services into Snowflake tables.

These skills work with <a href="https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code" target="_blank">Snowflake Cortex Code</a> and <a href="https://docs.anthropic.com/en/docs/claude-code" target="_blank">Claude Code</a>.

## Prerequisites

- Geo Data Connector installed in your Snowflake account (paid edition)
- SQL Procedures API feature enabled

## Installation

### Cortex Code CLI

Run the following command in your **terminal** (not inside the Cortex Code session):

```bash
cortex skill add "https://github.com/smartdatahub/gdc-snowflake-skills.git"
```

> **Note:** The `/skill add` slash command inside a Cortex Code session does not support remote URLs. Use the terminal command above instead.

### Cortex Code in Snowsight

Cortex Code in Snowsight does not support remote skill installation. To add the skill manually:

1. Download [`geo-data-connector/SKILL.md`](geo-data-connector/SKILL.md) from this repository
2. In Snowsight, open Cortex Code and press `/skill`
3. Press `a` to add a skill, then upload the downloaded file

The skill is scoped to your current workspace and must be re-added for each workspace.

### Claude Code

```
/skill add https://github.com/smartdatahub/gdc-snowflake-skills.git
```

### Preview Channel

To track upcoming changes before they reach the stable release:

```bash
# Cortex Code CLI (terminal)
cortex skill add "https://github.com/smartdatahub/gdc-snowflake-skills/tree/preview/geo-data-connector"

# Claude Code (session or terminal)
npx skills add https://github.com/smartdatahub/gdc-snowflake-skills/tree/preview/geo-data-connector
```

Preview skills may reference procedures or features not yet available in the current Marketplace release.

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
