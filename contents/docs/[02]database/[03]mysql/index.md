---
category: "database"
title: "MySQL"
description: "Learn how to install, configure, and use the Feather Database MySQL driver with MySQL or MariaDB."
order: 3
---

# MySQL
---

[Feather Database MySQL](https://github.com/feather-framework/feather-database-mysql) is the MySQL and MariaDB driver for the [Feather Database](https://github.com/feather-framework/feather-database) abstraction layer.

It lets your application use the same Feather Database API while connecting to a MySQL-compatible database server. This is useful when your application needs a production-ready relational database with transactions, indexes, constraints, and a familiar SQL dialect.

The driver is built on top of `MySQLNIO`, supports modern Swift concurrency, and provides the same connection, transaction, query, and row-decoding style as the other Feather database drivers.

## Installation

Add the MySQL driver to your `Package.swift` dependencies:

```swift
.package(
    url: "https://github.com/feather-framework/feather-database-mysql",
    exact: "1.0.0-beta.5"
),
```

Then add the product to your target dependencies:

```swift
.product(
    name: "FeatherDatabaseMySQL",
    package: "feather-database-mysql"
),
```

The package requires Swift 6.1 or newer.

## Local MySQL with Docker

For local development and testing, you can start a MySQL container with Docker:

```sh
docker run --name feather-mysql \
    -e MYSQL_ROOT_PASSWORD=mysql \
    -e MYSQL_DATABASE=feather \
    -e MYSQL_USER=feather \
    -e MYSQL_PASSWORD=feather \
    -p 3306:3306 \
    -d mysql:8.4
```

This creates:

- A database named `feather`.
- A user named `feather`.
- A password set to `feather`.
- A MySQL server listening on `127.0.0.1:3306`.

Check that the container is running:

```sh
docker ps
```

You can also wait until MySQL accepts connections:

```sh
docker exec feather-mysql mysqladmin ping -h 127.0.0.1 -uroot -pmysql
```

When you are done testing, stop and remove the container:

```sh
docker rm -f feather-mysql
```

This simple container is useful for local testing. For production-like testing, use TLS and persistent volumes.

## Imports

In the file where you configure the database, import the MySQL driver and the supporting networking modules:

```swift
import FeatherDatabase
import FeatherDatabaseMySQL
import Logging
import MySQLNIO
import NIOCore
import NIOPosix
import NIOSSL
```

`FeatherDatabase` provides the common database protocols. `FeatherDatabaseMySQL` provides the MySQL implementation. `MySQLNIO`, `NIOCore`, `NIOPosix`, and `NIOSSL` are used to create and manage the underlying MySQL connection.

## Create a MySQL connection

The MySQL driver wraps an existing `MySQLConnection`. Start by creating a logger and an event loop group:

```swift
var logger = Logger(label: "app.database.mysql")
logger.logLevel = .info

let eventLoopGroup = MultiThreadedEventLoopGroup(numberOfThreads: 1)
```

Then open a connection:

```swift
let connection = try await MySQLConnection.connect(
    to: try SocketAddress(ipAddress: "127.0.0.1", port: 3306),
    username: "feather",
    database: "feather",
    password: "feather",
    tlsConfiguration: nil,
    logger: logger,
    on: eventLoopGroup.next()
)
.get()
```

Finally, wrap it in a Feather database client:

```swift
let database = DatabaseClientMySQL(
    connection: connection,
    logger: logger
)
```

`DatabaseClientMySQL` exposes the common Feather Database API on top of the open MySQL connection.

## Close the connection

When the application shuts down, close the MySQL connection and shut down the event loop group:

```swift
try await connection.close().get()
try await eventLoopGroup.shutdownGracefully()
```

For small scripts, tests, or examples, the full lifecycle can look like this:

```swift
do {
    try await database.withConnection { connection in
        try await connection.run(
            query: #"""
                SELECT VERSION() AS `version`;
                """#
        )
    }

    try await connection.close().get()
    try await eventLoopGroup.shutdownGracefully()
}
catch {
    try await connection.close().get()
    try await eventLoopGroup.shutdownGracefully()
    throw error
}
```

In a server application, keep this lifecycle in your composition or server startup layer, not inside individual use cases.

## Run a query

Use `database.withConnection` when you want to run work on the MySQL connection:

```swift
try await database.withConnection { connection in
    try await connection.run(
        query: #"""
            CREATE TABLE IF NOT EXISTS `galaxies` (
                `id` INTEGER PRIMARY KEY,
                `name` TEXT NOT NULL
            );
            """#
    )
}
```

MySQL commonly uses backticks for identifiers, such as table and column names. String values still use regular SQL string syntax.

## Insert data

Feather Database queries support Swift string interpolation for parameter binding:

```swift
let name = "Andromeda"

try await database.withConnection { connection in
    try await connection.run(
        query: #"""
            INSERT INTO `galaxies`
                (`id`, `name`)
            VALUES
                (\#(1), \#(name));
            """#
    )
}
```

The value passed with `\#(name)` is bound as a query parameter. This keeps SQL readable while avoiding manual string concatenation for dynamic values.

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
                `id`,
                `name`
            FROM `galaxies`
            ORDER BY `name` ASC;
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
                `id`,
                `name`
            FROM `galaxies`
            ORDER BY `id` ASC;
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
            INSERT INTO `galaxies`
                (`id`, `name`)
            VALUES
                (\#(1), \#("Milky Way"));
            """#
    )

    try await connection.run(
        query: #"""
            INSERT INTO `galaxies`
                (`id`, `name`)
            VALUES
                (\#(2), \#("Triangulum"));
            """#
    )
}
```

The MySQL driver starts a transaction with `START TRANSACTION`, commits it when the closure succeeds, and rolls it back if the closure throws.

Transactions are important for write operations that must keep the database consistent. They are also useful when you need row-level locking with MySQL features such as `FOR UPDATE`.

## Use trusted SQL fragments carefully

Normal interpolation binds values. Sometimes you may need to insert a trusted SQL identifier, such as a table name or column name. Use `unescaped` only for values you fully control:

```swift
let table = "galaxies"
let column = "name"

let rows = try await database.withConnection { connection in
    try await connection.run(
        query: #"""
            SELECT `\#(unescaped: column)`
            FROM `\#(unescaped: table)`
            ORDER BY `\#(unescaped: column)` ASC;
            """#
    ) { sequence in
        try await sequence.collect()
    }
}
```

Do not pass user input to `unescaped`. It is for trusted SQL fragments only. For user-provided values, use normal interpolation.

## TLS and certificates

For local experiments, you may connect with `tlsConfiguration: nil`. For production and production-like testing, use TLS.

TLS protects the connection between your Swift application and the MySQL server. It helps prevent credentials and query data from being sent in plain text over the network.

A secure TLS setup usually has three important files:

- `ca.pem`: The Certificate Authority certificate. The client trusts this certificate.
- `server.pem`: The server certificate presented by MySQL or MariaDB.
- `server.key`: The private key for the server certificate. This stays on the database server.

The client should trust the CA certificate and verify the server certificate. In Swift, configure `NIOSSL` like this:

```swift
let caPath = "/path/to/ca.pem"

var tlsConfig = TLSConfiguration.makeClientConfiguration()
let rootCert = try NIOSSLCertificate.fromPEMFile(caPath)
tlsConfig.trustRoots = .certificates(rootCert)
tlsConfig.certificateVerification = .fullVerification
```

Then pass the TLS configuration when opening the connection:

```swift
let connection = try await MySQLConnection.connect(
    to: try SocketAddress(ipAddress: "127.0.0.1", port: 3306),
    username: "feather",
    database: "feather",
    password: "feather",
    tlsConfiguration: tlsConfig,
    logger: logger,
    on: eventLoopGroup.next()
)
.get()
```

With full verification enabled, the certificate must be valid for the host you connect to. If you connect to `127.0.0.1`, the certificate should include `IP:127.0.0.1` in its Subject Alternative Name. If you connect to `localhost`, it should include `DNS:localhost`.

For self-signed local certificates, generate your own CA and sign the server certificate with it. For production, use certificates issued by your infrastructure or a trusted certificate authority.

After connecting, you can verify TLS from SQL:

```swift
try await database.withConnection { connection in
    let rows = try await connection.run(
        query: #"""
            SHOW SESSION STATUS LIKE "ssl_cipher";
            """#
    ) { sequence in
        try await sequence.collect()
    }

    let cipher = try rows[0].decode(column: "Value", as: String.self)
    print(cipher)
}
```

If TLS is active, MySQL should report a non-empty cipher value.

## Local TLS container

The MySQL driver repository tests use a MariaDB container configured with TLS certificates. You can use the same idea for local testing:

1. Generate a local CA certificate.
2. Generate a server certificate and private key.
3. Include `DNS:localhost` and `IP:127.0.0.1` in the server certificate Subject Alternative Name.
4. Mount or copy the certificate files into the container.
5. Configure MariaDB or MySQL with `ssl-ca`, `ssl-cert`, and `ssl-key`.
6. Configure the Swift client to trust `ca.pem`.

The repository's test setup uses a MariaDB image and writes a server configuration like this:

```ini
[mariadbd]
ssl-ca=/etc/mysql/conf.d/ca.pem
ssl-cert=/etc/mysql/conf.d/server.pem
ssl-key=/etc/mysql/conf.d/server.key
```

The important point is that the database server owns the server certificate and private key, while the Swift client only needs the CA certificate to verify the server.

## Complete example

The following example connects to the local Docker container, creates a table, inserts two rows, reads them back, and closes the connection.

```swift
import FeatherDatabase
import FeatherDatabaseMySQL
import Logging
import MySQLNIO
import NIOCore
import NIOPosix
import NIOSSL

var logger = Logger(label: "example.mysql")
logger.logLevel = .info

let eventLoopGroup = MultiThreadedEventLoopGroup(numberOfThreads: 1)

let connection = try await MySQLConnection.connect(
    to: try SocketAddress(ipAddress: "127.0.0.1", port: 3306),
    username: "feather",
    database: "feather",
    password: "feather",
    tlsConfiguration: nil,
    logger: logger,
    on: eventLoopGroup.next()
)
.get()

let database = DatabaseClientMySQL(
    connection: connection,
    logger: logger
)

do {
    try await database.withConnection { connection in
        try await connection.run(
            query: #"""
                DROP TABLE IF EXISTS `galaxies`;
                """#
        )

        try await connection.run(
            query: #"""
                CREATE TABLE IF NOT EXISTS `galaxies` (
                    `id` INTEGER PRIMARY KEY,
                    `name` TEXT NOT NULL
                );
                """#
        )
    }

    try await database.withTransaction { connection in
        try await connection.run(
            query: #"""
                INSERT INTO `galaxies`
                    (`id`, `name`)
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
                    `id`,
                    `name`
                FROM `galaxies`
                ORDER BY `name` ASC;
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

    try await connection.close().get()
    try await eventLoopGroup.shutdownGracefully()
}
catch {
    try await connection.close().get()
    try await eventLoopGroup.shutdownGracefully()
    throw error
}
```

## When to use MySQL

MySQL and MariaDB are good choices when you want a mature relational database server with broad hosting support and a familiar SQL dialect.

Use MySQL or MariaDB when your application needs a real database server, multiple clients, persistent production storage, transactions, indexing, and operational tools. For local-only or embedded use cases, SQLite may be simpler. For larger systems that already standardize on PostgreSQL, the PostgreSQL driver may be a better fit.

The benefit of Feather Database is that the Swift API remains consistent across drivers, while still letting each database use its own SQL dialect and operational strengths.
