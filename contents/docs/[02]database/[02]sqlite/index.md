---
category: "database"
title: "SQLite"
description: "Learn how to install, configure, and use the Feather Database SQLite driver in a Swift application."
order: 2
---

# SQLite
---

[Feather Database SQLite](https://github.com/feather-framework/feather-database-sqlite) is the SQLite driver for the [Feather Database](https://github.com/feather-framework/feather-database) abstraction layer.

It lets your application use the same Feather Database API while storing data in SQLite. This is useful for local development, test environments, small services, tools, prototypes, and applications that need a simple file-based database.

The driver is built on top of `SQLiteNIO`, supports modern Swift concurrency, uses a connection pool, and provides the same connection, transaction, query, and row-decoding style as the other Feather database drivers.

## Installation

Add the SQLite driver to your `Package.swift` dependencies:

```swift
.package(
    url: "https://github.com/feather-framework/feather-database-sqlite",
    exact: "1.0.0-beta.9"
),
```

Then add the product to your target dependencies:

```swift
.product(
    name: "FeatherDatabaseSQLite",
    package: "feather-database-sqlite"
),
```

The package requires Swift 6.1 or newer.

## Imports

In the file where you configure the database, import the SQLite driver and its supporting modules:

```swift
import FeatherDatabase
import FeatherDatabaseSQLite
import Logging
import SQLiteNIO
import SQLiteNIOExtras
```

`FeatherDatabase` provides the common database protocols. `FeatherDatabaseSQLite` provides the SQLite implementation. `SQLiteNIO` and `SQLiteNIOExtras` provide the underlying SQLite client and storage configuration.

## Create a SQLite client

Start by creating a logger and a SQLite client configuration.

```swift
var logger = Logger(label: "app.database.sqlite")
logger.logLevel = .info

let configuration = SQLiteClient.Configuration(
    storage: .file(path: "/path/to/app.sqlite"),
    logger: logger
)

let client = SQLiteClient(configuration: configuration)
let database = DatabaseClientSQLite(
    client: client,
    logger: logger
)
```

The `SQLiteClient` owns the SQLite connection pool. The `DatabaseClientSQLite` wraps that client and exposes the common Feather Database API.

For tests or short-lived examples, you can use an in-memory database:

```swift
let configuration = SQLiteClient.Configuration(
    storage: .memory,
    logger: logger
)
```

Use a file database when data should persist after the process exits. Use an in-memory database when you only need temporary data.

## Start and stop the client

Before running queries, start the client:

```swift
try await client.run()
```

When your application shuts down, stop it:

```swift
await client.shutdown()
```

For small scripts, tests, or examples, the full lifecycle can look like this:

```swift
try await client.run()

do {
    try await database.withConnection { connection in
        try await connection.run(
            query: #"""
                SELECT sqlite_version() AS "version";
                """#
        )
    }

    await client.shutdown()
}
catch {
    await client.shutdown()
    throw error
}
```

In a server application, you will usually connect this startup and shutdown behavior to your service lifecycle instead of managing it manually in every place that uses the database.

## Configure the connection pool

The SQLite client uses a connection pool. You can configure the minimum and maximum number of connections:

```swift
let configuration = SQLiteClient.Configuration(
    storage: .file(path: "/path/to/app.sqlite"),
    logger: logger,
    minimumConnections: 1,
    maximumConnections: 4
)
```

The minimum connection count is warmed up when `client.run()` is called. The maximum connection count controls how many SQLite connections the pool may open.

You can also configure SQLite-specific options:

```swift
let configuration = SQLiteClient.Configuration(
    storage: .file(path: "/path/to/app.sqlite"),
    logger: logger,
    minimumConnections: 1,
    maximumConnections: 4,
    journalMode: .wal,
    foreignKeysMode: .on,
    busyTimeoutMilliseconds: 5_000
)
```

Useful options:

- `journalMode`: Sets the SQLite journal mode. `wal` can improve concurrency for file-based databases.
- `foreignKeysMode`: Enables or disables SQLite foreign key enforcement. The default is `on`.
- `busyTimeoutMilliseconds`: Tells SQLite how long to wait when the database is temporarily locked.

## Run a query

Use `database.withConnection` when you want to run work on a pooled connection:

```swift
try await database.withConnection { connection in
    try await connection.run(
        query: #"""
            CREATE TABLE IF NOT EXISTS "galaxies" (
                "id" INTEGER PRIMARY KEY,
                "name" TEXT NOT NULL
            );
            """#
    )
}
```

The connection is leased from the pool for the duration of the closure and returned to the pool when the closure finishes.

## Insert data

Feather Database queries support Swift string interpolation for parameter binding:

```swift
let name = "Andromeda"

try await database.withConnection { connection in
    try await connection.run(
        query: #"""
            INSERT INTO "galaxies"
                ("id", "name")
            VALUES
                (NULL, \#(name));
            """#
    )
}
```

The value passed with `\#(name)` is bound as a query parameter. This keeps the SQL readable while avoiding manual string concatenation for dynamic values.

Supported bound value types include:

- `String`
- `String?`
- `Int`
- `Double`
- `Bool`

When an optional string is `nil`, the query writes `NULL`.

## Query and decode rows

Use the query handler to collect rows and decode column values:

```swift
let rows = try await database.withConnection { connection in
    try await connection.run(
        query: #"""
            SELECT
                "id",
                "name"
            FROM "galaxies"
            ORDER BY "name" ASC;
            """#
    ) { sequence in
        try await sequence.collect()
    }
}

for row in rows {
    let id = try row.decode(column: "id", as: Int.self)
    let name = try row.decode(column: "name", as: String.self)
    print(id, name)
}
```

A row can decode individual columns by name. Use optional types when the column can contain `NULL`:

```swift
let notes = try row.decode(column: "notes", as: String?.self)
```

If the column is missing or the value cannot be converted to the requested type, decoding throws an error.

## Iterate result rows

The result sequence is an async sequence. You can collect all rows, or iterate them:

```swift
try await database.withConnection { connection in
    let sequence = try await connection.run(
        query: #"""
            SELECT
                "id",
                "name"
            FROM "galaxies"
            ORDER BY "id" ASC;
            """#
    )

    for await row in sequence {
        let name = try row.decode(column: "name", as: String.self)
        print(name)
    }
}
```

Use `collect()` when you want an array. Use iteration when the calling code is easier to express row by row.

## Use transactions

Use `database.withTransaction` when multiple operations should succeed or fail together:

```swift
try await database.withTransaction { connection in
    try await connection.run(
        query: #"""
            INSERT INTO "galaxies"
                ("id", "name")
            VALUES
                (NULL, \#("Milky Way"));
            """#
    )

    try await connection.run(
        query: #"""
            INSERT INTO "galaxies"
                ("id", "name")
            VALUES
                (NULL, \#("Triangulum"));
            """#
    )
}
```

The SQLite driver starts a transaction with `BEGIN`, commits it when the closure succeeds, and rolls it back if the closure throws.

This is important for write operations that must keep the database consistent. For example, if you insert a parent row and several child rows, a transaction prevents the database from keeping only part of the change when one insert fails.

## Use trusted SQL fragments carefully

Normal interpolation binds values. Sometimes you may need to insert a trusted SQL identifier, such as a table name or column name. Use `unescaped` only for values you fully control:

```swift
let table = "galaxies"
let column = "name"

let rows = try await database.withConnection { connection in
    try await connection.run(
        query: #"""
            SELECT \#(unescaped: column)
            FROM \#(unescaped: table)
            ORDER BY \#(unescaped: column) ASC;
            """#
    ) { sequence in
        try await sequence.collect()
    }
}
```

Do not pass user input to `unescaped`. It is for trusted SQL fragments only. For user-provided values, use normal interpolation.

## Service lifecycle support

The package includes a `ServiceLifecycle` trait by default. This adds `DatabaseServiceSQLite`, which can manage the SQLite client as a service.

```swift
import FeatherDatabaseSQLite
import Logging
import ServiceLifecycle
import SQLiteNIO
import SQLiteNIOExtras

var logger = Logger(label: "app")

let configuration = SQLiteClient.Configuration(
    storage: .file(path: "/path/to/app.sqlite"),
    logger: logger
)

let client = SQLiteClient(configuration: configuration)
let database = DatabaseClientSQLite(client: client, logger: logger)

let serviceGroup = ServiceGroup(
    services: [
        DatabaseServiceSQLite(client),
    ],
    logger: logger
)

try await serviceGroup.run()
```

When the service starts, the SQLite client is started. When the service receives a graceful shutdown signal, the client is shut down.

This is the preferred shape for server applications that already use `swift-service-lifecycle`.

## Complete example

The following example creates an in-memory SQLite database, creates a table, inserts two rows, reads them back, and shuts down the client.

```swift
import FeatherDatabase
import FeatherDatabaseSQLite
import Logging
import SQLiteNIO
import SQLiteNIOExtras

var logger = Logger(label: "example.sqlite")
logger.logLevel = .info

let configuration = SQLiteClient.Configuration(
    storage: .memory,
    logger: logger
)

let client = SQLiteClient(configuration: configuration)
let database = DatabaseClientSQLite(
    client: client,
    logger: logger
)

try await client.run()

do {
    try await database.withConnection { connection in
        try await connection.run(
            query: #"""
                CREATE TABLE IF NOT EXISTS "galaxies" (
                    "id" INTEGER PRIMARY KEY,
                    "name" TEXT NOT NULL
                );
                """#
        )
    }

    try await database.withTransaction { connection in
        try await connection.run(
            query: #"""
                INSERT INTO "galaxies"
                    ("id", "name")
                VALUES
                    (NULL, \#("Andromeda")),
                    (NULL, \#("Milky Way"));
                """#
        )
    }

    let galaxies = try await database.withConnection { connection in
        try await connection.run(
            query: #"""
                SELECT
                    "id",
                    "name"
                FROM "galaxies"
                ORDER BY "name" ASC;
                """#
        ) { sequence in
            try await sequence.collect()
        }
    }

    for galaxy in galaxies {
        let id = try galaxy.decode(column: "id", as: Int.self)
        let name = try galaxy.decode(column: "name", as: String.self)
        print("\(id): \(name)")
    }

    await client.shutdown()
}
catch {
    await client.shutdown()
    throw error
}
```

## When to use SQLite

SQLite is a good choice when you want a simple embedded database without running a separate database server. It works especially well for local development, tests, prototypes, command-line tools, small applications, and single-node deployments.

For larger multi-node production systems, you may prefer PostgreSQL or MySQL. The benefit of Feather Database is that the Swift API remains consistent across drivers, while still letting each database use its own SQL dialect and operational strengths.
