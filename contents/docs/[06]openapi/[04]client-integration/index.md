---
category: "openapi"
title: "Client Integration"
description: "Learn how to use a generated OpenAPI client from Swift application code."
order: 4
---

# Client Integration
---

OpenAPI is not only useful on the server. The same OpenAPI document can also generate a type-safe Swift client.

This is useful when a web app, iOS app, command-line tool, worker, or another backend service needs to call your API. Instead of writing request paths, headers, query parameters, and response decoding by hand, the generated client gives you Swift methods and generated input and output types.

## How the generated client works

Swift OpenAPI Generator creates a `Client` type for each generated API package.

That client needs two things:

- A `serverURL`, which is the base URL of the API.
- A `ClientTransport`, which performs the actual HTTP request.

For Apple platforms and simple tools, `OpenAPIURLSession` is a good default transport. For server-side Swift services, `OpenAPIAsyncHTTPClient` is usually a better fit.

## Package setup

Add the URLSession transport package. If you want to use client middleware, also depend on the OpenAPI runtime and HTTP types packages:

```swift
.package(
    url: "https://github.com/apple/swift-openapi-urlsession",
    from: "1.0.0"
),
.package(
    url: "https://github.com/apple/swift-openapi-runtime",
    from: "1.0.0"
),
.package(
    url: "https://github.com/apple/swift-http-types",
    from: "1.0.0"
),
```

Then add the generated OpenAPI package, runtime, HTTP types, and transport product to your target:

```swift
.target(
    name: "ExampleClient",
    dependencies: [
        .product(
            name: "ExampleAppOpenAPI",
            package: "example-openapi"
        ),
        .product(
            name: "OpenAPIURLSession",
            package: "swift-openapi-urlsession"
        ),
        .product(
            name: "OpenAPIRuntime",
            package: "swift-openapi-runtime"
        ),
        .product(
            name: "HTTPTypes",
            package: "swift-http-types"
        ),
    ]
)
```

The generated package name depends on your project. In this guide, `ExampleAppOpenAPI` stands for the Swift package generated from your OpenAPI document.

## Create a client

```swift
import ExampleAppOpenAPI
import Foundation
import OpenAPIRuntime
import OpenAPIURLSession

let client = Client(
    serverURL: URL(string: "https://api.example.com")!,
    transport: URLSessionTransport()
)
```

`Client` comes from the generated OpenAPI package. `URLSessionTransport` comes from `OpenAPIURLSession`.

## Call an endpoint

Generated clients provide one Swift method per OpenAPI operation.

For example, if the OpenAPI document contains a `GET /me` operation with the operation identifier `userAuthMe`, the generated client can call it like this:

```swift
let response = try await client.userAuthMe()

switch response {
case .ok(let ok):
    let account = try ok.body.json
    print(account.email)

case .unauthorized:
    print("The user is not signed in.")

case .forbidden:
    print("The user is not allowed to access this resource.")

case .undocumented(let statusCode, _):
    print("Unexpected response: \(statusCode)")
}
```

The exact cases depend on the responses described in your OpenAPI document. This is one of the main benefits of generated clients: documented responses become typed Swift cases.

## Send a request body

For endpoints with request bodies, use the generated input and schema types.

```swift
let input = Operations.CreateAccount.Input(
    body: .json(
        .init(
            email: "taylor@example.com",
            password: "correct-horse-battery-staple"
        )
    )
)

let response = try await client.createAccount(input)

switch response {
case .created(let created):
    let account = try created.body.json
    print(account.id)

case .badRequest(let problem):
    print(problem)

case .undocumented(let statusCode, _):
    print("Unexpected response: \(statusCode)")
}
```

The generated input type protects you from sending the wrong body shape. If the OpenAPI document changes, the compiler helps find the affected call sites.

## Add authorization

Many clients need to send an access token. If one authenticated request becomes many authenticated requests, prefer a client middleware. It keeps authorization in one place and applies it before the transport sends the request.

```swift
import Foundation
import HTTPTypes
import OpenAPIRuntime

struct BearerTokenMiddleware: ClientMiddleware {
    let accessToken: String

    func intercept(
        _ request: HTTPRequest,
        body: HTTPBody?,
        baseURL: URL,
        operationID: String,
        next: @Sendable (HTTPRequest, HTTPBody?, URL) async throws -> (HTTPResponse, HTTPBody?)
    ) async throws -> (HTTPResponse, HTTPBody?) {
        var request = request
        request.headerFields[.authorization] = "Bearer \(accessToken)"
        return try await next(request, body, baseURL)
    }
}

let client = Client(
    serverURL: URL(string: "https://api.example.com")!,
    transport: URLSessionTransport(),
    middlewares: [
        BearerTokenMiddleware(accessToken: accessToken)
    ]
)
```

You can still pass generated header values directly when an operation defines them in the OpenAPI document. Middleware is simply the cleaner choice for shared headers such as `Authorization`.

For larger clients, prefer a small API client wrapper so the rest of your app does not need to know about transport details:

```swift
struct AccountAPI {
    let client: Client

    func currentAccount() async throws -> Components.Schemas.Account {
        let response = try await client.userAuthMe()

        switch response {
        case .ok(let ok):
            return try ok.body.json
        case .unauthorized:
            throw APIError.unauthorized
        case .forbidden:
            throw APIError.forbidden
        case .undocumented(let statusCode, _):
            throw APIError.unexpectedStatus(statusCode)
        }
    }
}
```

This keeps generated response handling close to the API boundary and lets the rest of the application work with simpler methods.

## Server-side client transport

For server-side Swift code, you may prefer the Async HTTP Client transport:

```swift
.package(
    url: "https://github.com/swift-server/swift-openapi-async-http-client",
    from: "1.0.0"
),
```

Add the transport product to the client target:

```swift
.product(
    name: "OpenAPIAsyncHTTPClient",
    package: "swift-openapi-async-http-client"
),
```

Then use:

```swift
import ExampleAppOpenAPI
import Foundation
import OpenAPIAsyncHTTPClient

let client = Client(
    serverURL: URL(string: "https://api.example.com")!,
    transport: AsyncHTTPClientTransport()
)
```

Use `OpenAPIURLSession` for a simple Foundation-based client. Use `OpenAPIAsyncHTTPClient` when the client runs inside a server-side Swift service and you want SwiftNIO-based HTTP behavior.

## Recommended structure

Keep generated OpenAPI client code at the edge of your application.

A good structure is:

```text
Application feature code
   |
   v
Small API wrapper
   |
   v
Generated OpenAPI Client
   |
   v
URLSession or AsyncHTTPClient transport
```

This keeps business logic independent from generated transport details while still giving you the safety of generated request and response types.

Sources: [Swift OpenAPI Generator](https://github.com/apple/swift-openapi-generator), [OpenAPIURLSession](https://github.com/apple/swift-openapi-urlsession), and [OpenAPIAsyncHTTPClient](https://github.com/swift-server/swift-openapi-async-http-client).
