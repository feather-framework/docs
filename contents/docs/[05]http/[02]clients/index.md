---
category: "http"
title: "Clients"
description: "Learn how to use the Async HTTP Client and URLSession executors for real HTTP requests."
order: 2
---

# Clients
---

Feather HTTP separates the request API from the network transport. For real outgoing client requests, use either the Async HTTP Client executor or the URLSession executor.

## Async HTTP Client

[Feather HTTP AHC](https://github.com/feather-framework/feather-http-ahc) provides `AHCHTTPExecutor`, backed by Swift server-side [Async HTTP Client](https://github.com/swift-server/async-http-client).

It is a good default for server-side Swift applications.

Add the package:

```swift
.package(
    url: "https://github.com/feather-framework/feather-http-ahc",
    exact: "1.0.0-beta.1"
),
```

Then add the product:

```swift
.product(
    name: "FeatherHTTPAsyncHTTPClient",
    package: "feather-http-ahc"
),
```

```swift
import AsyncHTTPClient
import FeatherHTTP
import FeatherHTTPAsyncHTTPClient

let httpClient = HTTPClient(
    eventLoopGroupProvider: .singleton
)

let executor = AHCHTTPExecutor(
    client: httpClient,
    baseURL: URL(string: "https://api.example.com"),
    requestTimeout: .seconds(30)
)
```

Create the `HTTPClient` once during application startup and manage it with the application's service lifecycle. Reuse the same executor for requests instead of creating a new network client for every operation.

Then run requests through the shared API:

```swift
let account = try await executor.withOperation(
    request: Request(
        method: .get,
        path: "/account",
        headerFields: [
            .authorization: "Bearer \(token)"
        ]
    )
) { response in
    try await response.json(Account.self)
}
```

`AHCHTTPExecutor` can also resolve absolute URLs, or requests with explicit `scheme` and `authority`.

## URLSession

[Feather HTTP URLSession](https://github.com/feather-framework/feather-http-urlsession) provides `URLSessionHTTPExecutor`, backed by Foundation `URLSession`.

This is useful when you want a Foundation-based transport or when your code also needs to run in non-server contexts.

Add the package:

```swift
.package(
    url: "https://github.com/feather-framework/feather-http-urlsession",
    exact: "1.0.0-beta.1"
),
```

Then add the product:

```swift
.product(
    name: "FeatherHTTPURLSession",
    package: "feather-http-urlsession"
),
```

```swift
import FeatherHTTP
import FeatherHTTPURLSession

let executor = URLSessionHTTPExecutor(
    session: .shared,
    baseURL: URL(string: "https://api.example.com")
)
```

The request code stays the same:

```swift
let status = try await executor.withOperation(
    request: JSONRequest(
        path: "/events",
        body: EventPayload(name: "signed_up")
    )
) { response in
    response.status
}
```

## Choosing between them

Use `AHCHTTPExecutor` when you are building a server-side Swift service and already use SwiftNIO-based infrastructure.

Use `URLSessionHTTPExecutor` when you prefer Foundation networking, need platform integration, or want a simpler setup for smaller tools.

Both executors implement the same `HTTPExecutor` protocol, so application code can depend on `any HTTPExecutor` and keep the concrete transport in configuration.

## Timeouts and body limits

Both real request executors collect request and response bodies up to configured limits.

```swift
let executor = AHCHTTPExecutor(
    client: httpClient,
    baseURL: URL(string: "https://api.example.com"),
    maxBodyBytes: 16 * 1024 * 1024,
    requestTimeout: .seconds(30)
)
```

You can override timeout and response body size per request:

```swift
try await executor.withOperation(
    options: .init(
        responseMaxBodyBytes: 512 * 1024,
        timeout: .seconds(5)
    ),
    request: Request(method: .get, path: "/status")
) { response in
    try await response.plainTextBody()
}
```

Use smaller limits for endpoints where you know the response should be small. It makes bugs easier to catch and protects the process from unexpectedly large responses.
