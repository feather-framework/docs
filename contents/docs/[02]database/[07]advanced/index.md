---
category: "database"
title: "Advanced"
description: "Learn practical database patterns for indexes, foreign keys, errors, identifiers, and timestamps."
order: 7
---

# Advanced
---

After you have migrations and an access layer, the next step is to make the database model more reliable. This guide covers a few practical topics that appear in most real applications:

- Indexes
- Foreign key constraints
- Error handling
- ID generation
- Dates and timestamps

We will keep using the simple library example from the previous guides.

## Indexes

An index helps the database find rows faster. Without an index, the database may need to scan many rows to answer a query. With the right index, it can jump directly to the rows it needs.

Start with the queries your application actually runs. For example, if the access layer often lists books by author, this query is important:

```sql
SELECT
    id,
    title,
    author,
    published_year,
    created_at,
    updated_at
FROM book
WHERE author = ?
ORDER BY title ASC, id ASC;
```

A matching index can help:

```sql
CREATE INDEX IF NOT EXISTS book_author_title_idx
ON book (author, title, id);
```

The order of columns matters. In this example, `author` comes first because it is used in the `WHERE` clause. `title` and `id` follow because they are used for ordering.

Indexes are useful, but they are not free:

- They speed up reads.
- They take storage space.
- They add work when rows are inserted or updated.

In simple terms: add indexes for important query patterns, not for every column.

## Unique indexes

Some indexes protect data quality. For example, if the same author should not have two books with the same title, use a unique constraint or unique index:

```sql
CREATE UNIQUE INDEX IF NOT EXISTS book_author_title_unique_idx
ON book (author, title);
```

This makes the database enforce the rule. Even if two requests arrive at the same time, the database will not allow duplicate rows.

The access layer can then handle duplicates as a normal database error and map that error into a useful application error.

## Foreign key constraints

A foreign key connects one table to another. It tells the database that a value in one table must point to a real row in another table.

For example, instead of storing the author name directly on `book`, you can create an `author` table:

```sql
CREATE TABLE IF NOT EXISTS author (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

Then the `book` table can reference it:

```sql
CREATE TABLE IF NOT EXISTS book (
    id TEXT PRIMARY KEY,
    author_id TEXT NOT NULL,
    title TEXT NOT NULL,
    published_year INTEGER NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (author_id) REFERENCES author(id) ON DELETE RESTRICT
);
```

This means a book cannot reference an author that does not exist.

The `ON DELETE` behavior is important:

- `RESTRICT` prevents deleting the author while books still reference it.
- `CASCADE` deletes the related books when the author is deleted.
- `SET NULL` clears the reference, but only works when the column allows `NULL`.

Choose this behavior carefully. For business data, `RESTRICT` is often the safest default because it prevents accidental data loss.

## Error handling

Database errors should not leak through the whole application as raw driver errors. Keep database-specific errors close to the access layer or repository, then map them into application-level errors.

For example, a table method can throw a small table-specific error when an insert does not return a row:

```swift
struct BookTableError: Error {
    let message: String
}

extension BookTable {
    func create(row: Row.Create) async throws -> Row {
        try await connection.run(
            query: #"""
                INSERT INTO book (
                    id,
                    author_id,
                    title,
                    published_year
                )
                VALUES (
                    \#(row.id),
                    \#(row.authorId),
                    \#(row.title),
                    \#(row.publishedYear)
                )
                RETURNING
                    id,
                    author_id,
                    title,
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

Repositories can then decide what the error means for the application. For example:

- A unique constraint error can become `bookAlreadyExists`.
- A foreign key error can become `authorNotFound`.
- A missing row can become `nil` or `notFound`, depending on the use case.
- An unexpected database failure can become an internal infrastructure error.

This keeps the rest of the application independent from the concrete database driver.

## ID generation

Every table should have a clear ID strategy. There are two common options:

- Generate IDs in the application.
- Generate IDs in the database.

Application-generated IDs are common in Feather-style applications because they are available before the row is inserted. This is useful when several related rows need to be created together.

```swift
struct CreateBook {
    let id: String
    let authorId: String
    let title: String
    let publishedYear: Int?
}
```

The caller can create the ID before calling the table:

```swift
let row = BookTable.Row.Create(
    id: UUID().uuidString,
    authorId: author.id,
    title: "A Gentle Introduction to Databases",
    publishedYear: 2026
)
```

You can use UUIDs, NanoIDs, or another stable identifier format. The important part is consistency. Pick one strategy for your application and use it everywhere unless there is a good reason not to.

Database-generated IDs can also work, especially with auto-incrementing integers or database UUID defaults. In that case, the access layer should use `RETURNING` when supported, so the caller receives the generated ID immediately.

## Dates and timestamps

Most tables need timestamps. A simple default is:

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
```

Use `created_at` for the time the row was created. Use `updated_at` for the last time the row changed.

When inserting rows, let the database set the timestamps when possible:

```sql
INSERT INTO book (
    id,
    author_id,
    title,
    published_year
)
VALUES (?, ?, ?, ?)
RETURNING
    id,
    author_id,
    title,
    published_year,
    created_at,
    updated_at;
```

When updating rows, update `updated_at` in the same query:

```sql
UPDATE book
SET
    title = ?,
    updated_at = CURRENT_TIMESTAMP
WHERE id = ?
RETURNING
    id,
    author_id,
    title,
    published_year,
    created_at,
    updated_at;
```

In Swift, decode timestamps into `Date`:

```swift
extension BookTable.Row {
    init(from row: DatabaseRow) throws {
        self.id = try row.decode(column: "id", as: String.self)
        self.authorId = try row.decode(column: "author_id", as: String.self)
        self.title = try row.decode(column: "title", as: String.self)
        self.publishedYear = try row.decode(
            column: "published_year",
            as: Int?.self
        )
        self.createdAt = try row.decode(column: "created_at", as: Date.self)
        self.updatedAt = try row.decode(column: "updated_at", as: Date.self)
    }
}
```

For server applications, prefer storing timestamps in UTC. PostgreSQL's `TIMESTAMPTZ` is a good default because it represents an absolute point in time. MySQL and SQLite can also work well, but be consistent about how values are stored and decoded.
