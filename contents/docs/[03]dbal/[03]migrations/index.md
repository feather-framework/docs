---
category: "dbal"
title: "Migrations"
description: "Define platform-independent schema and seed migrations with Feather DBAL, using the System Module Variable table as an example."
order: 4
---

# Migrations
---

Feather DBAL migrations describe database schema and data changes next to the tables and repositories that use them. They do not write PostgreSQL, MySQL, or SQLite-specific SQL directly. Instead, a migration describes a schema with DBAL types, and the selected `DatabasePlatform` builds the appropriate queries.

The System Module `Variable` table in the [Feather CMS example application](https://github.com/feathercms/feather) is a useful example because it separates table creation, seed data, and the application's migration list.

## Migration types

The common migration protocols are small and composable:

```swift
public protocol Migration: Sendable {
    var id: String { get }

    func apply() async throws
    func revert() async throws
}

public protocol DatabaseMigration: Migration {
    var connection: any DatabaseConnection { get }

    func apply(
        on connection: any DatabaseConnection
    ) async throws

    func revert(
        on connection: any DatabaseConnection
    ) async throws
}
```

The default migration identifier is the reflected type name. A `DatabaseMigration` receives a connection, while its dependencies can also include a `DatabasePlatform`, transaction context, event publisher, or other infrastructure values required by the migration.

## Describe the table schema

The [`Variable.Table` definition from the Access Layer guide](/docs/dbal/access-layer/#define-the-table) already provides the table name and typed columns. The migration extends that same table type with a `DatabaseTableSchema`, so schema code reuses the column definitions instead of repeating raw column strings throughout the application:

```swift
extension Variable.Table {
    static var tableCreationSchema: DatabaseTableSchema {
        .init(
            name: Self.name,
            columns: [
                .init(name: Self.columns.id.name, type: .string),
                .init(
                    name: Self.columns.key.name,
                    type: .string,
                    constraints: [.notNull, .unique]
                ),
                .init(
                    name: Self.columns.value.name,
                    type: .string,
                    constraints: [.notNull]
                ),
                .init(name: Self.columns.name.name, type: .string),
                .init(name: Self.columns.notes.name, type: .string),
                .init(
                    name: Self.columns.createdAt.name,
                    type: .timestamp,
                    constraints: [.notNull, .default(.currentTimestamp)]
                ),
                .init(
                    name: Self.columns.updatedAt.name,
                    type: .timestamp,
                    constraints: [.notNull, .default(.currentTimestamp)]
                ),
            ],
            constraints: [
                .primaryKey([
                    .init(name: Self.columns.id.name, type: .string)
                ])
            ]
        )
    }
}
```

The schema expresses intent: `key` is required and unique, `value` is required, timestamps have database defaults, and `id` is the primary key. The platform later maps `.string`, `.timestamp`, and the constraints to its own SQL syntax.

## Build table queries with the platform

The System Module groups the Permission and Variable schemas in one `TableMigration`. `DatabaseQueryCreate` turns each schema into one or more platform-specific queries:

```swift
public struct TableMigration: DatabaseMigration {
    public let connection: any DatabaseConnection
    public let platform: any DatabasePlatform

    public init(
        connection: any DatabaseConnection,
        platform: any DatabasePlatform
    ) {
        self.connection = connection
        self.platform = platform
    }

    public func apply(
        on connection: any DatabaseConnection
    ) async throws {
        let schemas = [
            Permission.Table.tableCreationSchema,
            Variable.Table.tableCreationSchema,
        ]

        for schema in schemas {
            let queryBuilder = DatabaseQueryCreate(
                schema: schema,
                using: platform
            )

            for query in queryBuilder.build() {
                try await connection.run(query: query) { _ in }
            }
        }
    }
}
```

This migration does not need to know whether the connection is PostgreSQL, MySQL, or SQLite. The platform owns the database-specific type and syntax decisions.

## Seed data through repositories

Seed migrations use the same repositories as the rest of the application. The System Module receives seed definitions through events, creates a transaction context, and inserts them through `Variable.Repository.Database`:

```swift
public struct TableSeedMigration: DatabaseMigration {
    public let context: DatabaseTransactionContext
    public let platform: any DatabasePlatform
    private let events: any EventPublisher

    public var connection: any DatabaseConnection {
        context.connection
    }

    public init(
        context: DatabaseTransactionContext,
        platform: any DatabasePlatform,
        events: any EventPublisher
    ) {
        self.context = context
        self.platform = platform
        self.events = events
    }

    public func apply(
        on connection: any DatabaseConnection
    ) async throws {
        let variables = try await events.trigger(
            event: SystemModule.Variable.Event.Seed()
        )
        .flatMap { $0 }

        let repository = SystemModule.Variable.Repository.Database(
            context: context
        )

        let uniqueDefinitions = variables.reduce(
            into: [String: SystemModule.Variable.Seed]()
        ) { definitions, variable in
            definitions[variable.key] = variable
        }

        for variable in uniqueDefinitions.values {
            _ = try await repository.insert(
                try SystemModule.Variable.Model.create(
                    key: variable.key,
                    value: variable.value,
                    name: variable.name,
                    notes: variable.notes
                )
            )
        }
    }
}
```

This keeps seed definitions in feature modules. The migration only coordinates the event, deduplicates definitions by their unique key, and delegates the insert to the System Module repository.

## Build the migration context

The migrator creates one transaction context from the selected platform, connection, and ID generator. The same context is passed to transaction-bound seed migrations and repositories:

```swift
let context = DatabaseTransactionContext(
    platform: platform,
    connection: connection,
    idGenerator: idGenerator
)
```

The application then builds the migrations in dependency order. In the Feather CMS migrator, table migrations run first, followed by media-folder setup and seed migrations:

```swift
public func buildMigrations(
    connection: any DatabaseConnection,
    platform: any DatabasePlatform,
    events: any EventPublisher,
    idGenerator: any IDGenerator
) -> [any Migration] {
    let context = DatabaseTransactionContext(
        platform: platform,
        connection: connection,
        idGenerator: idGenerator
    )

    return [
        SystemInfrastructure.TableMigration(
            connection: connection,
            platform: platform
        ),
        // Other module table migrations...
        SystemInfrastructure.TableSeedMigration(
            context: context,
            platform: platform,
            events: events
        ),
        // Other module seed migrations...
    ]
}
```

The order matters. A table must exist before a repository can insert seed data into it, and a referenced table must be created before a table with a foreign key depends on it.

## Schema migrations versus seed migrations

Keep structural changes and data population separate:

- **Table migrations** create tables, columns, constraints, indexes, and foreign keys with `DatabaseTableSchema` and `DatabaseQueryCreate`.
- **Seed migrations** insert required baseline values through repositories and transaction contexts.
- **Feature modules** contribute seed definitions through events instead of hard-coding every application's values in the system module.

When a released schema needs to change, add a new table migration. Do not edit an old migration that may already have run in production. For a new Variable column, update the table columns and add a follow-up migration that describes the change for every supported platform.

## Why this pattern matters

The migration owns database preparation, while the Database Access Layer owns typed reads and writes. Both use the same table and column definitions, but neither needs to embed a concrete database driver into application code.

This gives Feather applications repeatable setup, platform-specific SQL generation, transaction-aware seeding, and a single place to evolve each module's schema.
