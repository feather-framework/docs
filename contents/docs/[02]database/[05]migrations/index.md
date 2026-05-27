---
category: "database"
title: "Migrations"
description: "Learn how to define, organize, and run database migrations for Feather applications."
order: 5
---

# Migrations
---

Migrations describe how your database is prepared before the application starts using it. They are the place for `CREATE TABLE`, `ALTER TABLE`, `CREATE INDEX`, seed data, and other setup queries.

The access layer defines how your application talks to existing tables. Migrations come first, because they create and update those tables.

In this guide, we will use a small library example with a `book` table.

## Why migrations matter

Database setup should be repeatable. A new developer, a test database, a staging server, and a production server should all be able to create the same schema in a controlled way.

Migrations help you:

- Create tables and indexes consistently.
- Keep schema changes in source control.
- Seed required baseline data.
- Run setup steps in the right order.
- Avoid guessing which database changes were already applied.

In simple terms: migrations make database setup predictable.

## Project setup

In a Feather application, migrations are commonly placed in the Infrastructure layer, next to the database tables they create. The migrator itself can be a separate executable target.

A package can expose a `Migrator` target like this:

```swift
// Package.swift

.executableTarget(
    name: "Migrator",
    dependencies: [
        .product(
            name: "FeatherDatabase",
            package: "feather-database"
        ),
        .product(
            name: "FeatherDatabasePostgres",
            package: "feather-database-postgres"
        ),
        .product(
            name: "Logging",
            package: "swift-log"
        ),
        "Infrastructure",
    ]
)
```

The exact dependencies depend on your project, but the idea is the same:

- The `Migrator` executable starts the database client.
- The Infrastructure module provides the migrations.
- The migration list decides the order.
- The migrator records which migrations already ran.

## The migration protocol

A migration is a small type with an identifier and an `apply` method. The identifier is stored after the migration runs, so the same migration is not applied again on the next startup.

```swift
public protocol Migration: Sendable {
    var id: String { get }

    func apply() async throws
    func revert() async throws
}

public extension Migration {
    var id: String {
        String(reflecting: type(of: self))
    }
}
```

For database migrations, it is useful to add a database-specific protocol. This keeps the migration connected to an `any DatabaseConnection`, while still allowing the migrator to work with a plain list of migrations.

```swift
import FeatherDatabase

public protocol DatabaseMigration: Migration {
    var connection: any DatabaseConnection { get }

    func apply(on connection: any DatabaseConnection) async throws
    func revert(on connection: any DatabaseConnection) async throws
}

public extension DatabaseMigration {
    func apply() async throws {
        try await apply(on: connection)
    }

    func revert() async throws {
        try await revert(on: connection)
    }

    func revert(on connection: any DatabaseConnection) async throws {}
}
```

With this shape, each migration receives a database connection, runs its SQL, and does not need to know how the database client was created.

## Create a table

Start with a table migration. It owns the schema for one area of your application.

```swift
import FeatherDatabase

struct BookTableMigration: DatabaseMigration {
    let connection: any DatabaseConnection

    init(connection: any DatabaseConnection) {
        self.connection = connection
    }

    func apply(on connection: any DatabaseConnection) async throws {
        let queries: [DatabaseQuery] = [
            #"""
            CREATE TABLE IF NOT EXISTS book (
                id TEXT PRIMARY KEY,
                title TEXT NOT NULL,
                author TEXT NOT NULL,
                published_year INTEGER NULL,
                created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
                UNIQUE (author, title)
            );
            """#
        ]

        for query in queries {
            try await connection.run(query: query) { _ in }
        }
    }
}
```

This migration creates the `book` table if it does not already exist. It also adds a unique constraint so the same author cannot have the same title twice.

## Add indexes

Indexes help the database find rows efficiently. If your application often lists books by author or searches by title, add indexes for those access patterns.

```swift
struct BookIndexMigration: DatabaseMigration {
    let connection: any DatabaseConnection

    func apply(on connection: any DatabaseConnection) async throws {
        let queries: [DatabaseQuery] = [
            #"""
            CREATE INDEX IF NOT EXISTS book_author_idx
            ON book (author);
            """#,
            #"""
            CREATE INDEX IF NOT EXISTS book_title_idx
            ON book (title);
            """#,
        ]

        for query in queries {
            try await connection.run(query: query) { _ in }
        }
    }
}
```

Only add indexes that match real query patterns. Indexes speed up reads, but they also add work when inserting or updating rows.

## Alter a table

After an application has been released, you usually should not edit an old table migration. That old migration may already have run in production. Instead, create a new migration that describes the next change.

For example, if books need an optional `summary` column later, add a new migration:

```swift
struct AddBookSummaryMigration: DatabaseMigration {
    let connection: any DatabaseConnection

    func apply(on connection: any DatabaseConnection) async throws {
        try await connection.run(
            query: #"""
                ALTER TABLE book
                ADD COLUMN IF NOT EXISTS summary TEXT NULL;
                """#
        ) { _ in }
    }
}
```

This migration is small, clear, and safe to run after the original `book` table already exists.

When you change a table, also update the access layer after the migration is in place. In this example, the `BookTable.Row` type, row decoding, and `SELECT` queries should include `summary` if the application needs to read it.

## Seed data

Some applications need baseline data. For example, a library app might include a few default books for local development.

Use a separate seed migration for data that should exist after setup:

```swift
struct BookSeedMigration: DatabaseMigration {
    let connection: any DatabaseConnection

    func apply(on connection: any DatabaseConnection) async throws {
        let queries: [DatabaseQuery] = [
            #"""
            INSERT INTO book (
                id,
                title,
                author,
                published_year,
                created_at,
                updated_at
            )
            VALUES
                ('book-1', 'A Gentle Introduction to Databases', 'Example Author', 2026, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
                ('book-2', 'Practical SQL for Swift Apps', 'Example Author', 2026, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP)
            ON CONFLICT (id) DO NOTHING;
            """#
        ]

        for query in queries {
            try await connection.run(query: query) { _ in }
        }
    }
}
```

Seed migrations should be safe to run more than once. `ON CONFLICT DO NOTHING` is useful for this because it avoids inserting duplicate seed rows.

## Build the migration list

Create one function that returns migrations in the order they should run.

```swift
func buildMigrations(
    connection: any DatabaseConnection
) -> [any Migration] {
    [
        // Tables
        BookTableMigration(connection: connection),

        // Schema changes
        BookIndexMigration(connection: connection),
        AddBookSummaryMigration(connection: connection),

        // Seed data
        BookSeedMigration(connection: connection),
    ]
}
```

The order matters. Create tables before creating indexes on those tables. Add columns before queries or seed data depend on those columns. Run seed migrations after the schema exists.

## Track applied migrations

A migrator should remember which migrations have already run. A simple approach is to create a private `_migrations` table and store each migration identifier after it succeeds.

```swift
import FeatherDatabase
import Logging

public struct Migrator {
    public let migrations: [any Migration]
    public let logger: Logger

    public init(
        migrations: [any Migration],
        logger: Logger
    ) {
        self.migrations = migrations
        self.logger = logger
    }

    public func apply(on connection: any DatabaseConnection) async throws {
        try await connection.run(
            query: #"""
                CREATE TABLE IF NOT EXISTS _migrations (
                    id TEXT PRIMARY KEY,
                    created_at TIMESTAMPTZ NOT NULL
                );
                """#
        ) { _ in }

        let existingIds = try await connection.run(
            query: #"""
                SELECT id FROM _migrations;
                """#
        ) { rows in
            try await rows.collect()
                .map { try $0.decode(column: "id", as: String.self) }
        }

        let migrationsToRun = migrations.filter {
            !existingIds.contains($0.id)
        }

        for migration in migrationsToRun {
            logger.info(
                "Running database migration.",
                metadata: ["id": .string(migration.id)]
            )

            try await migration.apply()

            try await connection.run(
                query: #"""
                    INSERT INTO _migrations (
                        id,
                        created_at
                    )
                    VALUES (
                        \#(migration.id),
                        CURRENT_TIMESTAMP
                    );
                    """#
            ) { _ in }
        }
    }
}
```

This migrator does three important things:

- It creates the `_migrations` table if needed.
- It reads the migrations that already ran.
- It only applies migrations that are not recorded yet.

That makes migrations safe to run during deployment or as a separate setup command.

## Run the migrator

Running migrations is a small application of its own. It needs to load configuration, start the database client, open a connection, build the migration list, and apply it.

For Postgres, the shape looks like this:

```swift
import FeatherDatabase
import FeatherDatabasePostgres
import Logging
import PostgresNIO

@main
struct Entrypoint {
    static func main() async throws {
        var logger = Logger(label: "app.migrator")
        logger.logLevel = .info

        let client = PostgresClient(
            configuration: .init(
                host: "127.0.0.1",
                port: 5432,
                username: "feather",
                password: "feather",
                database: "feather",
                tls: .disable
            ),
            backgroundLogger: logger
        )

        let database = DatabaseClientPostgres(
            client: client,
            logger: logger
        )

        let clientTask = Task {
            await client.run()
        }
        defer { clientTask.cancel() }

        try await Task.sleep(for: .milliseconds(25))

        try await database.withConnection { connection in
            let migrator = Migrator(
                migrations: buildMigrations(connection: connection),
                logger: logger
            )
            try await migrator.apply(on: connection)
        }

        logger.info("Database is now migrated.")
    }
}
```

The important part is the lifecycle. `PostgresClient` is a long-running service, so it must be running while the migrator uses the database. In a real project, configuration usually comes from environment variables or config files instead of hard-coded values.

The same pattern works with other Feather database drivers too. Start the driver, open a connection, build the migration list with that connection, and call `migrator.apply(on:)`.

For local development, this can be a separate executable target, for example:

```sh
swift run Migrator
```

For deployment, run the migrator before starting the server process. That way the application starts after the database schema is ready.

## Reset local data

For local development and tests, it can be useful to reset the database before running migrations. For PostgreSQL, one common development-only reset is:

```swift
try await connection.run(
    query: #"""
        DROP SCHEMA public CASCADE;
        CREATE SCHEMA public;
        """#
) { _ in }
```

Only use this for local development, tests, or disposable environments. It deletes existing database objects and data.

## Revert migrations

Some migration systems support reverting changes. A revert method can drop the objects created by a migration:

```swift
func revert(on connection: any DatabaseConnection) async throws {
    try await connection.run(
        query: #"""
            DROP TABLE IF EXISTS book;
            """#
    ) { _ in }
}
```

Be careful with revert queries. Dropping a table deletes data. In production systems, many teams prefer forward-only migrations that add a new migration to fix or change the schema instead of reverting old ones.

## Table migrations and seed migrations

Keep schema and seed data separate when possible.

Table migrations should create or change database structures:

- Tables
- Indexes
- Constraints
- Foreign keys
- Columns

Seed migrations should insert baseline data:

- Default roles
- Default settings
- Required system values
- Example records for local development

Separating them makes it easier to understand what changes the schema and what only inserts data.

## Next step

After migrations create the schema, the access layer describes how application code reads and writes that schema.

In the next guide, we will model the same `book` table as a Swift access type, decode database rows, and organize common queries in one clear place.
