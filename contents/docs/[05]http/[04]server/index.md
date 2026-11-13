---
category: "http"
title: "Server"
description: "Learn how to create a basic Server executable target with Hummingbird or Vapor."
order: 4
---

# Server
---

A server target is the executable that starts your HTTP application.

In a Feather Framework project, the server target usually stays thin. It loads configuration, builds the HTTP application, wires dependencies, registers routes, then starts the service loop.

This is similar to a `Migrator` target: the executable has one clear job, and most reusable logic lives in modules outside the entrypoint.

## Choose Hummingbird or Vapor

Feather Framework can work with either Hummingbird or Vapor.

Hummingbird is a lightweight, modular Swift web framework built around SwiftNIO. It is a good default when you want a small server layer and explicit composition.

Vapor is a mature Swift web framework with a large ecosystem. It provides routing, middleware, configuration patterns, and many web application tools.

Feather Framework is not a replacement for these frameworks. Hummingbird or Vapor handles the HTTP server layer. Feather Framework components help organize the application code around that layer.

## Package setup

Create a dedicated executable target named `Server`.

For Hummingbird:

```swift
// Package.swift

dependencies: [
    .package(
        url: "https://github.com/hummingbird-project/hummingbird.git",
        from: "2.0.0"
    ),
    .package(
        url: "https://github.com/apple/swift-log.git",
        from: "1.0.0"
    ),
],
targets: [
    .executableTarget(
        name: "Server",
        dependencies: [
            .product(
                name: "Hummingbird",
                package: "hummingbird"
            ),
            .product(
                name: "Logging",
                package: "swift-log"
            ),
            "Application",
            "Infrastructure",
        ]
    ),
]
```

For Vapor:

```swift
// Package.swift

dependencies: [
    .package(
        url: "https://github.com/vapor/vapor.git",
        from: "4.0.0"
    ),
],
targets: [
    .executableTarget(
        name: "Server",
        dependencies: [
            .product(
                name: "Vapor",
                package: "vapor"
            ),
            "Application",
            "Infrastructure",
        ]
    ),
]
```

The exact module names depend on your project. The important idea is that `Server` depends on your application and infrastructure packages, then chooses the concrete HTTP framework.

## Hummingbird entrypoint

A minimal Hummingbird server can be started from `Sources/Server/Entrypoint.swift`:

```swift
import Hummingbird

@main
struct Entrypoint {
    static func main() async throws {
        let router = Router()

        router.get("health") { _, _ in
            Response(status: .ok)
        }

        let app = Application(
            router: router,
            configuration: .init(
                address: .hostname("127.0.0.1", port: 8080)
            )
        )

        try await app.runService()
    }
}
```

This is enough for a basic `/health` route.

In a real application, keep the entrypoint small and move construction into builder functions:

```swift
@main
struct Entrypoint {
    static func main() async throws {
        let config = try await ServerConfigLoader().load()
        let server = try await buildServer(config: config)
        try await server.runService()
    }
}
```

That structure keeps startup readable:

1. Load typed configuration.
2. Build the server.
3. Run the service.

## Hummingbird build function

```swift
import Hummingbird
import Logging

func buildServer(
    config: ServerConfig
) async throws -> some ApplicationProtocol {
    var logger = Logger(label: config.serverName)
    logger.logLevel = config.logLevel

    let infrastructure = try await buildInfrastructure(
        config: config,
        logger: logger
    )

    let modules = AppModules(
        infrastructure: infrastructure
    )

    let router = try buildRouter(
        modules: modules
    )

    return Application(
        router: router,
        configuration: .init(
            address: .hostname(config.host, port: config.port)
        )
    )
}
```

The server builder wires concrete infrastructure into application modules, then creates the router.

## Hummingbird router

```swift
import Hummingbird

func buildRouter(
    modules: AppModules
) throws -> Router<AppRequestContext> {
    let router = Router(context: AppRequestContext.self)

    router.get("health") { _, _ in
        Response(status: .ok)
    }

    router.addMiddleware {
        LogRequestsMiddleware(.info)
        CORSMiddleware(
            allowOrigin: .originBased,
            allowHeaders: [.accept, .authorization, .contentType],
            allowMethods: [.get, .post, .put, .patch, .delete],
            allowCredentials: true
        )
    }

    let api = AppAPI(modules: modules)
    try api.registerHandlers(
        on: router,
        middlewares: [
            ErrorMiddleware()
        ]
    )

    return router
}
```

The router should connect HTTP concerns to application use cases. It should not contain business rules.

## Vapor entrypoint

A minimal Vapor server can also live in `Sources/Server/Entrypoint.swift`:

```swift
import Vapor

@main
enum Entrypoint {
    static func main() throws {
        var environment = try Environment.detect()
        try LoggingSystem.bootstrap(from: &environment)

        let app = Application(environment)
        defer {
            app.shutdown()
        }

        try routes(app)
        try app.run()
    }
}
```

Add a small routes function:

```swift
import Vapor

func routes(_ app: Application) throws {
    app.get("health") { _ in
        HTTPStatus.ok
    }
}
```

For larger applications, keep the same idea as the Hummingbird setup: load configuration, build infrastructure, build modules, register routes, then run the app.

## Where Feather Framework fits

The `Server` target is the place where concrete choices are made.

For example, the server target can choose:

- Hummingbird or Vapor for HTTP.
- PostgreSQL, MySQL, or SQLite for database access.
- S3 or filesystem storage.
- SMTP, SES, or the in-memory `FeatherMailTesting` client.
- Async HTTP Client or URLSession for outgoing HTTP clients.

Application use cases should depend on protocols and module builders. The `Server` target decides which concrete implementation is used at runtime.

## Recommended structure

```text
Sources/Server
  Entrypoint.swift
  Server+Build.swift
  Routes+Build.swift
  Environment/
  AppModules/
  APIs/
  Middlewares/
  Errors/
  Adapters/
```

Use this split when the server grows beyond a few routes:

- `Entrypoint.swift` starts the executable.
- `Environment/` loads typed configuration.
- `Server+Build.swift` builds infrastructure and the HTTP application.
- `Routes+Build.swift` registers routes and middleware.
- `AppModules/` wires use case builders.
- `APIs/` contains OpenAPI or route handlers.
- `Adapters/` bridges server code to application protocols.

This keeps the server layer professional and predictable without making the entrypoint heavy.
