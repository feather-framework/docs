---
category: "openapi"
title: "Server Integration"
description: "Learn how OpenAPI specifications connect to a Swift server implementation."
order: 3
---

# Server Integration
---

OpenAPI is most useful when the specification is connected to the server implementation.

In a typical Swift backend, the flow looks like this:

```text
OpenAPI document
Generated Swift package
API implementation
Hummingbird or Vapor router
```

Feather OpenAPI helps with the specification side. Swift OpenAPI Generator can then generate Swift protocol types and operation models from that specification.

## Generated API packages

For example, an application server can depend on generated OpenAPI packages:

```swift
.product(
    name: "ExampleAppOpenAPI",
    package: "example-openapi"
),
.product(
    name: "ExampleAdminOpenAPI",
    package: "example-openapi"
),
```

Those packages provide API protocols and generated request and response types.

For a Hummingbird server, add the OpenAPI runtime and Hummingbird transport:

```swift
.package(
    url: "https://github.com/apple/swift-openapi-runtime",
    from: "1.13.2"
),
.package(
    url: "https://github.com/hummingbird-project/swift-openapi-hummingbird",
    from: "2.0.1"
),
```

Then add the generated API package, runtime, and transport products to the server target:

```swift
.product(
    name: "ExampleAppOpenAPI",
    package: "example-openapi"
),
.product(
    name: "OpenAPIRuntime",
    package: "swift-openapi-runtime"
),
.product(
    name: "OpenAPIHummingbird",
    package: "swift-openapi-hummingbird"
),
```

The server implementation can then conform to the generated protocol:

```swift
import OpenAPIRuntime
import ExampleAppOpenAPI

struct AppAPI: APIProtocol {
    let modules: AppModules
}
```

Each operation can be implemented in a focused file. This keeps the API surface organized as the number of endpoints grows.

## Register handlers

For Hummingbird, generated OpenAPI handlers can be registered on the router:

```swift
import Hummingbird
import OpenAPIHummingbird

let router = Router(context: AppRequestContext.self)

let middlewares: [any ServerMiddleware] = [
    ErrorMiddleware(),
    AuthSubjectMiddleware(modules: modules),
]

let appAPI = AppAPI(modules: modules)
try appAPI.registerHandlers(
    on: router,
    middlewares: middlewares
)
```

This connects generated OpenAPI operations to your server framework.

## Works with Hummingbird and Vapor

Feather Framework is not tied to one HTTP server.

The OpenAPI specification belongs to the application contract. The HTTP server layer can be Hummingbird or Vapor, depending on what your project uses. Feather Framework is designed to work with both server frameworks.

Hummingbird and Vapor handle routing, middleware, request handling, and the HTTP server lifecycle. Feather Framework sits at a higher application-organization level and helps you structure the reusable pieces around that server layer.

## Testing OpenAPI routes

OpenAPI routes can be tested through Feather HTTP testing executors.

```swift
let executor = HummingbirdTestingHTTPExecutor(
    app: server,
    testingSetup: .router
)

try await executor.withOperation(
    request: Request(
        method: .get,
        path: "/health"
    )
) { response in
    #expect(response.status == .ok)
}
```

For JSON endpoints, decode the response using the generated schema types:

```swift
let output = try await response.json(
    status: .ok,
    LoginOutput.self
)
```

This makes tests read like real API calls while still exercising the generated OpenAPI handlers, middleware, and server routing.

## Recommended structure

For larger APIs, keep the generated API conformance small:

- One root API type per public API surface.
- One file per endpoint or operation group.
- DTO-to-schema adapters near the server layer.
- Business logic in application modules.
- Persistence and external services behind application protocols.

This keeps OpenAPI integration useful without letting generated transport details leak into the core application logic.
