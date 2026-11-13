---
category: "getting-started"
title: "HTTP Servers"
description: "Understand how Vapor, Hummingbird, and Feather Framework relate to each other in server-side Swift applications."
order: 4
---

# HTTP Servers
---

Swift has two well-known server frameworks that are often considered when starting a backend project: Vapor and Hummingbird. Both are strong choices, both are valuable parts of the Swift server ecosystem, and both can be used with Feather Framework.

The important thing to understand is how the responsibilities are divided. Vapor and Hummingbird handle the HTTP server layer, although they also provide web-framework tools such as Fluent ORM in the Vapor ecosystem or Hummingbird Auth in the Hummingbird ecosystem.

Feather Framework sits at a different abstraction level and helps organize the application beyond the core web framework concerns.

## Vapor

[Vapor](https://github.com/vapor/vapor) is a popular server-side Swift web framework. It provides a full-featured foundation for building websites, HTTP APIs, and cloud services in Swift. Vapor includes routing, middleware, request and response handling, validation, authentication support, database tooling through Fluent, templating through Leaf, and a large ecosystem of related packages.

Vapor is the most widely known Swift web framework, and many production applications are already built with Vapor 4.

## Vapor 5

Vapor 4 has been an important framework for server-side Swift. It helped many developers build real Swift backends and gave the ecosystem a mature, approachable web framework.

At the same time, Vapor 4 provides a broad, integrated framework experience. That is useful for many teams, especially when they want a complete ecosystem with well-known conventions. For projects that prefer composing smaller independent parts, a lighter and more modular HTTP layer can be a better fit.

Vapor 5 is in beta and its APIs are still evolving. It is built around structured concurrency, newer Swift ecosystem packages, updated WebSocket and multipart APIs, and a more modern backend architecture.

Vapor 5 is a significant redesign of the HTTP server and web framework, so check its current release documentation before adopting it for a new project.

## Hummingbird

[Hummingbird](https://github.com/hummingbird-project/hummingbird) is a lightweight and flexible Swift server framework. It provides the core pieces needed to build HTTP services: routing, middleware, request and response handling, custom encoding and decoding, streaming support, and extensions for additional features.

Hummingbird was built with a modular architecture from the beginning. Its core is intentionally small, and extra capabilities can be added through separate packages. This makes it a lightweight and flexible foundation for applications that want to choose their own higher-level structure.

## Feather Framework and Hummingbird

Feather Framework is built around small components, clear boundaries, and modular application structure. Hummingbird follows a similar philosophy across the whole framework, with a small core and optional extensions for additional capabilities.

Because Hummingbird is lightweight and modular, it is a natural companion for Feather Framework. Hummingbird can focus on HTTP routing, middleware, request handling, and response delivery, while Feather Framework provides higher-level building blocks around database access, storage, mail, HTTP clients, and OpenAPI integration.

This separation keeps responsibilities clear:

- Hummingbird handles the server and HTTP request lifecycle.
- Feather Framework organizes the application and its backend components.
- Your feature modules contain the business logic.

For this reason, we currently recommend Hummingbird 2 as the default HTTP server for Feather Framework projects.

## Vapor Compatibility

Yes. Feather Framework can work with Vapor 4 too.

Feather Framework is not tied to one HTTP server implementation. A Vapor application can still use Feather Framework components behind its routes. In that setup, Vapor handles the HTTP layer, while Feather Framework provides shared backend services such as database access, storage, mail, HTTP clients, and OpenAPI support.

If your team already uses Vapor 4, you do not need to replace your existing stack to benefit from Feather Framework. You can adopt Feather Framework gradually where it helps: database abstraction, storage, mail, OpenAPI contracts, application modules, and clearer internal boundaries.

## Feather Framework's Role

If you come from other ecosystems, the difference is similar to how some frameworks provide more than just the web server:

- Laravel in PHP gives you a full application structure around routing.
- Symfony in PHP provides reusable components and application conventions.
- NestJS in TypeScript sits above the raw HTTP server and gives applications a module-oriented structure.
- Spring in Java provides a broad application framework around web delivery.

Feather Framework follows that same idea for Swift. It is not trying to be the HTTP server itself. It provides structure and components for the application around the delivery layer.

## Recommendation

- Use **Hummingbird 2** for new Feather Framework projects when you want a lightweight, modular HTTP server that matches Feather Framework's design.
- Use **Vapor 4** when your project already depends on Vapor or your team prefers Vapor's ecosystem and developer experience.
- Evaluate **Vapor 5** against its current release documentation if you want its newer concurrency and HTTP APIs.

Feather Framework is designed to work with the Swift server ecosystem. The goal is to let the HTTP framework do what it does best, while Feather Framework helps keep the rest of the backend clean, modular, and professional.
