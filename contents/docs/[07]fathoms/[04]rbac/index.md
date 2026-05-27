---
category: "fathoms"
title: "RBAC"
description: "Learn how to model role-based access control in a FATHOMS application."
order: 4
---

# RBAC
---

RBAC means **Role-Based Access Control**.

It is a simple way to decide what an authenticated user is allowed to do. Instead of checking permissions directly against every user account, you group permissions into roles, then assign those roles to users.

For example:

```text
Editor role
  - article:create
  - article:list
  - article:update

Viewer role
  - article:list
```

In a FATHOMS application, RBAC belongs mostly to the Application layer. Use cases should ask for authorization before they perform protected work. The HTTP server should identify the current user, but the use case should decide whether that user is allowed to perform the business operation.

## Core terms

RBAC is easier to understand when the words are precise.

- A **subject** is the actor making the request. In most applications, this is the authenticated account.
- A **role** is a named group of permissions, such as `admin`, `editor`, or `viewer`.
- A **permission** is one allowed capability, such as `article:create` or `article:update`.
- An **action** is the operation a use case wants to perform.
- An **authorizer** checks whether the subject can perform the action.

The typical flow looks like this:

```text
HTTP request
   |
   v
Authenticated subject
   |
   v
Use case action
   |
   v
Authorizer checks roles and permissions
   |
   v
Use case continues or throws forbidden
```

## Define permission keys

Start by defining the permissions owned by a module.

For a simple article module, the permissions might look like this:

```swift
public enum ArticlePermissions: PermissionProvider {

    public enum Articles: PermissionProvider {
        static let create = PermissionKey("article:create")
        static let list = PermissionKey("article:list")
        static let update = PermissionKey("article:update")
        static let delete = PermissionKey("article:delete")

        public static func allPermissions() -> Set<PermissionKey> {
            [
                create,
                list,
                update,
                delete,
            ]
        }
    }

    public static func allPermissions() -> Set<PermissionKey> {
        Articles.allPermissions()
    }
}
```

This gives every protected capability a stable name.

Use clear permission names. A good permission key should describe the resource and the action. `article:update` is easier to understand than `edit` because it tells you what can be edited.

## Model actions in use cases

A permission key is only the stored value. The use case should check an action.

For simple permission checks, an action can map directly to one permission key:

```swift
public struct ListArticles: UseCase {

    struct Action: PermissionAction {
        let key = ArticlePermissions.Articles.list
    }

    let authorizer: any Authorizer
    let query: any QueryExecutor<ReadArticle>

    public struct Input: DTO {
        public let query: ArticleList.Query

        public init(query: ArticleList.Query) {
            self.query = query
        }
    }

    public func execute(
        subject: Subject,
        input: Input
    ) async throws -> ArticleList {
        guard try await authorizer.can(
            subject: subject,
            perform: Action()
        ) else {
            throw AuthError(
                kind: .forbidden,
                message: Action().key.rawValue
            )
        }

        return try await query.run { context in
            try await context.article.list(query: input.query)
        }
    }
}
```

This keeps the authorization rule close to the business operation. If someone reads `ListArticles`, they can immediately see which permission is required.

## Use custom actions for business rules

Not every authorization rule is a direct permission lookup.

Sometimes the subject needs a permission and a business-specific condition. For example, an author may update their own draft, while an editor may update any article.

That can be modeled as a custom action:

```swift
struct UpdateArticleAction: AuthAction {
    let articleAuthorId: String

    func authorize(
        subject: Subject,
        permissions: Set<PermissionKey>
    ) async throws -> Bool {
        if permissions.contains(ArticlePermissions.Articles.update) {
            return true
        }

        return subject.id == articleAuthorId
    }
}
```

Use this carefully. Most checks should stay permission-based. Custom actions are useful when the rule really depends on the subject, the resource, or both.

## Resolve permissions through an authorizer

The authorizer is responsible for turning a subject into a set of permissions.

In practice, it usually does this:

1. Load the subject's roles.
2. Load the permissions attached to those roles.
3. Ask the action whether those permissions are enough.

A simplified authorizer looks like this:

```swift
public struct DefaultAuthorizer: Authorizer {
    let query: any QueryExecutor<AuthScope>

    public func can(
        subject: Subject,
        perform action: any AuthAction
    ) async throws -> Bool {
        try await query.run { context in
            let roles = try await context.account.getRolesBy(
                accountId: subject.id
            )

            let permissions = try await context.rolePermissions.permissions(
                for: Set(roles)
            )

            return try await action.authorize(
                subject: subject,
                permissions: Set(permissions.map(PermissionKey.init))
            )
        }
    }
}
```

The use case does not need to know how roles and permissions are stored. It only depends on the `Authorizer` protocol.

That is the important FATHOMS rule: business code depends on clear application contracts, not database details.

## Store roles and permissions

A common RBAC database model uses five tables:

```text
account
role
account_role
role_permission
permission
```

The relationships are:

- One account can have many roles.
- One role can belong to many accounts.
- One role can have many permissions.
- One permission can belong to many roles.

In SQL terms, this usually means two join tables:

```sql
CREATE TABLE account_role (
    account_id TEXT NOT NULL,
    role_id TEXT NOT NULL,
    PRIMARY KEY (account_id, role_id)
);

CREATE TABLE role_permission (
    role_id TEXT NOT NULL,
    permission_id TEXT NOT NULL,
    PRIMARY KEY (role_id, permission_id)
);
```

Your access layer can then expose focused methods:

```swift
public protocol AccountQueries: Sendable {
    func getRolesBy(accountId: String) async throws -> [String]
}

public protocol RolePermissionQueries: Sendable {
    func permissions(for roles: Set<String>) async throws -> [String]
}
```

This is enough for the authorizer to answer the question, "Can this subject perform this action?"

## Keep authorization out of HTTP handlers

HTTP handlers should usually do three things:

1. Decode request input.
2. Resolve the current subject.
3. Call the use case.

The handler should not duplicate permission rules.

```swift
func articleList(
    _ input: Operations.ArticleList.Input
) async throws -> Operations.ArticleList.Output {
    let subject = try await CurrentSubject.require()

    let result = try await modules.article.listArticles().execute(
        subject: subject,
        input: .init(query: input.query.asArticleListQuery)
    )

    return .ok(.init(body: .json(result.schema)))
}
```

This keeps the server layer thin. The authorization rule remains in the Application layer, where it can be tested without starting Hummingbird, Vapor, or an OpenAPI server.

## Register permissions

Most applications need a way to seed or synchronize known permissions.

Because each module can provide its own permissions, the server composition layer can gather them during startup:

```swift
let permissions: Set<PermissionKey> =
    ArticlePermissions.allPermissions()
        .union(UserPermissions.allPermissions())
        .union(SystemPermissions.allPermissions())
```

You can then store missing permissions in the database through a migrator, setup command, or administrative task.

The important rule is that permissions should be defined in code first. The database stores assignments, but the application code owns the list of capabilities it understands.

## Testing RBAC

RBAC is easiest to test at the use-case level.

Use a test authorizer that returns the result you need:

```swift
struct TestAuthorizer: Authorizer {
    let isAllowed: Bool

    func can(
        subject: Subject,
        perform action: any AuthAction
    ) async throws -> Bool {
        isAllowed
    }
}
```

Then test both paths:

- An allowed subject receives the expected result.
- A denied subject receives a forbidden error.

You can also test the real authorizer with an in-memory or test database. That is useful for verifying role and permission queries, but most use-case tests should stay small and direct.

## Practical guidelines

Keep RBAC boring and explicit.

- Define permission keys near the module that owns the behavior.
- Check authorization inside use cases, not only in HTTP middleware.
- Use roles for assignment, but use permissions for checks.
- Keep permission names stable once they are used in production.
- Prefer simple `PermissionAction` checks for most use cases.
- Use custom `AuthAction` types only when the rule depends on the subject or resource.
- Return `unauthorized` when there is no authenticated subject.
- Return `forbidden` when the subject exists but is not allowed to perform the action.

With this structure, FATHOMS gives RBAC a clear home. The server identifies the subject, the use case declares the action, the authorizer resolves permissions, and the database stores role assignments.
