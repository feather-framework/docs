---
category: "dbal"
title: "SQL"
description: "Build parameterized, dialect-aware SQL queries with FeatherSQL, the SQLKit-derived lower-level query builder."
order: 2
---

# SQL
---

[FeatherSQL](https://github.com/feather-framework/feather-sql) is Feather's SQL expression and query builder. It follows the composable builder approach of [Vapor SQLKit](https://github.com/vapor/sql-kit), but its role in the Feather architecture is deliberately narrower: it only builds a query and serializes it into SQL text plus bound values.

FeatherSQL does not connect to a database, start a transaction, execute a query, or decode a result set. It is database-connection-independent. A selected dialect decides how identifiers, placeholders, operators, and other SQL features are rendered. Connections, execution, transactions, and row decoding live in Feather Database and the driver packages; the higher-level DBAL layer builds typed access on top of those pieces.

This makes the boundary clear:

- **FeatherSQL** builds a dialect-specific SQL statement and its bindings.
- **Feather Database** provides the common connection and execution APIs.
- **Feather DBAL** adds typed tables, access operations, migrations, and row decoding.

## Choose a dialect

Create an `SQLBuilder` with the dialect supplied by the database driver. The same builder chain can then produce PostgreSQL, MySQL, or SQLite syntax:

```swift
import FeatherSQL
import FeatherSQLPostgres

let builder = SQLBuilder(dialect: PostgresDialect())
```

The dialect controls identifier quoting, placeholders, SQL types, and other database-specific details. The query itself stays composed from the common builder API. The same query structure can therefore be serialized for different SQL dialects.

## Select queries

Builders are immutable-style: each method returns an updated builder, so a query can be read from left to right:

```swift
let result = builder
    .select()
    .columns("id", "email")
    .from("users")
    .where("email", .equal, "user@example.com")
    .orderBy("id")
    .limit(20)
    .build()

print(result.sql)
// SELECT "id", "email" FROM "users"
// WHERE "email" = $1 ORDER BY "id" ASC LIMIT 20

print(result.bindings)
// ["user@example.com"]
```

Calling `build()` serializes the query for the selected dialect. The resulting `SQLBuildResult` contains both `sql` and `bindings`; pass both to the database connection instead of interpolating values into the SQL string.

### Columns and aliases

Use `column`, `columns`, and aliases to control the select list:

```swift
let result = builder
    .select()
    .column("name")
    .column("email", as: "contact_email")
    .column(SQLFunction("COUNT", args: SQLColumn("id")), as: "count")
    .from("users")
    .build()
```

Use `SQLIdentifier` when you need to construct an identifier expression explicitly, and `SQLLiteral.all` for `*`:

```swift
let result = builder
    .select()
    .column(SQLLiteral.all)
    .column(SQLIdentifier("name"), table: SQLIdentifier("users"))
    .from("users")
    .build()
```

### Conditions and bindings

Comparison values passed to `where` are bound parameters. Add multiple `where` conditions for `AND`, or use `orWhere` for `OR`:

```swift
let result = builder
    .select()
    .column("*")
    .from("users")
    .where("status", .equal, "active")
    .where("age", .greaterThanOrEqual, 18)
    .orWhere("role", .equal, "admin")
    .build()
```

For `IN`, pass an array and FeatherSQL creates a group of bindings:

```swift
let result = builder
    .select()
    .column("*")
    .from("users")
    .where("status", .in, ["active", "pending"])
    .build()
```

You can compare two columns instead of a column and a value:

```swift
let result = builder
    .select()
    .column("*")
    .from("users")
    .where("first_name", .notEqual, column: "last_name")
    .build()
```

Group complex predicates when the SQL needs parentheses:

```swift
let result = builder
    .select()
    .column("*")
    .from("users")
    .where {
        $0.where("status", .equal, "active")
            .orWhere("status", .equal, "pending")
    }
    .where("deleted_at", .is, SQLLiteral.null)
    .build()
```

For a range condition, use `SQLBetween`:

```swift
let result = builder
    .select()
    .column("*")
    .from("orders")
    .where(SQLBetween("total", between: 10, and: 100))
    .build()
```

### Ordering, grouping, and pagination

`orderBy`, `groupBy`, `having`, `limit`, and `offset` compose on the same builder:

```swift
let result = builder
    .select()
    .column("status")
    .column(SQLFunction("COUNT", args: SQLColumn("id")), as: "total")
    .from("orders")
    .groupBy("status")
    .having("status", .notEqual, "cancelled")
    .orderBy("total", .descending)
    .limit(20)
    .offset(40)
    .build()
```

## Joins and subqueries

Join tables with an explicit column comparison:

```swift
let result = builder
    .select()
    .column("users.email")
    .column("orders.total")
    .from("users")
    .join(
        "orders",
        on: SQLColumn("user_id", table: "orders"),
        .equal,
        SQLColumn("id", table: "users")
    )
    .where("orders.status", .equal, "paid")
    .build()
```

When both sides share a column name, a `USING` join is shorter:

```swift
let result = builder
    .select()
    .column("*")
    .from("users")
    .join("profiles", using: SQLIdentifier("user_id"))
    .build()
```

A subquery is an SQL expression and can be selected or used in a larger query:

```swift
let result = builder
    .select()
    .column("email")
    .from("users")
    .where(
        "id",
        .in,
        SQLSubquery.select { query in
            query
                .column("user_id")
                .from("orders")
                .where("total", .greaterThan, 100)
        }
    )
    .build()
```

## Insert, update, and delete

The same builder entry point supports the other data operations:

```swift
let insert = builder
    .insert(into: "users")
    .columns("email", "display_name")
    .values("user@example.com", "Ada")
    .build()

let update = builder
    .update("users")
    .set("display_name", to: "Ada Lovelace")
    .where("email", .equal, "user@example.com")
    .build()

let delete = builder
    .delete(from: "users")
    .where("email", .equal, "user@example.com")
    .build()
```

Values passed to `where`, `values`, and `set` are encoded as bindings. Lists can also be bound for comparisons such as `IN`, rather than being assembled into SQL text manually.

### Multiple rows and defaults

Use `SQLLiteral.default` for database-generated defaults and `rows` for more than one insert row:

```swift
let result = builder
    .insert(into: "users")
    .columns("id", "email", "created_at")
    .rows([
        [SQLLiteral.default, SQLBind("ada@example.com"), SQLLiteral.default],
        [SQLLiteral.default, SQLBind("grace@example.com"), SQLLiteral.default],
    ])
    .build()
```

### Returning rows

When the dialect supports it, add `returning` to receive inserted, updated, or deleted values from the database:

```swift
let result = builder
    .insert(into: "users")
    .columns("email")
    .values("ada@example.com")
    .returning("id", "email")
    .build()
```

### Upserts

Insert builders can ignore conflicts or update selected columns when a conflict occurs:

```swift
let result = builder
    .insert(into: "users")
    .columns("email", "display_name")
    .values("ada@example.com", "Ada")
    .onConflict(with: ["email"]) { conflict in
        conflict
            .set("display_name", to: "Ada Lovelace")
    }
    .build()
```

## Expressions and raw SQL

Use the specific builder methods and expressions whenever possible. `SQLFunction` represents a SQL function, `SQLLiteral` represents trusted SQL literals, and `SQLBind` represents a value that must become a parameter:

```swift
let expression = SQLFunction("LOWER", args: SQLColumn("email"))
let value = SQLBind("user@example.com")
```

`SQLRaw` is intended only for trusted SQL fragments such as a dialect-specific keyword. It does not escape its contents, and its legacy `binds` property is ignored. Never concatenate user input into `SQLRaw`.

For mixed trusted SQL syntax and bound values, `SQLQueryString` provides typed interpolation:

```swift
let query = "SELECT * FROM \(ident: "users") WHERE email = \(bind: "ada@example.com")" as SQLQueryString
let result = builder.build(query)
```

This separation keeps SQL structure and user data distinct and lets each dialect choose its own placeholder format.

## Build without executing

The builder result is data, not a live database operation:

```swift
let result = builder
    .select()
    .column("*")
    .from("users")
    .where("id", .equal, userID)
    .build()

// The adapter or DBAL layer executes these separately.
let sql = result.sql
let bindings = result.bindings
```

This is the key difference from a database client API. FeatherSQL can be used in a migration, DBAL access layer, test, or custom adapter without knowing how the connection was created.

## When to use this layer

Use the SQL builder when a query needs explicit joins, expressions, database features, or a shape that is not worth introducing as a typed DBAL operation. For normal table access, prefer the higher-level [Access Layer](/docs/dbal/access-layer/) guide: it adds typed columns, row decoding, and a stable application-facing API on top of this layer.
