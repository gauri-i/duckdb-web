---
layout: post
title: "Schema Evolution End-to-End: Operational Database → Apache Iceberg → DuckDB"
authors:
  - Olake Go
excerpt: "Schema changes in operational databases — adding columns, widening types, renaming fields — can silently break downstream analytics pipelines. This post shows how Apache Iceberg's metadata-only schema evolution, combined with CDC replication and DuckDB's catalog-aware query engine, makes these changes transparent from source to query."
tag: using-duckdb
---

Schema changes happen constantly in production databases. A new column is added to track a feature flag. A `VARCHAR(50)` gets widened. A field is renamed to match a new convention. In traditional pipelines these changes require manual intervention: update the warehouse schema, backfill historical rows, fix the downstream queries. In an Iceberg-based pipeline, most of these changes propagate automatically and remain fully queryable in DuckDB without any downtime.

This post walks through the complete path: an operational database undergoes schema changes, a CDC tool replicates those changes into an Iceberg table, and DuckDB reads the evolved schema without any manual step.

## The Setup

The pipeline has three components:

1. **An operational database** — the source of truth for transactional data.
2. **[Olake Go](https://olake.io/olake-go/)** — an open-source CDC replication tool that continuously syncs the source database into Apache Iceberg tables on an object store, propagating schema changes as they happen.
3. **DuckDB** — queries the Iceberg table via a REST Catalog, always reading the current schema.

The Iceberg table lives in a REST Catalog (AWS Glue, Lakekeeper, Apache Polaris, and others are all supported). DuckDB attaches the catalog once and queries from there.

```sql
INSTALL iceberg;
LOAD iceberg;
LOAD httpfs;

CREATE SECRET iceberg_secret (
    TYPE iceberg,
    TOKEN '⟨bearer_token⟩'
);

ATTACH '⟨warehouse⟩' AS my_catalog (
    TYPE iceberg,
    SECRET iceberg_secret,
    URI '⟨https://rest_endpoint.example.com⟩'
);
```

With the catalog attached, the `orders` table (replicated from the source database) is immediately queryable:

```sql
FROM my_catalog.production.orders LIMIT 5;
```

```text
┌──────────┬─────────────┬──────────────────────┬──────────┐
│ order_id │ customer_id │     order_time       │  amount  │
│  int64   │    int64    │      timestamp       │  double  │
├──────────┼─────────────┼──────────────────────┼──────────┤
│        1 │         101 │ 2026-10-01 08:15:00  │   49.99  │
│        2 │         102 │ 2026-10-01 09:22:00  │  129.00  │
└──────────┴─────────────┴──────────────────────┴──────────┘
```

## Scenario 1: Adding a Column

A new `region` column is added to `orders` in the source database. Olake Go detects the change at the next sync and adds the field to the Iceberg schema with a new field ID. Historical rows receive `NULL` for the new column; new rows carry the value.

DuckDB sees the change on the next query — no `ALTER TABLE` required:

```sql
FROM my_catalog.production.orders LIMIT 5;
```

```text
┌──────────┬─────────────┬──────────────────────┬──────────┬─────────┐
│ order_id │ customer_id │     order_time       │  amount  │ region  │
│  int64   │    int64    │      timestamp       │  double  │ varchar │
├──────────┼─────────────┼──────────────────────┼──────────┼─────────┤
│        1 │         101 │ 2026-10-01 08:15:00  │   49.99  │  NULL   │
│        2 │         102 │ 2026-10-01 09:22:00  │  129.00  │  NULL   │
│        3 │         103 │ 2026-10-15 11:05:00  │   75.50  │  us-east│
└──────────┴─────────────┴──────────────────────┴──────────┴─────────┘
```

Historical rows retain `NULL` for `region` — consistent with Iceberg's field-ID-based schema tracking, which never rewrites existing data files.

## Scenario 2: Widening a Type

The `customer_id` column is promoted from `INT` to `BIGINT` in the source to accommodate a larger ID space. Iceberg v2 supports this widening promotion natively — no data files are rewritten. Olake Go propagates the change to the Iceberg schema; DuckDB reflects it immediately:

```sql
DESCRIBE my_catalog.production.orders;
```

```text
┌─────────────┬─────────────┬─────────┐
│ column_name │ column_type │  null   │
├─────────────┼─────────────┼─────────┤
│ order_id    │   BIGINT    │   YES   │
│ customer_id │   BIGINT    │   YES   │  -- widened from INTEGER
│ order_time  │  TIMESTAMP  │   YES   │
│ amount      │   DOUBLE    │   YES   │
│ region      │   VARCHAR   │   YES   │
└─────────────┴─────────────┴─────────┘
```

Supported widening promotions (Iceberg v2): `INT → BIGINT`, `FLOAT → DOUBLE`, `DATE → TIMESTAMP`, and `DECIMAL` precision increases. These all take effect without a full table rewrite.

## Scenario 3: Dropping a Column

When `amount` is dropped from the source table, Olake Go updates the Iceberg schema, but the column is not physically removed from existing data files. This is an intentional Iceberg design choice: historical snapshots remain fully queryable. New rows arrive without the column; queries against the current table no longer return it by default.

You can still query historical snapshots in DuckDB using `AT (VERSION = ⟨snapshot_id⟩)`:

```sql
-- Current snapshot: no amount column
FROM my_catalog.production.orders LIMIT 3;

-- Historical snapshot: amount column present
FROM my_catalog.production.orders
    AT (VERSION = 5981234567890123456)
LIMIT 3;
```

This is one of the key advantages of Iceberg's snapshot-based architecture: dropping a column from the live table never destroys historical data.

## Why This Matters for DuckDB Users

DuckDB's `iceberg` extension resolves table schemas from the catalog's current metadata on every query. When a CDC tool updates the Iceberg schema in the catalog, DuckDB picks up the change without any session restart or manual schema update. The result:

* **Add a column upstream** → it appears in DuckDB on the next query, with `NULL` for historical rows.
* **Widen a type** → DuckDB reflects the wider type; no cast errors.
* **Drop a column** → it disappears from the live schema; historical snapshots remain queryable.

For analytics workflows that run against continuously-replicated operational data, this means schema changes in the source database no longer require a coordinated pipeline pause. The Iceberg catalog is the single source of schema truth, and DuckDB reads it directly.

## Further Reading

* [Writing to Iceberg with DuckDB]({% link docs/2.0/core_extensions/iceberg/writing_to_iceberg.md %}#alter-table-and-schema-evolution)
* [Iceberg Catalogs supported by DuckDB]({% link docs/2.0/core_extensions/iceberg/catalogs.md %})
* [Olake Go CDC replication](https://olake.io/olake-go/)
* [Olake Go schema evolution documentation](https://olake.io/docs/features/schema/)
