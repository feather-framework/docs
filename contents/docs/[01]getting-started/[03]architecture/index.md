---
category: "getting-started"
title: "FATHOMS"
description: "Learn how FATHOMS helps structure server-side Swift applications using modular monoliths, clear layers, and explicit boundaries."
order: 3
---

# FATHOMS
---

FATHOMS stands for **Feather Architecture for Highly Organized Modular Systems**. It is the application structure used by Feather projects to keep server-side Swift code understandable as a backend grows.

The goal is simple: give each part of the application a clear place to live. Business rules should stay close to the domain. Use cases should describe application behavior. Technical details, such as databases and HTTP handlers, should stay near the outside of the system.

This keeps the project easier to read, easier to test, and easier to change over time.

## Why FATHOMS exists

Backend applications often start small, then slowly collect many responsibilities. Route handlers begin to contain business rules. Database queries appear in unrelated places. External services become hard to replace. Tests become slower and harder to write.

FATHOMS helps avoid that by giving the application a predictable structure from the beginning.

FATHOMS combines three practical ideas:

- A modular monolith for organizing features.
- Onion architecture for keeping dependencies pointed inward.
- Hexagonal architecture for connecting the core through ports and adapters.

In simple terms, FATHOMS lets you build one backend application without turning it into one large, tangled codebase.

## Modular monolith

A FATHOMS backend is usually a modular monolith.

A modular monolith is one deployable application split into internal feature modules. It runs as one process and can use one shared database, but the code is still organized by business area.

For example, a backend might contain modules such as:

- `system` for system values and permissions.
- `user` for accounts, roles, and invitations.
- `auth` for sessions, magic links, and sign-in flows.

Each module owns its own business logic and persistence adapters. Modules should communicate through clear interfaces instead of reaching directly into each other's internals.

This gives you simpler deployment than microservices, while still keeping the codebase organized enough for a growing product.

## Layers inside a module

Each FATHOMS module is split into three main layers. You can think of them as three circles, from the core of the product to the technical details around it:

- **Domain**: business models, rules, errors, and repository interfaces.
- **Application**: use cases, DTOs, scopes, authorization checks, and workflows.
- **Infrastructure**: database tables, migrations, repository adapters, and query adapters.

The dependency direction is always inward:

```text
Infrastructure -> Application -> Domain
```

This rule matters. The Domain layer should not know about HTTP, SQL, storage providers, or framework details. The Application layer can coordinate business workflows, but it should still depend on interfaces rather than concrete infrastructure. The Infrastructure layer implements those interfaces using real tools.

## Domain layer

The Domain layer describes what the business means.

This is where models and core rules live. For example, an account model can define what makes an email valid, which status values are allowed, and what errors can happen when creating or updating an account.

Domain code should be small, focused, and independent. It should avoid HTTP request types, SQL queries, framework-specific APIs, and other outer technical details.

In simple terms: Domain is the center of the application. It contains the rules that should remain true no matter which database, server framework, or external provider the project uses.

## Application layer

The Application layer describes what the system can do.

This is where use cases live. A use case is one business action, such as adding an account, listing magic links, or getting the current user's profile.

Use cases usually:

- Accept input DTOs.
- Run validation and authorization checks.
- Call domain models and repository/query interfaces.
- Use transactions for write operations.
- Return output DTOs.

A DTO, or Data Transfer Object, is a clear data shape used at application boundaries. DTOs help prevent internal domain details, such as password hashes or private state, from leaking into API responses.

## Infrastructure layer

The Infrastructure layer connects the application to real systems.

This is where database access, SQL queries, migrations, repository implementations, and query implementations live. Infrastructure implements the ports defined by the inner layers.

FATHOMS uses a SQL-first style for persistence. This keeps database behavior visible and predictable. Instead of hiding important details behind a heavy object-relational mapping system, the table and adapter layers make row mapping and query behavior explicit.

In simple terms: Infrastructure handles the technical work, but the business rules still belong inside the core.

## Ports and adapters

FATHOMS also uses the ports and adapters idea from Hexagonal architecture.

A port is an interface that the core application needs. An adapter is a concrete implementation of that interface.

For example:

- A repository protocol is a port.
- A database-backed repository is an adapter.
- An ID generator protocol is a port.
- A NanoID-based generator is an adapter.
- A mail sender protocol is a port.
- A provider-backed mail sender is an adapter.

This makes the system easier to test and easier to change. A use case can depend on a protocol, while the Server layer chooses the real implementation during application startup.

## Kernel and feature modules

FATHOMS projects often use a shared kernel package.

The kernel contains common contracts used by all modules. It should stay small and focused. Typical kernel concepts include:

- `Model`
- `Repository`
- `UseCase`
- `DTO`
- `Scope`
- query and transaction executors
- authorization contracts
- shared infrastructure helpers

Feature modules build on the kernel. Each feature module owns a specific business area and usually contains its own Domain, Application, and Infrastructure targets.

Think of the kernel as the shared language of the application, and feature modules as the places where product behavior is implemented.

## Server layer

The Server layer is the outer assembly shell of the application.

It is responsible for startup, configuration, routing, middleware, and connecting concrete implementations to use cases. It does not own the core business rules.

The actual HTTP server can be built with Vapor, Hummingbird, or another Swift server framework. FATHOMS does not require one specific server implementation. The server framework handles incoming requests and outgoing responses, while the FATHOMS structure keeps the application logic behind those routes organized.

A typical server startup flow looks like this:

```text
main()
   |
   v
load configuration
   |
   v
build shared infrastructure
   |
   v
build feature modules
   |
   v
build router and middleware
   |
   v
run server
```

This layer wires the application together. For example, it can create a database client, create module builders, register OpenAPI handlers, and attach middleware for errors, authentication, or response processing.

## Request flow

A typical request moves from the outside of the system toward the core, then back out again:

```text
HTTP request
   |
   v
OpenAPI handler
   |
   v
Server mapping
   |
   v
Application use case
   |
   v
Domain rules and ports
   |
   v
Infrastructure adapter
   |
   v
Database or external service
```

The response follows the reverse path. Infrastructure returns data to the use case, the use case returns a DTO, the Server layer maps that DTO to an API response, and the HTTP framework sends it back to the client.

This flow is useful when debugging. If something breaks, you can inspect the layer where the responsibility belongs instead of searching the whole codebase.

## What to remember

FATHOMS is not about adding ceremony for its own sake. It is about keeping a backend understandable.

The main idea is that each responsibility has a clear place:

- Domain keeps the business rules.
- Application keeps the workflows.
- Infrastructure keeps the technical implementations.
- Server keeps delivery, routing, configuration, and wiring.

This page is only a short introduction to FATHOMS. It gives you the basic vocabulary and the high-level shape of the architecture. More in-depth guides will explain each part in a separate section, with concrete examples for modules, server assembly, persistence, authorization, and feature development.
