---
category: "database"
title: "Access Layer"
description: "Learn how to model database tables, decode rows, and organize SQL queries with a lightweight access layer."
order: 6
---

# Access Layer
---

The database access layer is the lowest application-owned layer above Feather Database. It is the place where you model tables, write SQL, decode rows, and keep database-specific details out of the rest of your application.

This layer is intentionally simple. It does not try to be an ORM. Instead, each table gets a small Swift type that owns:

- The database table name and returned columns.
- The row shape returned by SQL queries.
- The input shape used when inserting or updating rows.
- The table-specific queries, such as `create`, `find`, `list`, `update`, and `delete`.

In the migrations guide, we created a small `book` table. In this guide, we will model that table in Swift and write the queries that use it.

## Why use an access layer?

Without an access layer, SQL can spread through repositories, use cases, route handlers, tests, and scripts. That makes the code harder to change because the table structure is no longer described in one place.

An access layer gives every table a clear home. If you need to rename a column, add an index, change row decoding, or adjust a query, you know where to look.

In simple terms: one table, one table access type.

## Basic shape

A table access type usually follows this structure:

```swift
import FeatherDatabase

import struct Foundation.Date

struct BookTableError: Error {
    let message: String
}

struct BookTable {
    struct Row {
        struct Create {
            let id: String
            let title: String
            let author: String
            let publishedYear: Int?
        }

        let id: String
        let title: String
        let author: String
        let publishedYear: Int?
        let createdAt: Date
        let updatedAt: Date
    }

    let connection: any DatabaseConnection

    init(connection: any DatabaseConnection) {
        self.connection = connection
    }
}
```

The table type receives an `any DatabaseConnection`. This keeps it independent from the concrete database driver. The same pattern can be used with Postgres, MySQL, or SQLite as long as the SQL syntax you write is supported by the database.

## Row types

The `Row` type describes the database row returned by the table.

```swift
struct Row {
    let id: String
    let title: String
    let author: String
    let publishedYear: Int?
    let createdAt: Date
    let updatedAt: Date
}
```

This is not necessarily your domain model or API response. It is the database representation. Keeping it separate gives you room to map database rows into domain objects or DTOs later.

For insert operations, use a nested payload type:

```swift
struct Create {
    let id: String
    let title: String
    let author: String
    let publishedYear: Int?
}
```

This avoids passing a full `Row` when the database creates some fields itself, such as `created_at` and `updated_at`.

## Row decoding

Each row type should know how to decode itself from a `DatabaseRow`.

Put the decoding initializer in an extension instead of inside the `Row` struct. This keeps Swift's generated memberwise initializer available, which is useful when tests, seed helpers, or adapters need to construct row values directly.

```swift
extension BookTable.Row {
    init(from row: DatabaseRow) throws {
        self.id = try row.decode(column: "id", as: String.self)
        self.title = try row.decode(column: "title", as: String.self)
        self.author = try row.decode(column: "author", as: String.self)
        self.publishedYear = try row.decode(
            column: "published_year",
            as: Int?.self
        )
        self.createdAt = try row.decode(column: "created_at", as: Date.self)
        self.updatedAt = try row.decode(column: "updated_at", as: Date.self)
    }
}
```

This keeps decoding consistent across every query. If a query returns a book row, it should return the columns needed by this initializer.

Use optional Swift types for nullable database columns. For example, `published_year` can be `NULL`, so it decodes as `Int?`.

## Create rows

The `create` method inserts a row and returns the inserted database row.

```swift
extension BookTable {
    func create(row: Row.Create) async throws -> Row {
        try await connection.run(
            query: #"""
                INSERT INTO book (
                    id,
                    title,
                    author,
                    published_year
                )
                VALUES (
                    \#(row.id),
                    \#(row.title),
                    \#(row.author),
                    \#(row.publishedYear)
                )
                RETURNING
                    id,
                    title,
                    author,
                    published_year,
                    created_at,
                    updated_at;
                """#
        ) { sequence in
            guard let row = try await sequence.collect().first else {
                throw BookTableError(
                    message: "Could not return inserted book row"
                )
            }
            return try Row(from: row)
        }
    }
}
```

There are a few important details here:

- Values are passed with Swift string interpolation, such as `\#(row.title)`.
- The query uses `RETURNING` so the database can return generated values.
- The method decodes the result through `Row(from:)`.
- The method throws a table-specific error if the database does not return a row.

## Find one row

A `find` method should return an optional row:

```swift
extension BookTable {
    func find(id: String) async throws -> Row? {
        try await connection.run(
            query: #"""
                SELECT
                    id,
                    title,
                    author,
                    published_year,
                    created_at,
                    updated_at
                FROM book
                WHERE id=\#(id)
                LIMIT 1;
                """#
        ) { sequence in
            guard let row = try await sequence.collect().first else {
                return nil
            }
            return try Row(from: row)
        }
    }
}
```

Returning `nil` is useful when absence is a valid result. A repository or use case can decide later whether that should become a `notFound` error.

## List rows

A `list` method returns an array:

```swift
extension BookTable {
    func list(author: String) async throws -> [Row] {
        try await connection.run(
            query: #"""
                SELECT
                    id,
                    title,
                    author,
                    published_year,
                    created_at,
                    updated_at
                FROM book
                WHERE author=\#(author)
                ORDER BY title ASC, id ASC;
                """#
        ) { sequence in
            try await sequence.collect().map { try Row(from: $0) }
        }
    }
}
```

Always make ordering explicit. This makes pagination, tests, and UI behavior more predictable.

## Update rows

An update method should return the updated row when the row exists:

```swift
extension BookTable {
    func updateTitle(
        id: String,
        title: String
    ) async throws -> Row? {
        try await connection.run(
            query: #"""
                UPDATE book
                SET
                    title=\#(title),
                    updated_at=CURRENT_TIMESTAMP
                WHERE id=\#(id)
                RETURNING
                    id,
                    title,
                    author,
                    published_year,
                    created_at,
                    updated_at;
                """#
        ) { sequence in
            guard let row = try await sequence.collect().first else {
                return nil
            }
            return try Row(from: row)
        }
    }
}
```

Use `RETURNING` for updates when the caller needs the latest database state. This avoids running a second `SELECT` after the update.

## Delete rows

A delete method can return a boolean to show whether anything was removed:

```swift
extension BookTable {
    func delete(id: String) async throws -> Bool {
        try await connection.run(
            query: #"""
                DELETE FROM book
                WHERE id=\#(id)
                RETURNING id;
                """#
        ) { sequence in
            try await sequence.collect().first != nil
        }
    }
}
```

This is often more useful than returning `Void`, because the caller can distinguish between "deleted" and "not found".

## Use tables with a connection

An access layer type does not open the database connection itself. The caller provides a connection, then creates the table type from that connection.

For a normal query, use `database.withConnection`:

```swift
let books = try await database.withConnection { connection in
    let books = BookTable(connection: connection)

    return try await books.list(author: "Example Author")
}
```

The table only knows about `any DatabaseConnection`. It does not need to know whether the connection came from PostgreSQL, MySQL, SQLite, a connection pool, or a test helper.

You can also use more than one table with the same connection:

```swift
let detail = try await database.withConnection { connection in
    let books = BookTable(connection: connection)
    let authors = AuthorTable(connection: connection)

    guard let book = try await books.find(id: "book-1") else {
        return nil
    }

    let author = try await authors.find(name: book.author)

    return BookDetail(
        book: book,
        author: author
    )
}
```

This is useful for read operations where several tables are needed to build one result.

## Use tables in a transaction

Use `database.withTransaction` when multiple writes must succeed or fail together. The transaction closure also receives a database connection, so the table setup looks the same:

```swift
try await database.withTransaction { connection in
    let books = BookTable(connection: connection)
    let authors = AuthorTable(connection: connection)

    _ = try await authors.createIfMissing(
        row: .init(
            name: "Example Author"
        )
    )

    _ = try await books.create(
        row: .init(
            id: "book-1",
            title: "A Gentle Introduction to Databases",
            author: "Example Author",
            publishedYear: 2026
        )
    )
}
```

If every operation succeeds, the transaction commits. If any operation throws, the database rolls the transaction back. This keeps related writes consistent and prevents half-finished changes.

The important pattern is simple:

- Open a connection or transaction at the composition boundary.
- Create the table access types with that connection.
- Call the table methods needed for the use case.
- Keep SQL inside the table types, not spread through the application.

## Repositories

The access layer is not the same thing as a repository.

A table type should focus on database work:

- SQL statements.
- Row decoding.
- Table-specific inserts, updates, deletes, and lookups.
- Database-shaped row structs.

A repository should focus on application or domain mapping:

- Convert table rows into domain models.
- Convert domain input into table payloads.
- Decide which table methods are needed for a use case.
- Hide database details from the Application layer.

For example:

```swift
struct DatabaseBookRepository {
    let connection: any DatabaseConnection

    func findBy(id: String) async throws -> Book? {
        let table = BookTable(connection: connection)
        guard let row = try await table.find(id: id) else {
            return nil
        }
        return Book(
            id: row.id,
            title: row.title,
            author: row.author,
            publishedYear: row.publishedYear
        )
    }
}
```

This keeps the table layer small and SQL-focused, while the repository owns the conversion into the rest of the application.
