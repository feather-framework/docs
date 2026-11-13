---
category: "database"
title: "Overview"
description: "Learn what Feather Database is, why it is useful, and how it helps you work with SQL databases in Swift."
order: 1
---

# Overview
---

[Feather Database](https://github.com/feather-framework/feather-database) is a database abstraction layer for Swift. It gives your application a clear and consistent way to talk to SQL databases while still letting you write SQL when you need it.

The database abstraction library is designed for modern Swift concurrency. This means it works well with `async` and `await`, and it helps you build database code that is easier to read, test, and maintain.

## Why use Feather Database?

Most applications need a database, but database code can quickly become repetitive. Each driver has its own connection type, query API, result format, and small differences in behavior. Feather Database reduces that complexity by providing one common database layer.

With Feather Database, you can use the same basic patterns across supported databases. Your code can stay focused on your application logic instead of driver-specific details.

This is useful because it helps you:

- Set up database access with less boilerplate.
- Use one standard approach for queries, results, connections, and transactions.
- Keep your code easier to move between database engines.
- Write safer SQL with consistent parameter handling.
- Decode database rows in a predictable way.

## A database layer, not an ORM

Feather Database is intentionally not an [ORM](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping).

An ORM usually tries to map your database tables directly to Swift objects and hide much of the SQL behind a higher-level model system. That can be useful in some projects, but it can also make complex database behavior harder to see and control.

Feather Database takes a different approach. It gives you a clean database layer while still keeping SQL visible and explicit. You remain in control of the queries, joins, indexes, and database-specific decisions that matter for performance and correctness.

In simple terms: Feather Database helps you write better database code without taking the database away from you.

## One interface for multiple drivers

Feather Database provides a database-agnostic abstraction layer. This means your application can depend on common interfaces instead of depending directly on a single database driver.

The currently available drivers are:

- SQLite
- MySQL and MariaDB
- PostgreSQL

This shared abstraction layer API is one of the main benefits of the library. It gives your project a consistent structure whether you are using SQLite for local development, MySQL or MariaDB for a production service, or PostgreSQL for a larger backend system.

Please note that while the Swift API remains consistent, choosing a specific SQL dialect can introduce compatibility issues when switching drivers, as SQL expressions and supported features may vary between them.

## Safer SQL parameters

Feather Database uses Swift string interpolation for SQL parameters. This gives you a convenient way to write queries while keeping parameter encoding uniform across drivers.

String interpolation acts as a safety mechanism against SQL injection attacks by ensuring that values are passed as parameters and properly encoded by the database layer, rather than being manually concatenated into raw SQL strings.

For developers, this keeps query code readable. For applications, it provides a safer default way to handle dynamic values.

## Standard result decoding

Database drivers often return rows in slightly different formats. Feather Database provides a standard way to decode query results so the rest of your application does not need to care about those differences.

This makes result handling more predictable. Whether your query returns one row, many rows, or no rows, you can use a consistent structure for reading the response and converting it into the values your application needs.

## Streaming results

Some queries can return a large number of rows. Loading all of them into memory at once is not always the best option.

Feather Database supports result streaming, which lets your application process rows in a structured way as they are returned by the database. This can make large queries more efficient and can help keep memory usage under control.

## Connection pooling

Opening a new database connection for every query is expensive. The concrete database drivers can provide connection pooling, so connections can be reused safely while the common Feather Database API stays the same.

A connection pool manages a set of active database connections for your application. When a query needs to run, the pool provides an available connection. When the query is finished, the connection can be returned to the pool and reused later.

This improves performance and gives your application a more reliable way to handle many database operations.

## Transaction support

Transactions let you group multiple database operations into one unit of work. If every operation succeeds, the transaction can be committed. If something fails, the transaction can be rolled back.

Feather Database includes structured transaction support, which helps keep these operations clear and safe. This is especially important when your application needs to update related data and must avoid leaving the database in a half-finished state.

## Documentation and quality

Feather Database includes DocC-based API documentation, unit tests, and code coverage. These help make the framework easier to learn and safer to use in real projects.

The goal is to provide a practical database foundation for Swift applications: simple enough to understand, flexible enough for real SQL work, and structured enough to keep your application code clean over time.

In the following guides, we will explore how to use Feather Database in a real-world server-side Swift application.

