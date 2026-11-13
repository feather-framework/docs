---
category: "getting-started"
title: "Feather Framework"
description: "Learn how Feather Framework's core components help you build database, storage, mail, HTTP, and OpenAPI features in server-side Swift applications."
order: 2
---

# Feather Framework
---

Feather Framework is built from focused components. Each component solves a common backend problem through a small, consistent Swift API.

The goal is not to force every project into one large framework. Instead, Feather Framework gives you practical building blocks that can be used together or separately. Your application can depend on stable interfaces, while the concrete implementation can come from the driver, provider, or executor that fits your environment.

This approach keeps application code cleaner. It also makes testing easier, because you can replace real infrastructure with the testing libraries and executors included by the components.

## Feather Database

[Feather Database](https://github.com/feather-framework/feather-database) provides a shared API surface for database drivers written in Swift.

Most backend applications need to read and write data, but database code can become tightly coupled to one specific driver. Feather Database helps avoid that by giving your application a database-agnostic abstraction layer.

In simple terms, your application can work with a common database interface instead of being built directly around SQLite, PostgreSQL, or MySQL-specific APIs. This gives your project a more consistent shape and makes database code easier to organize.

Feather Database is designed for modern Swift concurrency, so it fits naturally with `async` and `await`. It also includes DocC-based API documentation, unit tests, and code coverage support.

Available database driver implementations include:

- [SQLite](https://github.com/feather-framework/feather-database-sqlite)
- [PostgreSQL](https://github.com/feather-framework/feather-database-postgres)
- [MySQL](https://github.com/feather-framework/feather-database-mysql)

## Feather Mail

[Feather Mail](https://github.com/feather-framework/feather-mail) provides an abstract mail component for Swift applications.

Sending email is a common backend task, but the details can vary depending on the service you use. One project might send mail through SMTP, another might use Amazon SES, and tests might need a local or in-memory implementation that does not send real messages.

Feather Mail gives these use cases a shared structure. It provides a common mail type for sending messages, validation helpers, error handling, a raw MIME encoder for transport providers, and support for attachments and HTML emails.

This means your application can describe the email it wants to send without spreading provider-specific details through the codebase. The actual delivery can then be handled by the implementation you choose.

Available mail implementations include:

- [Amazon SES](https://github.com/feather-framework/feather-mail-ses)
- [SMTP](https://github.com/feather-framework/feather-mail-smtp)

For tests, the core package includes the `FeatherMailTesting` library with the in-memory `TestMailClient` and `TestMailbox` types.

## Feather Storage

[Feather Storage](https://github.com/feather-framework/feather-storage) provides a shared API surface for file storage drivers written in Swift.

Applications often need to store files such as uploads, images, generated documents, or private assets. Those files might live on the local filesystem during development, in S3 in production, or in memory during tests.

Feather Storage gives your application one standard way to work with files across those environments. Instead of writing application logic directly against a specific storage provider, you can depend on a storage abstraction and choose the driver separately.

This makes storage code easier to test and easier to move between environments. It also keeps provider-specific details, such as S3 configuration or filesystem paths, out of the parts of your application that only need to save, load, or delete files.

Available storage driver implementations include:

- [Filesystem](https://github.com/feather-framework/feather-storage-fs)
- [S3](https://github.com/feather-framework/feather-storage-s3)

For tests, the core package includes the `FeatherStorageTesting` library with the in-memory `TestStorageClient` and `TestStorageState` types.

## Feather HTTP

[Feather HTTP](https://github.com/feather-framework/feather-http) provides a basic HTTP abstraction for Swift applications.

Backend services often need to call other services. They might request data from an external API, send a webhook, upload a file, or communicate with another internal system. Without a common abstraction, these calls can become tied to one HTTP client implementation.

Feather HTTP provides a shared request and response model, plus an extensible executor protocol. Requests can be represented in a consistent way, while the executor decides how the request is actually performed.

The component includes helpers for status codes, headers, plain text, JSON, data, and streaming response bodies. It also supports per-request options such as timeouts and response body limits.

This gives your application a clean place to describe outgoing HTTP work, while still allowing different executors for production, platform-specific code, or tests.

Available executor implementations include:

- [Async HTTP Client](https://github.com/feather-framework/feather-http-ahc)
- [URLSession](https://github.com/feather-framework/feather-http-urlsession)
- [Vapor Testing](https://github.com/feather-framework/feather-http-vapor-testing)
- [Hummingbird Testing](https://github.com/feather-framework/feather-http-hummingbird-testing)

## Feather OpenAPI

[Feather OpenAPI](https://github.com/feather-framework/feather-openapi) helps you define OpenAPI specifications using Swift in a type-safe way.

OpenAPI documents describe how an HTTP API works: which endpoints exist, which request and response types they use, and how clients can interact with the service. These documents are useful for API documentation, client generation, validation, and team communication.

Writing OpenAPI by hand can be repetitive and easy to get wrong. Feather OpenAPI gives you a Swift interface for building OpenAPI documents with more structure and stronger type safety.

It also supports automatic identifier generation and resolution, which helps keep larger API specifications organized as they grow.

## Shared qualities

The Feather Framework components follow the same general direction:

- They are designed for modern Swift concurrency.
- They provide focused abstractions instead of large all-in-one systems.
- They include DocC-based API documentation.
- They use tests and code coverage to support long-term quality.
- They are designed so custom drivers, providers, or executors can be built when needed.

Each component is still evolving toward a stable 1.0 release, so check the linked repositories for the latest version and implementation details before adopting them in production.
