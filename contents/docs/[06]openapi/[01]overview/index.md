---
category: "openapi"
title: "Overview"
description: "Learn what Feather OpenAPI is and how it helps you describe HTTP APIs in Swift."
order: 1
---

# Overview
---

[Feather OpenAPI](https://github.com/feather-framework/feather-openapi) helps you define OpenAPI specifications using Swift in a type-safe way.

OpenAPI is a standard format for describing HTTP APIs. It explains which endpoints exist, what request and response bodies look like, which parameters are accepted, and how authentication works.

## Why use OpenAPI?

An OpenAPI document can be used for:

- API documentation.
- Client generation.
- Server stub generation.
- Contract testing.
- Sharing API behavior with frontend, mobile, or partner teams.

Writing OpenAPI YAML by hand is possible, but it becomes repetitive as the API grows. It is also easy to make small mistakes in references, schema names, response structures, or shared components.

Feather OpenAPI gives you a Swift-first way to build the document from smaller reusable pieces.

## What Feather OpenAPI provides

Feather OpenAPI builds on top of `OpenAPIKit`.

It provides representable Swift types for common OpenAPI concepts:

- Documents.
- Info.
- Servers.
- Paths and path items.
- Operations.
- Parameters.
- Request bodies.
- Responses.
- Headers.
- Schemas.
- Tags.
- Security schemes and requirements.
- Components and references.

The goal is to describe your API with normal Swift types, then produce a valid OpenAPI document from those types.

## A simple mental model

Think of the OpenAPI document as a tree:

```text
Document
Info
Servers
Paths
Operations
Parameters
Request bodies
Responses
Schemas
Components
```

Feather OpenAPI lets each part be modeled as its own Swift type. This keeps large specifications easier to organize.

## How it fits with generated code

Feather OpenAPI is about building the API specification.

You can then use the generated OpenAPI document with tools such as Swift OpenAPI Generator. In a typical application, generated OpenAPI packages are used by both server and client code. The server registers generated handlers on Hummingbird or Vapor, while clients use the generated `Client` type with a transport such as `URLSessionTransport`.

That gives the project a clear flow:

```text
OpenAPI specification
Generated Swift API package
Server implementation
Client integration
Hummingbird or Vapor HTTP layer
```

Feather does not force one server framework. The generated API layer can be connected to the HTTP server layer that fits your application.
