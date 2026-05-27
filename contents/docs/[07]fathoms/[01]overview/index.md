---
category: "fathoms"
title: "Overview"
description: "Learn what FATHOMS is and how it helps organize modular server-side Swift applications."
order: 1
---

# Overview
---

FATHOMS means **Feather Architecture for Highly Organized Modular Systems**.

It is an architectural guide for building server-side Swift applications that are easy to understand, test, and grow. FATHOMS is not a separate runtime framework. It is a way to organize your code around clear modules, clear layers, and clear dependency rules.

The goal is simple: a new developer should be able to understand where a feature belongs without reading the whole codebase first.

## What FATHOMS helps with

FATHOMS is useful when your backend has more than a few routes and tables.

It helps you answer questions such as:

- Where should business rules live?
- Where should database code live?
- Where should OpenAPI handlers live?
- How should feature modules depend on each other?
- How can tests replace real infrastructure with local implementations?
- How can a project stay modular without becoming a group of microservices too early?

FATHOMS gives these questions a practical structure.

## Modular monolith first

A FATHOMS-style backend is usually a **modular monolith**.

A modular monolith is one deployable backend application split into internal feature modules. It runs as one process and can use one shared database, but the code is still organized by business area.

For example:

```text
example-backend
  app-modules
    app-kernel
    app-system-module
    app-user-module
    app-auth-module
```

This gives you simpler deployment than microservices, while avoiding the common problem where every feature ends up mixed together in one large folder.

Each module should own its own business logic and persistence adapters. Modules should not casually reach into each other’s tables or implementation details. If one module needs behavior from another module, prefer explicit interfaces.

## Multi-project workspace

A FATHOMS-based product can use a shared API contract across multiple projects:

```text
                 openapi
       API contract + generated types
                    |
       +------------+------------+
       |            |            |
       v            v            v
    backend      web app      native app
 implements API  API client   API client
```

The exact project names do not matter. The important idea is that the OpenAPI specification acts as a shared contract. The backend implements it, while clients consume it through generated types.

This keeps the API shape visible and stable across the system.

## Onion architecture

FATHOMS uses Onion architecture to keep dependencies pointing inward.

```text
Infrastructure
  Application
    Domain
```

The rule is:

```text
Infrastructure -> Application -> Domain
```

Outer layers can depend on inner layers. Inner layers should not depend on outer layers.

In practical terms:

- Domain should not know about HTTP.
- Domain should not know about SQL.
- Application should not know how a database connection is created.
- Infrastructure can implement the database, mail, storage, and external service details.
- Server code can assemble the application and connect it to Hummingbird or Vapor.

This makes the core business behavior easier to test because it is not tied directly to framework runtime details.

## Hexagonal architecture

FATHOMS also uses ideas from Hexagonal architecture, often called ports and adapters.

In simple terms:

- **Ports** are protocols that describe what the application needs.
- **Adapters** are concrete implementations of those protocols.

For example, a use case might need an `IDGenerator`, `MailSender`, or `AccountRepository`. The application layer depends on the protocol. The server or infrastructure layer decides whether the implementation uses NanoID, Feather Mail, a database repository, an in-memory test adapter, or something else.

## Onion and Hexagonal together

Onion architecture explains the layered structure and dependency direction.

Hexagonal architecture explains how the application talks to external systems through ports and adapters.

Together, they create a clear mental model:

```text
HTTP / OpenAPI / Server
        |
        v
Application use cases
        |
        v
Domain models and rules
        ^
        |
Infrastructure adapters
```

The server receives requests. Application use cases coordinate behavior. Domain models enforce rules. Infrastructure adapters handle databases, mail, storage, ID generation, hashing, and other technical details.

## The core building blocks

FATHOMS uses a few recurring building blocks across modules.

### Domain model

A domain model is a business object with validation and state rules.

For example, an account model might enforce email length, account status rules, or password requirements.

### Use case

A use case is one business operation.

Examples:

- `CreateAccount`
- `GetAccount`
- `ListAccounts`
- `RequestLoginLink`
- `RemoveInvitation`

Use cases coordinate rules and data access, but they should not contain HTTP or framework-specific code.

### DTO

A **Data Transfer Object**, or DTO, is a shape passed into or returned from a use case.

DTOs are important because they keep domain internals from leaking into API responses. For example, an account domain model may contain a password hash, but an `AccountDetail` DTO should not expose it.

### Scope

A scope is a typed bundle of dependencies needed by a use case.

Read scopes expose query interfaces. Write scopes expose repositories used inside transactions.

This keeps use case dependencies explicit.

### Query and transaction executors

Use query executors for read-only operations.

Use transaction executors for state-changing operations that should succeed or fail as one unit.

The use case does not need to know how the database connection is created. It only receives the scope it needs.

### Authorizer and permission actions

Authorization can be represented with typed actions and permission keys.

A use case asks whether a subject can perform an action. The authorizer checks the subject’s roles or permissions and either allows the use case to continue or stops it.

## Request flow

A typical request follows this path:

```text
HTTP request
   |
   v
OpenAPI-generated handler
   |
   v
Server mapping code
   |
   v
Module builder creates use case
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

This flow is useful when debugging. If a failure happens, you can usually locate it by asking which layer owns the behavior.

## How to think when adding a feature

When adding a new backend feature, work from the inside outward:

1. Domain: what business rules or model changes are needed?
2. Application: what use case exposes this behavior?
3. Infrastructure: what repository, query, or adapter implements the required ports?
4. Server: which API handler maps an HTTP request to the use case?

This order keeps changes aligned with the architecture and prevents technical details from leaking into the core business logic.

This guide is only an introduction to FATHOMS. More detailed guides in this section explain how to model modules and how to assemble the server layer.
