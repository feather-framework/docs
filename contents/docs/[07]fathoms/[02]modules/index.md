---
category: "fathoms"
title: "Modules"
description: "Learn how to structure a FATHOMS module with Domain, Application, and Infrastructure layers."
order: 2
---

# Modules
---

A FATHOMS module is a self-contained business area inside a modular monolith.

For example, an application might have modules for accounts, authentication, billing, content, notifications, or system settings. Each module should own its own business rules, use cases, and persistence adapters.

This guide uses a simple **Account** module as the running example.

## Module shape

Each module follows the same basic structure:

```text
AccountDomain
AccountApplication
AccountInfrastructure
```

The dependency direction is always inward:

```text
Infrastructure -> Application -> Domain
```

That means:

- Domain contains the business meaning.
- Application contains use cases and DTOs.
- Infrastructure implements database and external service details.

## Kernel package

Most FATHOMS projects also have a small shared kernel package.

The kernel provides common contracts used by modules:

- Domain contracts such as `Model`, `Repository`, and `DomainError`.
- Application contracts such as `UseCase`, `DTO`, `Scope`, query executors, transaction executors, and authorization interfaces.
- Infrastructure contracts such as migration helpers, database executors, or shared adapter utilities.

Keep the kernel thin. It should hold common language and reusable patterns, not project-specific feature logic.

## Domain layer

The Domain layer describes business meaning.

It should avoid HTTP, SQL, database clients, mail clients, storage clients, and framework runtime details. A domain model should be understandable without knowing how the server is deployed.

### Domain model

An account model might look like this:

```swift
public struct Account: Model {
    public enum Error: DomainError {
        case emailTooShort
        case emailTooLong
        case passwordTooShort
    }

    public struct New: Sendable {
        public let id: String
        public let email: String
        public let passwordHash: String
        public let status: Status
    }

    public enum Status: String, Sendable {
        case pending
        case active
        case disabled
    }

    public let id: String
    public let email: String
    public let passwordHash: String
    public let status: Status

    public static func create(
        id: String,
        email: String,
        password: String,
        passwordHash: String
    ) throws(Self.Error) -> Self.New {
        try validate(email: email)
        try validate(password: password)

        return .init(
            id: id,
            email: email,
            passwordHash: passwordHash,
            status: .pending
        )
    }
}
```

The important point is not the exact fields. The important point is that business validation lives with the domain model.

### Repository port

Repository protocols are ports. They describe what persistence behavior the application needs, without saying how it is implemented.

```swift
public protocol AccountRepository: Repository {
    func findBy(id: String) async throws -> Account?
    func findBy(email: String) async throws -> Account?
    func insert(_ model: Account.New) async throws -> Account
    func update(_ model: Account) async throws -> Account
    func delete(id: String) async throws -> Bool
}
```

The domain knows that accounts can be found, inserted, updated, and deleted. It does not know whether the data lives in PostgreSQL, SQLite, MySQL, or an in-memory test store.

## Application layer

The Application layer runs business operations.

It contains:

- Use cases.
- Input and output DTOs.
- Query interfaces.
- Read and write scopes.
- Authorization checks.
- Unit-of-work boundaries.

Application depends on Domain and kernel Application contracts. It should not depend on Infrastructure details.

### Use case

```swift
public struct CreateAccount: UseCase {
    let transaction: any TransactionExecutor<WriteAccount>
    let idGenerator: any IDGenerator
    let passwordHasher: any PasswordHasher

    public struct Input: DTO {
        public let email: String
        public let password: String
    }

    public func execute(_ input: Input) async throws -> AccountDetail {
        let id = idGenerator.generate()
        let passwordHash = try await passwordHasher.hash(input.password)

        let account = try await transaction.run { context in
            let newAccount = try Account.create(
                id: id,
                email: input.email,
                password: input.password,
                passwordHash: passwordHash
            )

            return try await context.account.insert(newAccount)
        }

        return account.asDetail
    }
}
```

This use case:

1. Receives an input DTO.
2. Generates an ID.
3. Hashes the password through a port.
4. Creates domain input through domain validation.
5. Persists inside a transaction.
6. Returns an output DTO.

The use case does not know how the ID is generated, how the password is hashed, or how the database transaction is created.

### DTOs

A DTO is the shape exchanged at application boundaries.

```swift
public struct AccountDetail: DTO {
    public let id: String
    public let email: String
    public let status: Account.Status
    public let createdAt: Date
    public let updatedAt: Date
}
```

Use DTOs to return safe, intentional data. Do not return raw domain entities directly from public API handlers.

### Query interfaces

Query interfaces are read-side ports.

```swift
public protocol AccountQueries: Sendable {
    func list(query: AccountList.Query) async throws -> AccountList
    func count(query: AccountList.Query) async throws -> Int
    func getBy(id: String) async throws -> AccountDetail
}
```

Repositories usually handle model persistence. Queries usually handle read-only views and list screens.

### Scopes

Scopes group dependencies for a use case.

```swift
public struct WriteAccount: Scope {
    public let account: any AccountRepository
}

public struct ReadAccount: Scope {
    public let account: any AccountQueries
}
```

Use `TransactionExecutor<WriteAccount>` for state-changing use cases. Use `QueryExecutor<ReadAccount>` for read-only use cases.

## Infrastructure layer

Infrastructure implements the ports defined by Domain and Application.

This is where database code, row mapping, repository adapters, query adapters, and migrations live.

### SQL-first data access

FATHOMS works well with a SQL-first access layer.

The goal is explicit queries, predictable behavior, and easier performance tuning. The table layer owns row mapping. Repository and query adapters translate between database rows, domain models, and DTOs.

### Table helper

```swift
struct AccountTable {
    let connection: any DatabaseConnection

    func find(id: String) async throws -> Row? {
        try await connection.run(
            query: #"""
                SELECT *
                FROM account
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

This table helper owns SQL and row decoding. It does not return API schemas.

### Repository adapter

```swift
public struct DatabaseAccountRepository: AccountRepository {
    let connection: any DatabaseConnection

    public func insert(_ model: Account.New) async throws -> Account {
        let table = AccountTable(connection: connection)
        let row = try await table.insert(
            .init(
                id: model.id,
                email: model.email,
                passwordHash: model.passwordHash,
                status: model.status.rawValue
            )
        )
        return try row.asDomain
    }
}
```

The adapter converts domain input into database rows, then converts stored rows back into domain models.

### Query adapter

```swift
public struct DatabaseAccountQueries: AccountQueries {
    let connection: any DatabaseConnection

    public func getBy(id: String) async throws -> AccountDetail {
        let table = AccountTable(connection: connection)

        guard let row = try await table.find(id: id) else {
            throw RepositoryError.notFound
        }

        return row.asDetail
    }
}
```

The query adapter returns DTOs for read-side use cases.

## End-to-end flow

For `CreateAccount`, the flow is:

```text
HTTP request
   |
   v
API handler
   |
   v
CreateAccount use case
   |
   v
AccountRepository port
   |
   v
DatabaseAccountRepository adapter
   |
   v
SQL query execution
   |
   v
AccountDetail DTO response
```

Each step has one responsibility. This is what keeps the module understandable.

## Naming conventions

Consistent naming helps developers move between modules.

Recommended conventions:

- Use cases: `CreateX`, `UpdateX`, `GetX`, `ListX`, `DeleteX`.
- Repository lookups: `findBy...`.
- Query methods: `get...`, `list...`, `count...`.
- Scopes: `ReadX`, `WriteX`.
- DTOs: `XDetail`, `XList`, `XSummary`, `XCreateInput`, `XUpdateInput`.

The exact verbs can be adjusted to your project, but keep them consistent once chosen.

## Feature checklist

When adding one new module feature:

1. Add or update the domain model rules.
2. Add repository and query interfaces.
3. Add use case input and output DTOs.
4. Add or extend read and write scopes.
5. Add permissions and actions when access control is required.
6. Implement table helpers, query adapters, and repository adapters.
7. Add migrations and seed data when needed.
8. Wire the use case in the module builder.
9. Add or update the API handler.

Following this checklist keeps new features aligned with the rest of the system.
