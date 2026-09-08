# Integration Library Specification

This document specifies the integration library: the machine-readable format
in which data sources, their ingestion logic, and their outputs are described
for the Open Source Data Integrator Platform. It is the companion to the
*System Architecture* (section 3.1) and the *Interoperability and Standards
Mapping* (section 6), and is an independently reusable deliverable: a library
written to this specification can be consumed by any platform implementation
that conforms to it, and an implementation that conforms to it can run any
library written to it.

The specification has three parts: the **library layout** on disk, the
**integration definition** document, and the **job contract** that the
definition's code is executed against. The key words MUST, SHOULD, and MAY
are used as in RFC 2119. This document describes contract version 2, the
value carried in every definition's `contract` field.

## 1. Library layout

A library is a directory tree, versioned in ordinary source control:

```
library/
  integrations/<id>/definition.json     the definition (section 2)
  integrations/<id>/code.py | code.mjs  the integration code (section 3)
  skills/<id>/SKILL.md                  reusable know-how (section 5)
```

- Each integration directory holds exactly one `definition.json` and exactly
  one code file. The code file's extension MUST match the definition's
  `runtime` (`code.py` for `python3.12`, `code.mjs` for `node22`). On
  install, the file's contents become the definition's `code` field, which
  is therefore left empty (`""`) in the library copy.
- Directory names are the integration and skill ids and MUST match the
  identifier grammar of section 2.1.
- Library entries are templates. They SHOULD omit `aoi` so an installed
  copy inherits the deployment's area of interest, and their `trigger` and
  `limits` are starting values the deployment may change (section 4).
- Secret values MUST NOT appear anywhere in a library.

## 2. Integration definition

A definition is one JSON document (RFC 8259, UTF-8) describing everything
about an integration except secret values. It is the unit of portability:
the same document is installed from a library, exported from one deployment,
and imported into another.

```json
{
  "id": "worldpop_population",
  "name": { "_base": "en", "en": "WorldPop Population" },
  "description": { "_base": "en", "en": "Annual constrained population counts at about 100 m for the area of interest." },
  "source": {
    "name": { "_base": "en", "en": "WorldPop Global 2 constrained population counts" },
    "provider": "WorldPop, University of Southampton",
    "url": "https://hub.worldpop.org/",
    "license": "CC BY 4.0",
    "documentation": "https://data.worldpop.org/..."
  },
  "trigger": { "cron": "0 6 1 * *" },
  "runtime": "python3.12",
  "limits": { "cpus": 2, "memoryMb": 4096, "timeoutMs": 3600000 },
  "destinations": [
    {
      "kind": "raster",
      "layerId": "population",
      "name": { "_base": "en", "en": "Population" },
      "temporality": "time-series",
      "bands": [
        {
          "id": "population",
          "name": { "_base": "en", "en": "Population count" },
          "units": "people per grid cell",
          "nodata": -99999,
          "rendering": {
            "colorMap": [{ "value": 0, "color": "#ffffe5" }, { "value": 280, "color": "#005a32" }],
            "defaultRange": { "min": 0, "max": 280 },
            "resampling": "nearest"
          }
        }
      ]
    }
  ],
  "code": "",
  "contract": 2
}
```

### 2.1 Identifiers

Every identifier in a definition (integration `id`, input ids, `tableId`,
column ids, `layerId`, band ids, skill ids) MUST match:

```
^[a-z][a-z0-9_]{0,62}$
```

Identifiers are permanent. Renaming an integration is creating a new one;
table and layer ids appear in query, tile, and download URLs and in the
relational schema.

### 2.2 Localized strings

Human-facing text is a **localized string**: an object whose `_base` key
names the locale the text was authored in, and whose other keys are locale
codes mapping to text. The base locale MUST be present as a key.

```json
{ "_base": "en", "en": "Population count", "fr": "Nombre d'habitants" }
```

Consumers resolve a localized string by the requested locale, falling back
to the `_base` locale. Locale codes are BCP 47 language tags (in practice
two-letter ISO 639-1 codes).

### 2.3 Top-level fields

| Field | Type | Required | Meaning |
|---|---|---|---|
| `id` | identifier | yes | Published identifier of the integration |
| `name` | localized string | yes | Display name |
| `description` | localized string | no | What the integration does and any caveats |
| `source` | object (2.4) | no | Provenance of the external source; absent for derived integrations |
| `trigger` | object (2.5) | yes | When runs happen |
| `secrets` | string[] | no | Names of credentials the code reads; values are never in the definition |
| `aoi` | object (2.6), `null`, or absent | no | Area of interest, tri-state |
| `inputs` | map of id to input reference (2.7) | no | Platform data the run consumes |
| `runtime` | `"node22"` or `"python3.12"` | yes | Runtime image the code executes in |
| `limits` | object (2.8) | no | Per-run resource limits |
| `destinations` | array (2.9) | yes | Tables and raster layers the integration writes |
| `validation` | object (2.10) | no | Data quality rules evaluated on each run |
| `code` | string | yes | The single self-contained code file |
| `contract` | `2` | yes | Job contract version |

Field names not listed here are reserved. Validators SHOULD reject unknown
fields so that a definition means the same thing to every implementation.

### 2.4 Source

Documentation and lineage only; nothing in `source` affects behavior.

| Field | Type | Meaning |
|---|---|---|
| `name` | localized string | Name of the dataset or service |
| `url` | string | Primary URL of the source or its API |
| `provider` | string | Organization publishing the data |
| `license` | string | License of the source data, as an SPDX identifier where one exists |
| `documentation` | string | Link to source documentation |
| `limitations` | localized string | Caveats a data consumer should see: accuracy, coverage, timeliness, interpretation |

### 2.5 Trigger

| Field | Type | Meaning |
|---|---|---|
| `cron` | string | Five-field POSIX cron expression, evaluated in UTC |
| `webhook` | boolean | Accept pushed events on the integration's capability URL |
| `webhookDelaySeconds` | integer 0 to 86400 | Batch window: the first event schedules a run this far in the future and later events join it; default 0 (immediate) |

A trigger MAY declare both `cron` and `webhook`. A definition with neither
can only be run manually.

### 2.6 Area of interest

| Field | Type | Meaning |
|---|---|---|
| `bbox` | `[west, south, east, north]` | Geographic bounding box in WGS 84 (EPSG:4326) decimal degrees; `west > east` crosses the antimeridian |
| `countries` | string[] | ISO 3166-1 alpha-3 codes covered by the box, a hint for sources keyed by country rather than extent |

`aoi` is tri-state. An object clips ingestion and input extracts to the box.
An explicit `null` means the source's full extent. An absent field inherits
the deployment's default area of interest at run time. The platform resolves
this before launch: the job's copy of the definition carries either a
concrete `aoi` or no `aoi` field at all (full extent), never `null`.

### 2.7 Inputs

`inputs` maps an input id to a reference to platform data. Before a run, the
platform resolves each reference to concrete versions, materializes them
read-only under `/in`, and records the resolved versions on the run. An
input MUST NOT reference one of the definition's own destinations.

**Raster input**

| Field | Type | Meaning |
|---|---|---|
| `kind` | `"raster"` | |
| `layerId` | identifier | Catalog layer to read |
| `select` | `"current"`, `{ "latest": n }`, or `{ "from": iso, "to"?: iso }` | Which snapshots to materialize |

**Table input**

| Field | Type | Meaning |
|---|---|---|
| `kind` | `"table"` | |
| `tableId` | identifier | Table to extract from |
| `columns` | identifier[] | Columns to include; absent means all |
| `where` | filter (below) | Row filter applied before extraction |
| `format` | `"gpkg"`, `"json"`, or `"csv"` | Extract file format; defaults to `gpkg` for `python3.12` and `json` for `node22` |

A filter is an array of clauses combined with AND, each
`{ "column": id, "op": "=" | "!=" | ">" | ">=" | "<" | "<=" | "in", "value": any }`
(`in` takes an array). Extracts are select, filter, and clip only, never
aggregation, and implementations MAY cap their size.

### 2.8 Limits

| Field | Type | Meaning |
|---|---|---|
| `cpus` | number | CPU cores |
| `memoryMb` | integer | Memory ceiling in megabytes |
| `timeoutMs` | integer | Wall-clock timeout in milliseconds |
| `workDiskMb` | integer | Quota for the persistent `/work` workspace, checked between runs |

Declared limits are requests; the deployment caps them.

### 2.9 Destinations

A destination is a table or a raster layer. Each is created and owned by
the installing integration.

**Table destination**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `kind` | `"table"` | yes | |
| `tableId` | identifier | yes | Table name in the data schema |
| `name` | localized string | no | Display name |
| `description` | localized string | no | One-sentence summary shown in the data catalog |
| `action` | `"replace"`, `"append"`, or `"merge"` | yes | How each run's rows are applied |
| `mergeColumns` | identifier[] | when `merge` | Key columns identifying a row; each MUST be a declared column |
| `columns` | column[] | yes | Schema of the table |

A column is `{ id, name?, type, enumValues?, units? }` with `type` one of:

| Type | Semantics |
|---|---|
| `text` | Unicode string |
| `number` | Floating-point number |
| `integer` | Exact 64-bit integer |
| `boolean` | true or false |
| `date` | Calendar date, ISO 8601 `YYYY-MM-DD` |
| `datetime` | Instant, ISO 8601 with offset, stored in UTC |
| `geometry` | GeoJSON geometry object in WGS 84 |
| `enum` | Text drawn from `enumValues`, the harmonization target for source codings |
| `json` | Arbitrary JSON |

`enumValues` is an array of `{ id, name }` (name a localized string).
Implementations SHOULD flag runs that deliver values outside the vocabulary.
`units` is free text (for example `mm/day`).

`replace` truncates the table and loads the run's rows; `append` adds them;
`merge` upserts on `mergeColumns`. Every row additionally records the run
that produced it, which is the lineage link.

**Raster layer destination**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `kind` | `"raster"` | yes | |
| `layerId` | identifier | yes | Catalog layer; appears in tile and API URLs |
| `name` | localized string | no | Display name |
| `description` | string | no | One-sentence summary shown in the data catalog |
| `temporality` | `"time-series"` or `"current-only"` | yes | Whether promoted snapshots are retained as history or superseded |
| `bands` | band[] | yes | Bands in file band order |

A band is `{ id, name?, units?, nodata?, rendering? }`. `nodata` is the
sentinel value written in the files; absent means the file's own nodata or
NaN. `rendering` is `{ colorMap?, defaultRange?, resampling? }`:

| Field | Type | Meaning |
|---|---|---|
| `colorMap` | `[{ value, color }]` | Value-to-color stops (CSS hex colors), interpolated between stops |
| `defaultRange` | `{ min, max }` | Default value range for display scaling |
| `resampling` | `"nearest"` or `"average"` | Overview resampling; `nearest` avoids inflating sparse data |

### 2.10 Validation

| Field | Type | Meaning |
|---|---|---|
| `expectedCadence` | ISO 8601 duration (for example `P1D`) | Expected data cadence; freshness is measured against it |
| `ranges` | `[{ destinationId, field, min?, max? }]` | Allowed value range for a column (`field` is a column id) or band (`field` is a band id) of the named destination |

Violations flag the run; they do not reject its data.

### 2.11 Structural rules

A validator MUST reject a definition when:

- any identifier violates the grammar of 2.1;
- `aoi` is an object whose `bbox` is malformed;
- `trigger.cron` is not a valid five-field expression, or
  `webhookDelaySeconds` is outside 0 to 86400;
- a table destination's `action` is not one of the three values, a `merge`
  destination lacks `mergeColumns`, or a merge column is not a declared
  column;
- a column `type` is outside the vocabulary of 2.9;
- an input references one of the definition's own destinations;
- a human-facing text field is a plain string where a localized string is
  required.

## 3. Job contract

Integration code runs as a job inside an isolated container built from the
declared runtime image. The contract between the platform and the job is
language-agnostic: a JSON payload on standard input, a JSON result on
standard output, logs on standard error, and three mount points. A
per-language **harness** inside the image maps this onto a single entrypoint
function, so integration code is a plain module with no platform-specific
imports.

### 3.1 Payload (stdin)

```json
{
  "definition": { ...the definition, with aoi resolved... },
  "secrets": { "name": "value" },
  "previousState": { ... } | null,
  "webhookData": [ ...pending events, oldest first... ] | null,
  "params": { ... }
}
```

- `definition` is the installed definition with `aoi` resolved to the
  effective area of interest (2.6). It includes `code`, which the harness
  loads from there.
- `secrets` holds the values for the names declared in `definition.secrets`.
- `previousState` is the `state` returned by the last successful run, or
  null on the first run and after a cold start.
- `webhookData` is every buffered event since the last run, and absent or
  null for runs not triggered by webhooks.
- `params` are free-form parameters supplied by the trigger, an empty
  object for scheduled runs.

### 3.2 Entrypoint

The harness exposes the payload as a context object and calls the
integration's entrypoint with it:

| Runtime | Entrypoint | Context fields |
|---|---|---|
| `python3.12` | module-level `def run(ctx)` | `ctx.definition`, `ctx.secrets`, `ctx.previous_state`, `ctx.webhook_data`, `ctx.params`, `ctx.in_dir`, `ctx.work_dir`, `ctx.out_dir` |
| `node22` | default-exported `async function run(ctx)` | `ctx.definition`, `ctx.secrets`, `ctx.previousState`, `ctx.webhookData`, `ctx.params`, `ctx.inDir`, `ctx.workDir`, `ctx.outDir` |

The entrypoint returns (or resolves to) a run result (3.5). A nonzero exit
or an uncaught exception fails the run, and the tail of standard error
becomes the run's error message. Standard output is reserved for the result:
the harness redirects the code's own prints and any child-process output to
standard error.

### 3.3 Mounts

| Path | Access | Contents |
|---|---|---|
| `/in` | read-only | Materialized inputs plus `/in/manifest.json` (3.4); absent when the definition declares no inputs |
| `/work` | read-write, persistent per integration | Cache workspace; may be wiped at any time, so correctness MUST NOT depend on it. Subject to `limits.workDiskMb` |
| `/out` | read-write, per run | Where the job writes the files it references in its result |

The container's root filesystem is read-only, so any other write MUST go to
`/work` or `/out`. Curated libraries are part of the runtime image; jobs
cannot install packages at run time. Network egress is limited to the public
internet.

### 3.4 Input manifest

`/in/manifest.json` describes what was materialized, keyed by input id:

```json
{
  "inputs": {
    "rain": {
      "kind": "raster",
      "layerId": "chirps_monthly",
      "snapshots": [
        { "path": "rain/2026-06-01T00:00:00Z.tif", "snapshotId": "…", "timestamp": "2026-06-01T00:00:00Z", "bands": ["precip"] }
      ]
    },
    "sites": { "kind": "table", "tableId": "water_points", "path": "sites.gpkg", "format": "gpkg", "rowCount": 1240 }
  }
}
```

- Raster snapshots are listed ascending by `timestamp`, each with a `path`
  relative to `/in` (null for an empty snapshot), the catalog `snapshotId`
  recorded for lineage, `timestamp`, optional `timestampEnd` for periods,
  and `bands` in file band order. A multi-file snapshot is presented as one
  openable virtual raster.
- Table extracts are one file in the requested `format`. In `json`, dates
  are ISO 8601 strings and geometry columns are GeoJSON geometry objects.

The manifest is authoritative about paths; code SHOULD read it rather than
assume a layout.

### 3.5 Run result (stdout)

```json
{
  "data": { "tableId": [ { "column": value, ... } ] },
  "tables": [ { "tableId": "…", "path": "…", "format": "geojson" | "fgb" | "gpkg" | "csv" } ],
  "rasters": [ { "layerId": "…", "path": "…", "timestamp": "…" } ],
  "state": { ... }
}
```

Every field is optional; an empty object is a successful run that changed
nothing.

- **`data`**: rows per destination table id, inline. Values follow the
  column types of 2.9 (geometry as GeoJSON geometry objects, dates and
  instants as ISO 8601 strings). Inline rows are bounded by the platform's
  standard-output cap; larger tables go through `tables`.
- **`tables`**: table files written under `/out`, loaded into their
  destination table. `format` defaults to the file extension. Fields are
  matched to declared columns by name, plus the file's geometry for a
  declared geometry column, and cast to the declared types. A table id MUST
  appear in `data` or `tables`, never both.
- **`rasters`**: one entry per snapshot produced. Fields: `layerId`; exactly
  one of `path` (single file) or `sources` (array of `{ path, footprint? }`
  making up one observation, mosaicked at read time; an empty array
  publishes an empty snapshot that supersedes the previous one); `timestamp`
  and optional `timestampEnd`; optional `footprint`
  (`[west, south, east, north]` in WGS 84) and `crs` (authority code such as
  `EPSG:4326`), both derived from the files when absent; optional free-form
  `metadata`. Files MUST be Cloud-Optimized GeoTIFFs whose band order
  matches the destination's `bands` and whose nodata matches the declared
  sentinel. Omitting a layer from `rasters` means no update to it.
- **`state`**: the next incremental state, persisted and delivered as
  `previousState` to the next run. This is the only authoritative
  incremental state; it is what makes ingestion resumable after outages.

The platform, not the job, applies the result: it validates rows against the
declared schema, applies tabular changes transactionally, promotes raster
files into the store, registers them in the catalog, and records volumes,
lineage, and validation flags on the run.

### 3.6 Runtime images

Two runtime images are specified: `node22`, a JavaScript runtime, and
`python3.12`, which additionally carries an open geospatial stack (raster,
array, and vector libraries, and the command-line raster tools the library's
integrations rely on). An implementation MUST provide both and MAY add
further runtimes as additional images implementing the same harness
contract; shared contract tests verify a harness against sections 3.1 to
3.5.

## 4. Lifecycle semantics

- **Install** copies a library entry into the deployment, where the copy
  becomes the integration's single specification. There is no separate
  override layer.
- **Versions** are assigned by the deployment on install and on every
  subsequent change; each version is archived with attribution. Reinstalling
  an unchanged definition is a no-op.
- **Library updates** replace the installed definition but carry over the
  deployment-owned fields: `aoi`, `trigger`, and `limits`.
- **Export** produces the definition document as-is. **Import** installs it
  paused, with `aoi` removed so the importer's default area of interest
  applies, and prompts for any declared `secrets` before the first run.
- **Secrets** are stored by the deployment per integration, encrypted at
  rest, and injected only into that integration's runs.

## 5. Skills

A skill is prose know-how about a source or technique, independent of any
particular integration, kept beside the definitions so that an AI designer
or a human developer can load it on demand. `skills/<id>/SKILL.md` is
Markdown with YAML front matter:

```
---
name: WorldPop population source
description: How WorldPop publishes gridded population: country-keyed files, release layout, encoding pitfalls
verified: 2026-08-28
---
```

`name` and `description` are required; `description` is a one-line answer
to "when would I read this", used to index skills for on-demand loading.
`verified` is the date the content was last confirmed against the live
source. Skills are documents to read, never code to execute.

## 6. Conformance

A **library** conforms when every definition validates under section 2, the
code file matches the declared runtime, and no secret value appears.

A **platform implementation** conforms when it:

1. accepts and validates definitions per section 2, rejecting the
   violations of 2.11;
2. executes code under the job contract of section 3, providing both
   runtime images and the manifest and mounts as specified;
3. applies run results per 3.5, including transactional tabular loads,
   COG promotion and cataloging, and persistent incremental state;
4. implements the lifecycle semantics of section 4;
5. serves skills to its design tooling as read-only documents.

An implementation meeting these criteria runs every entry of a conforming
library unchanged, and definitions exported from it import into any other
conforming implementation.
