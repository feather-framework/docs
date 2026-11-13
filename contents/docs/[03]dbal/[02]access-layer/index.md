---
category: "dbal"
title: "Access Layer"
description: "Build typed table access with DatabaseAccessLayer using the System Module Variable feature as a complete example."
order: 3
---

# Access Layer
---

The Database Access Layer is the application-facing part of DBAL. It describes tables with Swift types, turns typed operations into SQL, and decodes database rows into values that repositories and use cases can consume.

The System Module `Variable` feature in the [Feather CMS example application](https://github.com/feathercms/feather) shows the complete pattern. Its infrastructure code keeps the table, row payloads, queries, repository, and migration close together while the rest of the application depends on interfaces.

## Define the table

The table type contains only the table name and its columns. The column types are used later by key-path based queries:

```swift
import FeatherDBAL
import Foundation
import SystemModule

extension Variable {
    struct Table: DatabaseTable {
        static let name = "system_variable"
        static let columns = Columns()

        struct Columns: DatabaseTableColumns {
            let id = DatabaseTableColumn<String>(name: "id")
            let key = DatabaseTableColumn<String>(name: "key")
            let value = DatabaseTableColumn<String>(name: "value")
            let name = DatabaseTableColumn<String?>(name: "name")
            let notes = DatabaseTableColumn<String?>(name: "notes")
            let createdAt = DatabaseTableColumn<Date>(name: "created_at")
            let updatedAt = DatabaseTableColumn<Date>(name: "updated_at")
        }
    }
}
```

The table only describes the database shape. It does not own a database connection or application behavior; it gives DBAL a stable name, columns, and key paths.

## Add row capabilities

Capabilities are added by conforming the table to the protocols needed by the feature. `Variable.Table` supports insert, update, detail, list, and delete operations:

```swift
extension Variable.Table: DatabaseTableWithInsert {
    struct Insert: DatabaseTableRowEncodable {
        let id: String
        let key: String
        let value: String
        let name: String?
        let notes: String?
    }
}

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

extension Variable.Table: DatabaseTableWithDetail {
    struct Detail: DatabaseTableRowDecodable {
        let id: String
        let key: String
        let value: String
        let name: String?
        let notes: String?
    }
}

extension Variable.Table: DatabaseTableWithList {
    struct List: DatabaseTableRowDecodable {
        let id: String
        let key: String
        let value: String
        let name: String?
        let notes: String?
    }
}

extension Variable.Table: DatabaseTableWithDelete {}
```

The `Insert` and `Update` payloads describe values written to the table. The `Detail` and `List` types describe values read from it. They do not need to contain `createdAt` or `updatedAt` when those columns are maintained by the database and are not part of the feature's output.

## Create the access layer

Each query object owns a `DatabaseAccessLayer<Variable.Table>`. The context is injected from the application infrastructure:

```swift
public struct Database: Interface {
    private let accessLayer: DatabaseAccessLayer<Variable.Table>

    public init(context: any DatabaseContext) {
        self.accessLayer = .init(context: context)
    }
}
```

The query object does not choose PostgreSQL, MySQL, or SQLite. That decision is already contained in the context and its configured database platform.

## Find one row

Use a typed key path for a lookup. `selectFirst` returns an optional row, so a query interface can map absence to a feature-specific error:

```swift
public func find(id: String) async throws -> SystemModule.Variable.Model {
    guard
        let row = try await accessLayer.selectFirst(
            by: \.id,
            equals: id
        )
    else {
        throw QueryError.notFound
    }

    return .init(
        id: row.id,
        key: row.key,
        value: row.value,
        name: row.name,
        notes: row.notes
    )
}
```

The query stays typed from the key path through the returned `Detail` row. No table name or SQL column string is repeated in the feature query.

## List, filter, order, and limit

The list query starts with a `DatabaseQuerySelect<Variable.Table>` and applies filters conditionally. The Variable module supports ID filtering, search over several columns, deterministic ordering, and a bounded page size:

```swift
public func list(
    query: SystemModule.Variable.List.Input.Query
) async throws -> SystemModule.Variable.List.Output {
    let items = try await accessLayer.select { queryBuilder in
        applyFilters(query, to: queryBuilder)
            .order(by: \.key, .ascending)
            .limit(min(max(query.limit ?? 20, 1), 100))
    }
    .map { row in
        .init(
            id: row.id,
            key: row.key,
            value: row.value,
            name: row.name,
            notes: row.notes
        )
    }

    return .init(items: items)
}

private func applyFilters(
    _ query: SystemModule.Variable.List.Input.Query,
    to initialQuery: DatabaseQuerySelect<Variable.Table>
) -> DatabaseQuerySelect<Variable.Table> {
    var result = initialQuery

    if let ids = query.ids, !ids.isEmpty {
        result = result.where(\.id, in: ids)
    }

    if let search = query.search {
        result = result.where(
            searchText: search,
            columns: [
                .init(\.key),
                .init(\.name),
                .init(\.value),
                .init(\.notes),
            ]
        )
    }

    return result
}
```

The query builder returns a new query value when filters are added. This makes it straightforward to build a base query and apply optional conditions without falling back to string concatenation.

## Insert, update, and delete

Repositories translate feature values into table row payloads. The access layer then encodes those payloads and applies the operation to the selected table:

```swift
public func insert(_ model: SystemModule.Variable.Model.New) async throws
    -> SystemModule.Variable.Model
{
    let row = Variable.Table.Insert(
        id: context.idGenerator.generate(),
        key: model.key,
        value: model.value,
        name: model.name,
        notes: model.notes
    )

    try await accessLayer.insert(row)

    return .init(
        id: row.id,
        key: row.key,
        value: row.value,
        name: row.name,
        notes: row.notes
    )
}

public func update(_ model: SystemModule.Variable.Model) async throws
    -> SystemModule.Variable.Model
{
    let row = Variable.Table.Update(
        key: model.key,
        value: model.value,
        name: model.name,
        notes: model.notes
    )

    try await accessLayer.update(row, where: \.id, equals: model.id)
    return model
}

public func delete(ids: [String]) async throws {
    _ = try await accessLayer.delete(where: \.id, in: ids)
}
```

`afterEncode` is useful for table-owned values. In this example, every update sets `updated_at` to `.currentTimestamp` without exposing that implementation detail to the repository.

## Keep repositories and queries separate

The Variable module separates responsibilities into small files:

- `Variable+Table.swift` defines the table and columns.
- `Variable+Table+Rows.swift` adds insert, update, detail, list, and delete capabilities.
- `Variable+List+Query+Database.swift` implements list filtering and pagination.
- `Variable+Detail+Query+Database.swift` implements one-row lookup.
- `Variable+Repository+Database.swift` coordinates feature values and table rows.
- `Variable+Table+Migration.swift` defines the table schema.

This keeps the Database Access Layer close to the feature while preventing raw SQL and database context details from leaking into application use cases.

## Schema belongs next to the table

The Variable migration uses the same column definitions to create the schema:

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

The migration converts this schema into platform-specific SQL. The table and its access code remain independent from the concrete database driver.
