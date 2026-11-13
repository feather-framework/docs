---
category: "dbal"
title: "Advanced"
description: "Apply production database patterns with Feather DBAL, using the System Module Variable table for constraints, IDs, timestamps, and indexes."
order: 5
---

# Advanced
---

The Advanced guide builds on the [Access Layer](/docs/dbal/access-layer/) and [Migrations](/docs/dbal/migrations/) patterns. The System Module `Variable` table shows how constraints, indexes, context dependencies, ID generation, timestamps, and foreign keys are handled by Feather DBAL.

## Enforce invariants in the schema

The Variable schema makes `key` unique and not nullable, makes `value` required, and declares `id` as the primary key:

```swift
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
// ...
constraints: [
    .primaryKey([
        .init(name: Self.columns.id.name, type: .string)
    ])
]
```

The unique constraint protects the database when two requests try to create the same key at the same time. Constraints belong in the table schema so every access path and supported driver applies the same rules.

## Add indexes for real access patterns

The Variable repository looks up records by both `id` and `key`:

```swift
try await accessLayer.selectFirst(by: \.id, equals: id)
try await accessLayer.selectFirst(by: \.key, equals: key)
```

The primary key already supports the `id` lookup. The unique constraint on `key` also creates the database's uniqueness structure and supports the key lookup in the supported drivers. No additional index is needed for this table.

When a table has a frequent filter or sort that is not already covered by a primary key or unique constraint, add an index to its table schema or a follow-up migration. Indexes should follow measured access patterns; they add storage and write overhead.

## Database context and ID generation

`IDGenerator` is a small infrastructure dependency. Feather provides the protocol, while the application composition root chooses the implementation. The CMS application creates a shared `NanoIDGenerator` and stores it in `DatabaseClientContext` together with the database client and platform:

```swift
let idGenerator = NanoIDGenerator()

let databaseContext = DatabaseClientContext(
    database: database,
    idGenerator: idGenerator,
    platform: PostgresDatabasePlatform()
)
```

The context is passed to `DatabaseQueryExecutor` or `DatabaseTransactionExecutor`. When either executor opens a connection or transaction, it creates a request-scoped `DatabaseQueryContext` or `DatabaseTransactionContext`. Both contexts conform to `DatabaseContext`, so they expose the same generator alongside the database connection and platform:

```swift
let transaction = DatabaseTransactionExecutor(
    databaseContext: databaseContext,
    scope: { context in
        Variable.Repository.Database(context: context)
    }
)
```

The database-backed repository stores the context and constructs its `DatabaseAccessLayer` from it. The generated ID is available through `context.idGenerator`, while query encoding and execution use the same context's platform and connection:

```swift
public init(context: any DatabaseContext) {
    self.context = context
    self.accessLayer = .init(context: context)
}

public func insert(
    key: String,
    value: String,
    name: String?,
    notes: String?
) async throws {
    let row = Variable.Table.Insert(
        id: context.idGenerator.generate(),
        key: key,
        value: value,
        name: name,
        notes: notes
    )

    try await accessLayer.insert(row)
}
```

This keeps ID generation configurable at composition time. Tests can inject a deterministic `IDGenerator`, while production can use `NanoIDGenerator`; database code only depends on the `IDGenerator` protocol exposed by its context.

## Let the database own timestamps

The Variable schema gives both timestamp columns database defaults:

```swift
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
```

When an update row is encoded, `afterEncode` changes `updated_at` without requiring every repository method to remember the timestamp:

```swift
extension Variable.Table: DatabaseTableWithUpdate {
    struct Update: DatabaseTableRowEncodable {
        let key: String
        let value: String
        let name: String?
        let notes: String?

        func afterEncode(values: inout DatabaseTableRowValues) throws {
            values[Variable.Table.columns.updatedAt] = .currentTimestamp
        }
    }
}
```

This gives inserts a database-generated `created_at` and `updated_at`, while updates always refresh `updated_at` at the access-layer boundary. Keep timestamp storage consistent across platforms and decode them as `Date` when they belong in a returned row.

## Foreign keys belong in the schema

The System Module `Variable` table does not reference another table, so it does not need a foreign key. For tables that do have a relationship, declare the foreign key in `DatabaseTableSchema` and choose the delete behavior explicitly. Keep the relationship in the migration rather than relying only on repository code; only the database can enforce it safely under concurrent writes.

Use `RESTRICT` when deleting a referenced row must be prevented, `CASCADE` when dependent rows should be removed with it, and `SET NULL` only when the foreign key column is nullable and that behavior matches the relationship.

## The boundary in one example

The current System Module pattern keeps the database responsibilities separate:

- The **table schema** declares primary keys, uniqueness, nullability, and timestamp defaults.
- The **repository** generates IDs, creates typed row payloads, and delegates to `DatabaseAccessLayer`.
- The **access layer** encodes rows, updates timestamps, and executes through the configured context.
- The **database platform** turns the schema and operations into driver-specific SQL.

This keeps database correctness in the schema and access boundary while application concerns remain outside the persistence implementation.
