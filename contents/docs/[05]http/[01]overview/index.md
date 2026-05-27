---
category: "http"
title: "Overview"
description: "Learn what Feather HTTP is and how it helps with clients, testing, and server-side Swift HTTP work."
order: 1
---

# Overview
---

[Feather HTTP](https://github.com/feather-framework/feather-http) is an HTTP abstraction for Swift applications.

It gives your application a common way to describe outgoing HTTP requests and handle responses, while the actual transport is provided by an executor such as Async HTTP Client, URLSession, or a testing executor.

The HTTP section is split into three practical areas:

- **Clients**: making real outgoing HTTP requests.
- **Testing**: exercising Hummingbird or Vapor applications without opening a public network port.
- **Server**: setting up a basic Hummingbird or Vapor `Server` target.

## Why use Feather HTTP?

Backend applications often need to call other services:

- Payment providers.
- Authentication services.
- Internal APIs.
- Webhooks.
- File upload endpoints.
- Generated OpenAPI clients.

If every integration uses a different HTTP client directly, request code can become inconsistent and hard to test.

Feather HTTP keeps the request and response model consistent. Your code builds a request, passes it to an `HTTPExecutor`, and handles the response through `ResponseContext`.

## The executor

The core protocol is `HTTPExecutor`:

```swift
public protocol HTTPExecutor: Sendable {
    func execute(
        req: HTTPRequest,
        body: HTTPBody
    ) async throws -> (response: HTTPResponse, body: HTTPBody)
}
```

Most application code uses `withOperation`, which handles request conversion and response wrapping:

```swift
let todo = try await executor.withOperation(
    request: Request(
        method: .get,
        path: "/todos/1"
    )
) { response in
    try await response.json(Todo.self)
}
```

## Request types

Feather HTTP includes several request helpers:

- `Request` for direct HTTP request metadata and body.
- `JSONRequest` for JSON-encoded request bodies.
- `BinaryRequest` for binary payloads.
- `MultipartRequest` for form-data uploads.
- `RequestRepresentable` for custom request types.

Example JSON request:

```swift
struct CreateTodo: Encodable, Sendable {
    var title: String
}

try await executor.withOperation(
    request: JSONRequest(
        path: "/todos",
        body: CreateTodo(title: "Buy milk")
    )
) { response in
    try await response.json(status: .created, Todo.self)
}
```

`JSONRequest` automatically applies the `application/json` content type and encodes the body.

## Response helpers

`ResponseContext` provides helpers for common response work:

```swift
let status = response.status
let token = response.bearerToken()
let contentType = response.headerValue(.contentType)
let values = response.headerValues(.cacheControl)

let text = try await response.plainTextBody()
let data = try await response.dataBody()
let object = try await response.jsonBody(Todo.self)
```

For JSON APIs, prefer `json(status:_:)`:

```swift
let value = try await response.json(
    status: .ok,
    Todo.self
)
```

This validates the expected status code and checks that the response content type is JSON before decoding.

## Request options

Per-request options let you control timeout and response body limits:

```swift
try await executor.withOperation(
    options: .init(
        responseMaxBodyBytes: 1024 * 1024,
        timeout: .seconds(10)
    ),
    request: Request(method: .get, path: "/slow")
) { response in
    try await response.plainTextBody()
}
```

Body limits are important because tests and integrations should not accidentally buffer an unlimited response into memory.

## Available executors

The currently available executor implementations are:

- [Async HTTP Client](https://github.com/feather-framework/feather-http-ahc)
- [URLSession](https://github.com/feather-framework/feather-http-urlsession)
- [Hummingbird Testing](https://github.com/feather-framework/feather-http-hummingbird-testing)
- [Vapor Testing](https://github.com/feather-framework/feather-http-vapor-testing)

Use a real network executor for production integrations, a testing executor when you want to exercise your server without opening a real network port, and the server guide when you need a basic executable target for Hummingbird or Vapor.
