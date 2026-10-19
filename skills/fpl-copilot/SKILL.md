---
name: fpl-copilot
version: 1.3.0
description: "Use when the user asks about Fantasy Premier League player stats, transfer advice, captain picks, fixture difficulty, gameweek strategy, or saved squad management."
---

# FPL Copilot

Analyze current FPL data, maintain saved squads, and produce self-contained reports. This skill specifies operations and completion criteria; choose suitable tools or commands from the actual environment.

## Workflow

### 1. Establish capabilities and storage

Inspect the available tools and locate this skill’s resources relative to its installed location. Identify capabilities for HTTP requests, JSON processing, structured-data queries and writes, document persistence, and report delivery. On a shell-equipped host, infer suitable installed commands or scripting libraries; on other hosts, use exposed native tools. Tool names and executables are implementation choices.

Use an existing FPL data store when present. On desktop, the default persistent root is `~/.fplcopilot/`, containing `fplcopilot.db`, `squads/`, and `reports/`. On other hosts, resolve equivalent durable locations or resource handles. Reuse that mapping throughout the task and state whether saved squads are available across conversations. Keep FPL data separate from the agent application's internal storage.

Use `references/schema.sql` as the data contract for a new store. SQL examples in the references describe queries, not a required command-line interface. A non-SQL store must preserve the same fields, keys, types, and calculation semantics. If a required capability is missing, identify the blocked operation and complete only the supported parts; distinguish unsaved output and unavailable live data explicitly.

**Complete when:** the agent has identified working operations and persistent locations for the requested task, or reported a specific missing capability.

### 2. Establish data freshness

Before data analysis, read `references/sync.md` and the relevant mappings in `references/api.md`. Inspect successful-refresh metadata and refresh missing or stale datasets according to that contract. Fetch only the player histories required for the requested calculation; a full-player batch requires an explicit user request.

**Complete when:** required datasets have successful refresh timestamps and verified persisted records, or the answer can identify the age and limitations of a usable cached snapshot. An unsuccessful refresh is not evidence of fresh data.

### 3. Load squads and analyze

For “my squad/team,” check saved squads before asking the user to share again. Read `references/squad.md` for selection, persistence, and validation. Read `references/analysis.md` for metrics and decision frameworks; use the documented formulas with the available query/calculation capabilities. Resolve player IDs by stored names, checking ambiguity before selecting a player.

Save newly shared squads and confirmed changes using the squad reference. Keep hypothetical transfers in draft plans; recording a change locally does not submit it to the user's FPL account. After squad analysis, persist key takeaways in Notes.

**Complete when:** recommendations are backed by retrieved data and documented calculations, and every required squad write is verified or explicitly reported as unsaved.

### 4. Deliver the result

Use Markdown for quick lookups, short reasoning, deadlines, and small results. For the spatial or comparative outputs below, read `references/html-output.md` and adapt the matching template with actual data.

| Output | Template |
|--------|----------|
| Squad formation, bench, totals | `templates/squad-view.html` |
| Fixture difficulty matrix | `templates/fixture-matrix.html` |
| Transfer comparison | `templates/transfer-comparison.html` |
| Captain ranking with reasoning | `templates/captain-ranking.html` |
| Gameweek strategy | `templates/gameweek-report.html` |

Create one self-contained HTML report with inline CSS, JavaScript, and SVG; use vanilla HTML/CSS/JS and responsive layouts. Store it under the resolved `reports/` location as `{YYYY-MM-DD}-{slug}.html`. Provide the supported preview, attachment, share link, or accessible file path. Infer an appropriate viewing operation when available. When report creation or delivery is unavailable, provide the substantive result in chat and identify the missing report capability.

**Complete when:** the user has a substantive answer and an accessible report reference where supported, with data freshness and any limitations stated.

## Reference routing

Read each resource before performing its operation:

| Resource | Operation |
|----------|-----------|
| `references/sync.md` | Refresh, normalize, persist, and verify FPL datasets |
| `references/api.md` | Resolve endpoints and API-to-storage field mappings |
| `references/schema.sql` | Initialize storage or resolve tables, types, and keys |
| `references/analysis.md` | Compute metrics and recommendations |
| `references/squad.md` | Load, validate, create, or update squads |
| `references/html-output.md` | Generate and deliver reports |

## Data conventions

- Resolve names against `web_name`, then partial display names and full names; clarify ambiguous matches.
- Prices use 0.1m units: `130` means £13.0m. Display as `£X.Xm`; normalize by cost in millions for value comparisons.
- Positions are GKP, DEF, MID, FWD, mapped from API codes 1–4.
- Availability codes: `a` available, `d` doubtful, `i` injured, `s` suspended, `u` unavailable.
- FDR runs from 1 (very easy) to 5 (very hard).
