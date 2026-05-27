---
category: "http"
title: "Testing"
description: "Learn how to use Feather HTTP testing executors with Hummingbird, Vapor, and server tests."
order: 3
---

# Testing
---

Feather HTTP is especially useful in server tests because the same request and response helpers can exercise a real application without depending on a live external server.

For example, a Hummingbird application can use this pattern in its test suite:

```swift
import FeatherHTTP
import FeatherHTTPHummingbirdTesting

let executor = HummingbirdTestingHTTPExecutor(
    app: server,
    testingSetup: .router
)

let result = try await executor.withOperation(
    request: Request(
        method: .get,
        path: "/health"
    )
) { response in
    response.status
}
```

## Hummingbird testing

[Feather HTTP Hummingbird Testing](https://github.com/feather-framework/feather-http-hummingbird-testing) adapts `hummingbird-testing` to the `HTTPExecutor` protocol.

It sends requests into a Hummingbird application under test and returns a normal Feather HTTP response context.

This means your tests can use the same helpers as real HTTP integrations:

```swift
try await executor.withOperation(
    request: JSONRequest(
        path: "/admin/auth/login",
        body: LoginRequest(email: "admin@example.com")
    )
) { response in
    let output = try await response.json(
        status: .ok,
        LoginResponse.self
    )

    #expect(output.token.isEmpty == false)
}
```

## Build a small test runner

A common pattern is to wrap the executor in a test helper:

```swift
struct TestRunner {
    let server: any ApplicationProtocol

    @discardableResult
    func run<Output>(
        request: some RequestRepresentable,
        _ block: @escaping @Sendable (ResponseContext) async throws -> Output
    ) async throws -> Output {
        let executor = HummingbirdTestingHTTPExecutor(
            app: server,
            testingSetup: .router
        )

        return try await executor.withOperation(
            request: request,
            response: block
        )
    }
}
```

Then tests stay short:

```swift
try await runner.run(
    request: Request(method: .get, path: "/health")
) { response in
    #expect(response.status == .ok)
}
```

This is beginner-friendly because each test reads like an HTTP call, but it still exercises your real router, middleware, and OpenAPI handlers.

## Vapor testing

[Feather HTTP Vapor Testing](https://github.com/feather-framework/feather-http-vapor-testing) follows the same idea for Vapor applications.

Use the Vapor testing executor when your server layer runs on Vapor and you want to test routes through the common Feather HTTP API.

```swift
import FeatherHTTP
import FeatherHTTPVaporTesting
import Testing
import Vapor

@Test
func healthRouteReturnsOK() async throws {
    let app = try await Application.make(.testing)

    app.routes.get("health") { _ async throws -> HTTPStatus in
        .ok
    }

    do {
        let executor = VaporTestingHTTPExecutor(app: app)

        try await executor.withOperation(
            request: Request(
                method: .get,
                path: "/health"
            )
        ) { response in
            #expect(response.status == .ok)
        }

        try await app.asyncShutdown()
    }
    catch {
        try? await app.asyncShutdown()
        throw error
    }
}
```

You can use the same request helpers here as in the Hummingbird example. For example, `JSONRequest` can send a JSON payload, and `ResponseContext` can validate status, headers, and decoded response bodies.

## Why this testing style works well

This approach keeps tests close to how clients use your application:

- The test sends a real HTTP request shape.
- Middleware and routing still run.
- Responses are checked through status, headers, and body helpers.
- JSON decoding uses the same types your application expects.
- The server does not need to bind to a public port.

It also keeps the test code independent from Hummingbird or Vapor-specific response assertion APIs.
