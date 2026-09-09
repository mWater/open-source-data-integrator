# Data Integrator Interoperability and Standards Mapping

This document is the companion to the *System Architecture*. Where the
architecture specifies components by responsibility and contract, this
document specifies the standards those contracts conform to and the concrete
paths by which national systems, desktop tools, and other platforms exchange
data with a deployment.

The mapping follows two rules. First, it claims conformance only where the
platform implements the standard, and says so plainly where an interface is a
documented convention rather than a ratified standard. Second, every
interface is specified at the format and protocol level, so a conforming
alternative implementation interoperates with the same clients and sources
without sharing any code with the reference implementation.

## 1. Standards summary

| Interface | Standard or convention | Role |
|---|---|---|
| Tabular query results | JSON (RFC 8259), CSV (RFC 4180), GeoJSON (RFC 7946) | Data exchange to clients and national systems |
| Raster snapshots | Cloud-Optimized GeoTIFF (OGC GeoTIFF 1.1; COG layout) | Raster exchange; direct use in GIS tools |
| Raster map tiles | XYZ tile scheme over PNG (Web Mercator) | Embedding maps in any web GIS client |
| Vector map tiles | Mapbox Vector Tile 2.1 | Dashboard and table map rendering |
| Coordinate references | EPSG registry codes; WGS 84 (EPSG:4326); Web Mercator (EPSG:3857) | Spatial harmonization |
| Timestamps | ISO 8601 / RFC 3339, always UTC | Temporal harmonization |
| Schedules | POSIX/Vixie five-field cron expressions, UTC | Trigger declaration |
| Query API | HTTP/1.1 over TLS; REST described by OpenAPI 3 | Programmatic access contract |
| API authentication | Bearer tokens in the HTTP Authorization header | Token-based programmatic access |
| Push ingestion | HTTP POST of JSON to capability URLs | Sensor and event feeds into the platform |
| Analytical SQL | ISO/IEC 9075 SQL with OGC Simple Features functions (SQL/MM spatial) | Zonal statistics, dashboards, assistant queries |
| Geometry interchange | GeoJSON on the way in and out; WKB in the tabular plane | Vector data harmonization |
| Integration definitions | JSON documents against a published schema | Portable pipeline specifications |
| Licensing | SPDX license identifiers; Apache 2.0 for the platform | Open-source compliance and attribution |
| Container images | OCI image format | Runtime images for the sandbox |

The sections below give the normative detail per area.

## 2. Data formats

### 2.1 Tabular data

Destination tables live in a relational database with spatial types.
Externally, every table is readable through the query API in three
serializations:

- **JSON** (RFC 8259): objects keyed by the declared column ids; the default
  for programmatic consumers.
- **CSV** (RFC 4180): for spreadsheets and statistical tools.
- **GeoJSON** (RFC 7946): a FeatureCollection for tables with a geometry
  column, with non-spatial columns as feature properties; loadable directly
  by QGIS, ArcGIS, web mapping libraries, and spatial ETL tools.

Geometry columns accept GeoJSON on ingestion and are stored in WGS 84.
Column types are drawn from a small declared vocabulary (text, number,
integer, boolean, date, datetime, geometry, enum, json) that maps onto
standard SQL types, so the schema of every destination table is portable.

### 2.2 Raster data

The raster plane stores every promoted snapshot as a **Cloud-Optimized
GeoTIFF**: a GeoTIFF conforming to OGC GeoTIFF 1.1 whose internal layout
(tiling, overviews, header ordering) supports efficient HTTP range reads.
Consequences:

- Any GDAL-based tool (QGIS, ArcGIS, rasterio, R terra) opens a downloaded
  snapshot directly; no export step or proprietary reader exists.
- Bands declare units and a nodata value, both recorded in the file and in
  the catalog, so downstream statistics are computed correctly by tools that
  never talk to the platform.
- Each snapshot records its CRS as an authority code (EPSG, or another
  authority known to the spatial registry) and keeps its native grid;
  nothing is resampled at ingest.

Source-side, integrations commonly consume NetCDF (including CF-convention
files), GRIB2, HDF, GeoPackage, shapefile archives, and plain GeoTIFF; the
runtime images carry an open geospatial stack that reads them.
Harmonization to the storage formats above happens inside the integration,
so upstream format diversity never reaches consumers.

### 2.3 Map tiles

- **Raster tiles** are 256-pixel PNG tiles in the XYZ addressing scheme in
  Web Mercator, the convention consumed by every mainstream web mapping
  client (MapLibre, Leaflet, OpenLayers among them). Tile URLs pin the
  snapshot id, making them immutable and safely cacheable. Drawing is
  clipped to the layer's area-of-interest shape; a query parameter opts out
  and draws everything the files cover, and a short key of the shape rides
  in the URL so a reshaped area is never served from cache.
- **Vector tiles** are Mapbox Vector Tile 2.1 protocol buffers, produced for
  dashboard map queries and for direct map views of destination tables with
  geometry.

Tiles are a rendering interface; the data interchange interfaces are the
formats in 2.1 and 2.2.

## 3. Spatial and temporal harmonization

- **CRS.** Tabular geometries are stored in WGS 84 (EPSG:4326). Rasters
  retain their native CRS, identified by authority code against the
  deployment's spatial reference registry; map rendering warps to Web
  Mercator (EPSG:3857) at tile time. Areas of interest are geographic
  bounding boxes in WGS 84, ordered west, south, east, north, optionally
  naming ISO 3166-1 alpha-3 countries.
- **Boundaries.** An area naming countries is also resolved to a land-only
  shape from a bundled boundary dataset (Natural Earth 1:10m Admin 0
  countries with lakes removed, public domain), cut to the box, with the
  dataset and version recorded as provenance on every statistic taken over
  it. Deployments needing an official national outline can substitute one
  at the same point; the statistics and tile interfaces do not change.
- **Time.** All timestamps are ISO 8601 in UTC, including raster snapshot
  observation times (instants or explicit ranges) and run records. Cron
  triggers evaluate in UTC.
- **Units.** Every raster band and numeric column declares its units;
  library entries use SI or the source discipline's canonical unit (for
  example cubic meters per second for discharge), recorded in the catalog
  and surfaced through the API.
- **Missing data.** Every band declares its nodata sentinel; the analytical
  SQL surface and the tile renderer honor it, so sentinels never contaminate
  statistics or maps.

## 4. Access interfaces

### 4.1 Query API

The query API is the contractual programmatic interface: REST over HTTPS,
described by an OpenAPI 3 document the platform serves about itself. The
description is machine-readable, so client generation and API gateways work
against a live deployment. Authentication is a personal access token
presented as an HTTP Bearer credential; browser sessions use cookies. All
routes enforce server-side role checks (admin, editor, viewer).

The surface covers: listing tables and layers with their schemas and
metadata; filtered, paginated reads of any destination table in the three
serializations of 2.1; the raster catalog with per-snapshot metadata; and
direct snapshot downloads (2.2). The API is fully functional with no AI
model configured.

### 4.2 Webhooks (push ingestion)

Push-capable sources deliver events by HTTP POST of a JSON body to a
per-integration capability URL: an unguessable URL that is itself the
credential, rotatable by an administrator. Payload conventions are
deliberately liberal: a single JSON object, an array of objects, or an
`{"events": [...]}` wrapper are all accepted; malformed events are rejected
individually and visibly without failing the batch. Delivery is
at-least-once from the sender's perspective; integrations declare merge keys
so redelivery updates rather than duplicates. This is the integration path
for sensor networks and event-emitting national systems that push rather
than serve an API.

### 4.3 Analytical SQL

Dashboards, the assistant, and raster-vector statistics execute standard SQL
with OGC Simple Features / SQL-MM spatial functions under a read-only role
scoped to destination data and the catalog. Promoted raster snapshots are
registered as out-of-database rasters in this surface, so zonal statistics
and point sampling run in SQL without pixels entering the database. The
area-of-interest shape is a table of this surface, and a function returns
area-weighted statistics of any layer over it (mean, share meeting a
condition, coverage by valid data, boundary provenance), the standard way
to answer "what share of the country" questions. For national analysts,
this is a familiar, standard interface: any analyst or BI tool fluent in
spatial SQL can be granted the same read-only role.

## 5. Ingestion-side protocols

Integrations reach sources over the public internet from inside the sandbox
(private and metadata address ranges are blocked). The library's entries
demonstrate the common protocol families:

| Source family | Protocols and formats |
|---|---|
| Satellite and gridded product archives | HTTPS file retrieval (GeoTIFF, NetCDF, GRIB2, zipped vector archives), directory or manifest discovery |
| Asynchronous retrieval APIs | Job-based request/poll/download APIs (for example the Copernicus data store family), with authenticated requests and queued fulfilment |
| Catalog and metadata endpoints | JSON catalog and constraint documents published alongside datasets |
| Sensor networks | Webhook push (4.2) |
| National databases and MIS | HTTPS APIs where offered; file exchange (CSV, GeoJSON, GeoPackage) where not |

Credentials for authenticated sources are named in the definition, supplied
by the deployment, encrypted at rest, and injected into the sandboxed job
only at run time; they never appear in definitions, library entries, logs,
or images.

## 6. Portability formats

- **Integration definitions** are single JSON documents against a published
  schema (the *Integration Library Specification*). A definition exported
  from one deployment imports into another, arriving paused for review with
  the importer's area of interest applying. This is the unit of
  pipeline-level interoperability between deployments and between
  implementations.
- **Library entries** are directories holding a definition plus prose
  documentation and per-source know-how, consumable by any implementation of
  the specification.
- **Deployment data** is exportable losslessly through standard formats:
  tables as CSV or GeoJSON, rasters as the COG files themselves.

## 7. National-system integration paths

The concrete conventions, by direction and need:

| Need | Path | Format / protocol |
|---|---|---|
| National dashboard or portal shows platform maps | Embed tile URLs | XYZ PNG tiles (2.3) |
| National GIS analyzes platform data | Query API or snapshot download | GeoJSON, CSV, COG |
| Statistical office ingests time series | Query API, filtered and paginated | JSON or CSV |
| Sensor network feeds the platform | Webhook push | JSON to capability URL (4.2) |
| WASH MIS or hydrological service feeds the platform | Pull integration against its API, or scheduled file exchange | Source's API; CSV/GeoJSON/GeoPackage |
| National system needs derived analytics | Derived integration over `inputs`; results exposed like any table or layer | Same as above |
| Analyst needs ad-hoc spatial queries | Read-only analytical SQL role | SQL with spatial functions (4.3) |
| Another deployment needs a working pipeline | Definition export/import | Definition JSON (6) |

Two properties make these paths durable. Interfaces are pinned and
versioned: tile and download URLs pin immutable snapshots, definitions carry
attributed version history, and the API is described by a served OpenAPI
document. And every path works offline-tolerantly: pulls resume from
recorded incremental state after connectivity loss, and pushed events buffer
across pauses.

## 8. Open-source and Digital Public Goods alignment

The platform is built to satisfy the Open Source Definition and the Digital
Public Goods Standard. The mapping onto the DPG Standard's indicators:

| DPG indicator | How the platform meets it |
|---|---|
| Relevance to SDGs | Directly supports SDG 6 (water and sanitation) and SDG 13 (climate) monitoring and early warning |
| Open licensing | Apache 2.0 for all platform code; OSI-approved throughout |
| Clear ownership | Repository and license state ownership and provenance |
| Platform independence | Self-hosted on commodity Linux; no mandatory vendor account or hosted service; the AI model is a replaceable, optional dependency with a documented capability profile, satisfiable by open-weights models |
| Documentation | Architecture, standards mapping, deployment and API documentation ship in the repository; the API self-describes via OpenAPI |
| Mechanism for data extraction | Complete data egress through standard formats (CSV, GeoJSON, COG, SQL); no lock-in formats exist |
| Privacy and applicable law | The platform stores environmental and infrastructure data; personal data is out of scope by design |
| Standards and best practices | This document; dependency licensing verified with SPDX identifiers and an SBOM in CI |
| Do no harm by design | Role-gated access, sandboxed untrusted code, encrypted secrets, no default credentials |

Dependency hygiene is continuous: the build verifies that every runtime
dependency carries an OSI-approved license (recorded by SPDX identifier) and
produces a software bill of materials, so a procuring institution can audit
the supply chain of any release.

## 9. Conformance for alternative implementations

An implementation is interoperable with this ecosystem when it:

1. consumes and produces integration definitions per the *Integration
   Library Specification*;
2. implements the job contract (stdin/stdout payloads, `/in`, `/work`,
   `/out` mounts) so library code runs unchanged;
3. stores rasters as COGs with cataloged CRS, units, nodata, and observation
   times, and serves tabular data in the serializations of 2.1;
4. exposes a query API described by a served OpenAPI document with bearer
   authentication and the role model of 4.1;
5. accepts webhook payloads per 4.2 on rotatable capability URLs.

Everything else (server language, database engine, container sandbox, tile
renderer, AI provider) is an implementation choice, substitutable without
breaking any client, source, or shared library entry.
