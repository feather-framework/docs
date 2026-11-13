---
category: "dbal"
title: "Overview"
description: "Learn what Feather DBAL provides, how its database drivers fit together, and where to start."
order: 1
---

# Overview
---

[Feather DBAL](https://github.com/feather-framework/feather-dbal) is the typed Database Access Layer built on top of [Feather Database](https://github.com/feather-framework/feather-database) and [Feather SQL](https://github.com/feather-framework/feather-sql).

It gives an application a driver-independent way to describe tables, rows, schema changes, and typed select, insert, update, and delete operations. DBAL does not open connections itself; connections and concrete drivers belong to the database layer.

## Drivers

The DBAL API stays the same while the platform and SQL dialect change underneath it:

- [PostgreSQL](https://github.com/feather-framework/feather-dbal-postgres) — `FeatherDBALPostgres`, `PostgresDatabasePlatform`, release `1.0.0-alpha.1`.
- [MySQL](https://github.com/feather-framework/feather-dbal-mysql) — `FeatherDBALMySQL`, `MySQLDatabasePlatform`, release `1.0.0-alpha.1`.
- [SQLite](https://github.com/feather-framework/feather-dbal-sqlite) — `FeatherDBALSQLite`, `SQLiteDatabasePlatform`, release `1.0.0-alpha.1`.

Each driver package combines its concrete Feather Database driver with the matching Feather SQL dialect and exposes a `DatabasePlatform` to DBAL.

## What DBAL provides

DBAL is the Database Access Layer for Feather applications. Its documentation is organized around the four tasks you perform when working with a database:

1. **SQL** covers the lower-level, SQLKit-derived query builder, parameter bindings, and dialect-aware SQL generation.
2. **Access Layer** covers typed table definitions, reusable database operations, and row decoding through `DatabaseAccessLayer`.
3. **Migrations** covers schema creation, schema changes, indexes, and seed data.
4. **Advanced** covers production database concerns such as constraints, error handling, identifiers, and timestamps.

The selected database platform and connection remain infrastructure details. They supply the SQL dialect and execute the generated queries, while the application-facing table and access code can remain the same when the infrastructure changes from PostgreSQL to MySQL or SQLite.

## Installation

Add the core DBAL package and the platform package for the database you use:

```swift
.package(
    url: "https://github.com/feather-framework/feather-dbal",
    exact: "1.0.0-alpha.1"
),
.package(
    url: "https://github.com/feather-framework/feather-dbal-postgres",
    exact: "1.0.0-alpha.1"
),
```

Add the corresponding products to the target that owns your database access:

```swift
.product(
    name: "FeatherDBAL",
    package: "feather-dbal"
),
.product(
    name: "FeatherDBALPostgres",
    package: "feather-dbal-postgres"
),
```

For package tests, use the `FeatherDBALTesting` product from `feather-dbal`. It provides a lightweight mock platform without changing the table-facing code.
