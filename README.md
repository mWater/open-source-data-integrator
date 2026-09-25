# Open Source Data Integrator Platform

The Data Integrator is an open-source, self-hosted platform that ingests multi-source
water and climate data (satellite and gridded products, in-situ sensors,
national databases), harmonizes it into standard formats, and exposes it
through a documented query API, a dashboard, and an optional AI assistant.
An agentic AI capability designs, tests, and maintains the ingestion
pipelines themselves.

It is developed by [mWater](https://www.mwater.co) for the UNICEF WASH
Innovation Hub as the reference implementation of the **Open Source Data
Integrator Platform** architecture, with initial deployments in Angola and Madagascar.

## Architecture documents

The architecture is specified independently of this implementation. It
describes components by responsibility and contract so that any vendor can
implement it with their own technology choices; the reference
implementation is one set of choices that satisfies those contracts.

| Document | Contents |
|---|---|
| [System Architecture](docs/System%20Architecture.md) | Design principles, system context, components, data flows, security and deployment architecture, extensibility |
| [Interoperability and Standards Mapping](docs/Interoperability%20and%20Standards%20Mapping.md) | The standards each interface conforms to, integration paths for national systems, Digital Public Goods alignment, conformance criteria |
| [Integration Library Specification](docs/Integration%20Library%20Specification.md) | The machine-readable definition format for data sources, the job contract integration code runs against, and library conformance |

Diagrams are Mermaid and render directly on GitHub.

## Status

This repository currently holds the architecture documents. The source code
of the reference implementation, the integration library, and deployment
documentation are published here with the proof-of-concept deliverable.

## License

Apache License 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
