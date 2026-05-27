---
category: "fathoms"
title: "Server Layer"
description: "Learn how the FATHOMS server layer assembles configuration, modules, routes, middleware, and adapters."
order: 3
---

# Server Layer
---

The Server layer is the outer assembly shell of a FATHOMS application.

It owns startup, configuration loading, runtime wiring, route registration, middleware, error mapping, and adapters between HTTP schemas and application DTOs.

It should not own core business rules. Those belong in Domain and Application layers inside modules.

## Startup flow

A typical server startup path looks like this:

```text
Entrypoint.main
   |
   v
ServerConfigLoader.load
   |
   v
buildServer(config)
   |
   +--> build logger
   +--> build database client
   +--> build shared infrastructure
   +--> build application modules
   +--> build router
   |
   v
Application.runService
```

The entry point should stay small:

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

This keeps startup easy to follow:

1. Load configuration.
2. Build the server.
3. Run the service.

## Configuration

The configuration layer converts raw configuration values into typed runtime values.

```swift
struct ServerConfig: Sendable {
    var host: String
    var port: Int
    var serverName: String
    var system: SystemConfig
}
```

A loader can read environment variables, configuration files, or any other source your application supports:

```swift
func load() async throws -> ServerConfig {
    let reader = try await environmentLoader.loadConfigReader(
        defaultEnvironmentFilePrefix: "server"
    )

    let http = reader.scoped(to: "server.http")

    return .init(
        host: http.string(forKey: "host", default: "127.0.0.1"),
        port: http.int(forKey: "port", default: 8080),
        serverName: reader.string(forKey: "serverName"),
        system: systemConfigLoader.load(reader: reader)
    )
}
```

Typed configuration matters because the build layer should receive one clear object, not scattered environment reads across the codebase.

## Runtime composition

The composition layer wires concrete infrastructure into module builders.

```swift
struct AppInfrastructure: Sendable {
    let database: any DatabaseClient
    let idGenerator: any IDGenerator
    let mailSender: any MailSender
}
```

`AppInfrastructure` is the shared dependency bag passed into module builders.

```swift
struct AppModules: Sendable {
    let system: SystemModule
    let account: AccountModule
    let auth: AuthModule
}
```

Each module builder exposes factory methods for use cases:

```swift
func makeCreateAccount() -> CreateAccount {
    let transaction = DatabaseTransactionExecutor(
        database: infrastructure.database,
        scope: { connection in
            WriteAccount(
                account: DatabaseAccountRepository(
                    connection: connection
                )
            )
        }
    )

    return .init(
        transaction: transaction,
        idGenerator: infrastructure.idGenerator,
        passwordHasher: infrastructure.passwordHasher
    )
}
```

This is dependency injection in a simple form. The use case receives protocols and executors. The server layer chooses the concrete implementations.

## HTTP server framework

The Server layer can be built on top of Hummingbird or Vapor.

Hummingbird and Vapor handle the HTTP server layer: routing, middleware, request and response handling, and service lifecycle. They also provide useful web-framework tools. Feather and FATHOMS sit at a different abstraction level: they help organize the application, modules, components, and adapters around that HTTP layer.

In other words, FATHOMS is not a competitor to Hummingbird or Vapor. It is an application architecture style that can work with either.

## Router assembly

The router connects HTTP routes to application behavior.

For an OpenAPI-based server, route assembly usually registers generated handlers:

```swift
let router = Router(context: AppRequestContext.self)

router.get("/health") { _, _ in
    Response(status: .ok)
}

let middlewares: [any ServerMiddleware] = [
    ErrorMiddleware(),
    AuthSubjectMiddleware(modules: modules),
]

let appAPI = AppAPI(modules: modules)
try appAPI.registerHandlers(
    on: router,
    middlewares: middlewares
)

let adminAPI = AdminAPI(modules: modules)
try adminAPI.registerHandlers(
    on: router,
    middlewares: middlewares
)
```

The exact API groups are up to your project. A common split is public app routes and admin routes.

## API handlers

API handlers are server adapters. They translate HTTP/OpenAPI inputs into application DTOs, call use cases, and translate DTOs back into generated response schemas.

```swift
struct AppAPI: APIProtocol {
    let modules: AppModules
}
```

A handler should be thin:

```swift
func accountGet(
    _ input: Operations.AccountGet.Input
) async throws -> Operations.AccountGet.Output {
    let useCase = modules.account.makeGetAccount()

    let result = try await useCase.execute(
        .init(id: input.path.accountId)
    )

    return .ok(
        .init(
            body: .json(map(result))
        )
    )
}
```

This handler:

1. Receives generated OpenAPI input.
2. Builds the matching use case.
3. Executes application logic.
4. Maps the result DTO into a response schema.
5. Returns a typed OpenAPI output.

The handler does not contain business rules. It only coordinates the HTTP boundary.

## Middleware

Middleware handles cross-cutting request and response behavior.

Common middleware responsibilities include:

- Logging.
- CORS.
- Authentication subject loading.
- Session renewal.
- Error handling.
- Header normalization.

Keep middleware focused. If middleware starts to contain business decisions, move that behavior into Application use cases or authorization policies.

## Error mapping

The Server layer can adapt domain, application, infrastructure, and framework errors into one consistent HTTP response shape.

A useful pattern is to separate debug tracing from public error output.

`ErrorTraceRepresentable` can provide a debug tree:

```swift
protocol ErrorTraceRepresentable: Error {
    func trace() -> ErrorTrace
}
```

`HTTPErrorRepresentable` can provide the public HTTP shape:

```swift
protocol HTTPErrorRepresentable: Error {
    associatedtype Content: Encodable

    var status: HTTPResponseStatus { get }
    var headers: HTTPHeaders? { get }
    var content: Content? { get }
}
```

Then error middleware can:

1. Catch the thrown error.
2. Log the internal trace.
3. Map the error to an HTTP status and response body.
4. Return JSON or plain text depending on the request.

This keeps errors useful for developers without exposing private implementation details to clients.

## Infrastructure adapters

Infrastructure adapters connect abstract application contracts to concrete libraries.

Examples:

- `NanoIDGenerator` implements `IDGenerator`.
- `BcryptPasswordHasher` implements `PasswordHasher`.
- `FeatherMailSender` implements `MailSender`.
- `FeatherAvatarStorage` implements an application storage port.

```swift
struct NanoIDGenerator: IDGenerator {
    func generate() -> String {
        NanoID().rawValue
    }
}
```

Use cases depend on interfaces. The server layer decides which concrete implementation to use.

## DTO and schema adapters

Generated OpenAPI schemas should not leak through the whole application.

Use mapping helpers to convert between generated schemas and application DTOs:

```swift
func map(_ result: AccountDetail) -> Components.Schemas.Account {
    .init(
        id: result.id,
        email: result.email,
        status: map(result.status)
    )
}
```

For incoming requests, map schema input into use case input:

```swift
func map(
    _ body: Components.Schemas.CreateAccountRequest
) -> CreateAccount.Input {
    .init(
        email: body.email,
        password: body.password
    )
}
```

This keeps controllers thin and prevents generated transport types from spreading into Domain or Application layers.

## Recommended server folders

A FATHOMS server target can be organized like this:

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

Suggested responsibilities:

- `Environment/`: typed server configuration and loading.
- `AppModules/`: module builder aggregation and use case factory methods.
- `APIs/`: OpenAPI operation handlers.
- `Middlewares/`: cross-cutting HTTP behavior.
- `Errors/`: HTTP error contracts, details, and tracing.
- `Adapters/`: infrastructure adapters, error adapters, and DTO/schema mapping helpers.

## Mental model

The Server layer is the outer edge of the application.

It assembles the runtime, connects the HTTP framework, registers handlers, and adapts external concerns to application use cases.

If you keep business rules out of the Server layer, the rest of the system remains easier to test, easier to change, and easier for new developers to understand.
