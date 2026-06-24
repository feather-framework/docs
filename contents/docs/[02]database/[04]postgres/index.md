---
category: "database"
title: "Postgres"
description: "Learn how to install, configure, and use the Feather Database Postgres driver with PostgreSQL."
order: 4
---

# Postgres
---

[Feather Database Postgres](https://github.com/feather-framework/feather-database-postgres) is the PostgreSQL driver for the [Feather Database](https://github.com/feather-framework/feather-database) abstraction layer.

It lets your application use the same Feather Database API while connecting to PostgreSQL. This is useful when your application needs a powerful relational database with transactions, constraints, indexes, schemas, row-level locking, and a rich SQL feature set.

The driver is built on top of `PostgresNIO`, supports modern Swift concurrency, and provides the same connection, transaction, query, and row-decoding style as the other Feather Database drivers.

## Installation

Add the Postgres driver to your `Package.swift` dependencies:

```swift
.package(
    url: "https://github.com/feather-framework/feather-database-postgres",
    exact: "1.0.0-rc.1"
),
```

Then add the product to your target dependencies:

```swift
.product(
    name: "FeatherDatabasePostgres",
    package: "feather-database-postgres"
),
```

The package requires Swift 6.1 or newer.

## Local Postgres with Docker

For local development and testing, you can start a PostgreSQL container with Docker:

```sh
docker run --name feather-postgres \
    -e POSTGRES_USER=feather \
    -e POSTGRES_PASSWORD=feather \
    -e POSTGRES_DB=feather \
    -p 5432:5432 \
    -d postgres:18
```

This creates:

- A database named `feather`.
- A user named `feather`.
- A password set to `feather`.
- A PostgreSQL server listening on `127.0.0.1:5432`.

Check that the container is running:

```sh
docker ps
```

You can also wait until PostgreSQL accepts connections:

```sh
docker exec feather-postgres pg_isready -U feather -d feather
```

When you are done testing, stop and remove the container:

```sh
docker rm -f feather-postgres
```

This simple container is useful for local testing. For production-like testing, use TLS and persistent volumes.

## Imports

In the file where you configure the database, import the Postgres driver and the supporting modules:

```swift
import FeatherDatabase
import FeatherDatabasePostgres
import Logging
import NIOSSL
import PostgresNIO
```

`FeatherDatabase` provides the common database protocols. `FeatherDatabasePostgres` provides the PostgreSQL implementation. `PostgresNIO` provides the underlying PostgreSQL client. `NIOSSL` is used when you configure TLS.

## Create a Postgres client

The Postgres driver wraps a `PostgresClient`. Start by creating a logger and a client configuration:

```swift
var logger = Logger(label: "app.database.postgres")
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
```

Then wrap it in a Feather Database client:

```swift
let database = DatabaseClientPostgres(
    client: client,
    logger: logger
)
```

`DatabaseClientPostgres` exposes the common Feather Database API on top of the running Postgres client.

## Run the client

`PostgresClient` is a long-running service. You need to run it while your application uses the database.

For a small script or example, use a task group:

```swift
try await withThrowingTaskGroup(of: Void.self) { group in
    group.addTask {
        await client.run()
    }

    group.addTask {
        try await database.withConnection { connection in
            try await connection.run(
                query: #"""
                    SELECT version() AS "version";
                    """#
            )
        }
    }

    try await group.next()
    group.cancelAll()
}
```

In a server application, run the `PostgresClient` as part of your service lifecycle or server startup. Keep that lifecycle in your composition layer, not inside individual use cases.

## Run a query

Use `database.withConnection` when you want to run work using a managed PostgreSQL connection:

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

PostgreSQL commonly uses double quotes for identifiers, such as table and column names. String values still use regular SQL string syntax.

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
                (\#(1), \#(name));
            """#
    )
}
```

The value passed with `\#(name)` is bound as a query parameter. Internally, the Postgres driver maps bound values to PostgreSQL-style placeholders such as `$1`, `$2`, and so on.

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

    for try await row in sequence {
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
                (\#(1), \#("Milky Way"));
            """#
    )

    try await connection.run(
        query: #"""
            INSERT INTO "galaxies"
                ("id", "name")
            VALUES
                (\#(2), \#("Triangulum"));
            """#
    )
}
```

The Postgres driver starts a transaction with `BEGIN`, commits it when the closure succeeds, and rolls it back if the closure throws.

Transactions are important for write operations that must keep the database consistent. They are also useful when you need row-level locking with PostgreSQL features such as `FOR UPDATE`.

## Use trusted SQL fragments carefully

Normal interpolation binds values. Sometimes you may need to insert a trusted SQL identifier, such as a table name or column name. Use `unescaped` only for values you fully control:

```swift
let table = "galaxies"
let column = "name"

let rows = try await database.withConnection { connection in
    try await connection.run(
        query: #"""
            SELECT "\#(unescaped: column)"
            FROM "\#(unescaped: table)"
            ORDER BY "\#(unescaped: column)" ASC;
            """#
    ) { sequence in
        try await sequence.collect()
    }
}
```

Do not pass user input to `unescaped`. It is for trusted SQL fragments only. For user-provided values, use normal interpolation.

## TLS and certificates

For local experiments, you may connect with `tls: .disable`. For production and production-like testing, use TLS.

TLS protects the connection between your Swift application and the PostgreSQL server. It helps prevent credentials and query data from being sent in plain text over the network.

A secure TLS setup usually has three important files:

- `ca.pem`: The Certificate Authority certificate. The client trusts this certificate.
- `server.pem`: The server certificate presented by PostgreSQL.
- `server.key`: The private key for the server certificate. This stays on the database server.

The client should trust the CA certificate and verify the server certificate. In Swift, configure `NIOSSL` like this:

```swift
let caPath = "/path/to/ca.pem"

var tlsConfig = TLSConfiguration.makeClientConfiguration()
let rootCert = try NIOSSLCertificate.fromPEMFile(caPath)
tlsConfig.trustRoots = .certificates(rootCert)
tlsConfig.certificateVerification = .fullVerification
```

Then require TLS in the Postgres client configuration:

```swift
let client = PostgresClient(
    configuration: .init(
        host: "127.0.0.1",
        port: 5432,
        username: "feather",
        password: "feather",
        database: "feather",
        tls: .require(tlsConfig)
    ),
    backgroundLogger: logger
)
```

With full verification enabled, the certificate must be valid for the host you connect to. If you connect to `127.0.0.1`, the certificate should include `IP:127.0.0.1` in its Subject Alternative Name. If you connect to `localhost`, it should include `DNS:localhost`.

For self-signed local certificates, generate your own CA and sign the server certificate with it. For production, use certificates issued by your infrastructure or a trusted certificate authority.

After connecting, you can verify TLS from SQL:

```swift
try await database.withConnection { connection in
    let rows = try await connection.run(
        query: #"""
            SELECT ssl
            FROM pg_stat_ssl
            WHERE pid = pg_backend_pid();
            """#
    ) { sequence in
        try await sequence.collect()
    }

    let isSecure = try rows[0].decode(column: "ssl", as: Bool.self)
    print(isSecure)
}
```

If TLS is active, PostgreSQL should return `true`.

## Local TLS container

The Postgres driver repository tests use a PostgreSQL container configured with TLS certificates. You can use the same idea for local testing:

1. Generate a local CA certificate.
2. Generate a server certificate and private key.
3. Include `DNS:localhost` and `IP:127.0.0.1` in the server certificate Subject Alternative Name.
4. Mount or copy the certificate files into the container.
5. Configure PostgreSQL with `ssl`, `ssl_cert_file`, `ssl_key_file`, and `ssl_ca_file`.
6. Configure `pg_hba.conf` to allow `hostssl` connections.
7. Configure the Swift client to trust `ca.pem`.

The repository's test setup writes PostgreSQL configuration like this:

```ini
ssl = on
ssl_cert_file = '/certs/server.pem'
ssl_key_file = '/certs/server.key'
ssl_ca_file = '/certs/ca.pem'
```

It also uses `hostssl` rules in `pg_hba.conf` so remote TCP connections must use TLS:

```text
hostssl all all 0.0.0.0/0 md5
hostssl all all ::/0 md5
```

The important point is that the database server owns the server certificate and private key, while the Swift client only needs the CA certificate to verify the server.

## Complete example

The following example connects to the local Docker container, creates a table, inserts two rows, reads them back, and stops the client task.

```swift
import FeatherDatabase
import FeatherDatabasePostgres
import Logging
import NIOSSL
import PostgresNIO

var logger = Logger(label: "example.postgres")
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

try await withThrowingTaskGroup(of: Void.self) { group in
    group.addTask {
        await client.run()
    }

    group.addTask {
        try await database.withConnection { connection in
            try await connection.run(
                query: #"""
                    DROP TABLE IF EXISTS "galaxies" CASCADE;
                    """#
            )

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
                        (\#(1), \#("Andromeda")),
                        (\#(2), \#("Milky Way"));
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
    }

    try await group.next()
    group.cancelAll()
}
```

## When to use Postgres

PostgreSQL is a good choice when you want a mature relational database with strong SQL features, robust transactions, indexing, constraints, schemas, JSON support, and excellent production tooling.

Use PostgreSQL when your application needs a real database server, multiple clients, persistent production storage, strong consistency, and room to grow. For local-only or embedded use cases, SQLite may be simpler. For teams already standardized on MySQL or MariaDB, the MySQL driver may be a better fit.

The benefit of Feather Database is that the Swift API remains consistent across drivers, while still letting each database use its own SQL dialect and operational strengths.
