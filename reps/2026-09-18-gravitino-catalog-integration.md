# Proposal: Apache Gravitino catalog integration for Ray Data

## Summary

This proposal would make Apache Gravitino a Ray Data catalog provider for organizations that manage their Hadoop and Spark data estate in Gravitino. Ray jobs would resolve table names and access paths through Gravitino's own metadata and services.

The initial implementation targets existing Iceberg table reads and writes through Gravitino Iceberg REST, plus registered external Delta table reads through Gravitino's Generic Lakehouse Catalog. The provider uses Ray Data's existing Catalog interface and format-specific readers and writers.

## General Motivation

For many Hadoop and Spark-centered enterprises, particularly in China, the relevant starting point is an existing Hive and Iceberg catalog estate. Apache Gravitino is designed to manage these catalogs and provides a [Spark connector](https://gravitino.apache.org/docs/1.3.0/spark-connector/spark-connector/). Ray Data should consume the same cataloged data for AI workloads instead of requiring a separate metadata system or repeated physical-path configuration.

Ray's [DatabricksUnityCatalog](https://docs.ray.io/en/latest/data/api/doc/ray.data.DatabricksUnityCatalog.html) connects to a Databricks workspace and cannot resolve Gravitino tables. Ray therefore needs a Gravitino-native path through its existing format-specific APIs.

The integration gives Ray users a consistent table name across Spark and Ray, keeps table discovery in their existing Gravitino deployment, and lets Ray's distributed readers and writers operate on data they already manage. Gravitino's wider Hive, Iceberg, Glue, JDBC, fileset, and other catalog support provides room for later formats, but this proposal does not assume that all of them are Ray-readable today.

## Should this change be within `ray` or outside?

The provider can be built outside Ray using the public alpha [Catalog API](https://docs.ray.io/en/master/data/api/doc/ray.data.Catalog.html). The merged [Ray PR #64193](https://github.com/ray-project/ray/pull/64193) provides design background for the Catalog and DatabricksUnityCatalog integration.

The recommended first step is an externally testable Gravitino provider. Ray maintainers can then decide whether an external Gravitino-maintained connector plus a Ray documentation link is preferable, or whether Ray should accept `GravitinoCatalog` as an alpha public provider.

## Stewardship

### Required Reviewers

- @goutamvenkat-anyscale, as a Ray Data Catalog reviewer.
- Apache Gravitino maintainers for the Python client and Iceberg REST service.

### Shepherd of the Proposal

Proposed: @goutamvenkat-anyscale. This is a request for confirmation, not an assertion that the shepherd has accepted.

## Design and Architecture

### Coverage

| Ray workflow | Gravitino path | Acceptance gate |
| --- | --- | --- |
| Iceberg read and write | Resolve through the dynamic catalog configuration provider of [Gravitino's Iceberg REST service](https://gravitino.apache.org/docs/1.3.0/iceberg-rest-service/) (`catalog-config-provider=dynamic-config-provider`) and use Ray's existing PyIceberg paths. | Existing-table read and write, authorization, and storage access pass real integration tests. |
| External Delta read | Resolve format and location from Gravitino Generic Lakehouse metadata and use `read_delta`. | Driver and distributed workers read successfully with a documented storage identity, including denied-access tests. |
| External Delta write | Use `write_delta` only after defining how Delta changes update registered metadata. | Schema and partition changes cannot silently leave Gravitino metadata stale. |
| Parquet read and write | Evaluate Gravitino-managed Hive or Glue Parquet tables against Ray's existing Parquet APIs. | Format, partition, location, write, and credential semantics are specified and tested before support is claimed. |
| Gravitino-only operation | Use Gravitino endpoints and documented authentication and storage-access mechanisms. | Supported workflows run in an environment without Unity Catalog. |

Iceberg and Delta reads form the initial implementation. Delta writes and Parquet are follow-on capability gaps. Gravitino can meet these goals through its own mechanisms; the provider does not copy another catalog's internal protocols.

### Catalog resolution

`GravitinoCatalog` implements Ray's `Catalog.resolve(table, reader, mode)` contract. Reads use `CatalogAccessMode.READ`; Ray write paths call the same method with `CatalogAccessMode.WRITE`. The configured metalake and logical table identifier select the Gravitino catalog.

The Gravitino main API and Iceberg REST service have distinct endpoints and authentication. The provider must configure them explicitly. The optional `apache-gravitino` Python client is imported only when used. HTTP clients and credentials must not appear in serialized Catalog state, representations, or logs.

### Iceberg

Gravitino selects an Iceberg catalog through the REST `warehouse` property. PyIceberg continues to scan and commit. Gravitino credential vending and access control apply only under their documented deployment settings; server-side Iceberg access control requires auxiliary mode.

### External Delta

The provider validates the Generic Lakehouse catalog, table format, external-table marker, and declared location. It distinguishes Gravitino metadata permission from permission to read or write the underlying files. Gravitino 1.3.0 does not support `ALTER` for these tables and does not list Generic Lakehouse Delta in its credential-vending support. Delta writes and storage access therefore require their own design and validation.

### When both catalogs are present

When an organization also has Unity Catalog, Ray can consume data from each catalog through its own provider. Gravitino's [Iceberg REST backend](https://gravitino.apache.org/docs/1.3.0/iceberg-rest-service/) can proxy another Iceberg REST catalog, and [Unity Catalog exposes an Iceberg REST endpoint](https://docs.databricks.com/aws/en/external-access/iceberg). This suggests a possible Iceberg interoperability path that must be tested for naming, authentication, credentials, reads, writes, and metadata freshness. These sources do not establish reverse-direction or general bidirectional metadata synchronization.

## Compatibility, Deprecation, and Migration Plan

The provider is additive and alpha. Existing Unity Catalog, path-based Delta, and direct Iceberg REST users need no migration. The first compatibility claim should list exact tested Ray, Gravitino, client, PyIceberg, deltalake, Python, and storage versions. No deprecation is proposed.

## Test Plan and Acceptance Criteria

- Unit-test table-name resolution, CatalogAccessMode handling, catalog and format checks, permission errors, missing locations, unsupported operations, and secret-safe behavior.
- Use real Gravitino 1.3.0 services to test Iceberg and external Delta with multiple Ray read tasks. Test Iceberg writes, denied access, and a supported object store.
- Run supported workflows in a Gravitino-only test environment and record exact dependency and service versions.
- Close the Delta-write and Parquet rows in the coverage table with real metadata and storage tests before claiming broad provider coverage.
- If both catalogs are deployed, separately test the proposed Gravitino-to-UC Iceberg REST proxy path for naming, authentication, credentials, reads, writes, and metadata freshness.

No implementation, compatibility result, or cross-catalog interoperability is claimed by this Proposal.

## Follow-on Work

Potential follow-on work includes catalog-based external Delta writes with a defined metadata consistency policy, scoped Hive or Glue Parquet reads, Lance namespace support, and a separately validated Gravitino–Unity Catalog interoperability path.

## References

- [Ray Catalog API](https://docs.ray.io/en/master/data/api/doc/ray.data.Catalog.html)
- [Ray PR #64193](https://github.com/ray-project/ray/pull/64193)
- [Ray Gravitino integration epic #1355](https://github.com/apache/gravitino/issues/1355)
- [Gravitino Iceberg REST service](https://gravitino.apache.org/docs/1.3.0/iceberg-rest-service/)
- [Gravitino external Delta table support](https://gravitino.apache.org/docs/1.3.0/delta-table-support/)
- [Gravitino credential vending](https://gravitino.apache.org/docs/1.3.0/security/credential-vending/)
- [Gravitino Python client](https://gravitino.apache.org/docs/1.3.0/how-to-use-gravitino-python-client/)
