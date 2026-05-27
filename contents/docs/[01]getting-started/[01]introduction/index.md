---
category: "getting-started"
title: "Introduction"
description: "Learn what Feather Framework is, what problems it solves, and how it helps you structure server-side Swift applications."
order: 1
---

# Introduction
---

Feather Framework is a collection of reusable Swift components for common backend tasks, such as database access, storage, mail, HTTP clients, and OpenAPI integration.

FATHOMS is the application architecture that shows how to organize those components into clear modules, layers, use cases, and infrastructure adapters.

Together, Feather and FATHOMS help you build server-side Swift applications without rebuilding the same foundation again and again.

Feather is not an HTTP server like Vapor or Hummingbird. You can still use the server framework that makes sense for your team. Feather sits at a different abstraction level and helps organize the application beyond the core web framework concerns.

## What Feather provides

Modern backend applications usually need more than routing and request handling. They often need a database, file storage, email delivery, external HTTP calls, OpenAPI contracts, and a clear way to organize business logic.

Feather focuses on these shared concerns through a component-based structure:

- Database access through a common database abstraction layer.
- File storage through a standard storage interface.
- Email delivery through a consistent mail API.
- HTTP client access through a unified client layer.
- OpenAPI support with type-safe generated interfaces.

Each component is designed to solve one practical problem. You can use the parts you need and keep the rest out of your application.

## What FATHOMS provides

FATHOMS gives guidance on how to structure a server-side Swift application.

It is an application pattern for organizing backend code into clear layers and responsibilities. The goal is to make your project easier to understand as it grows. Instead of placing everything in route handlers or service classes without a consistent shape, FATHOMS gives your application a predictable structure.

This is useful for both small and large teams. New contributors can find where things belong, tests can target the right layer, and the project can grow without losing its internal organization.

## Why this approach helps

Backend projects can become hard to maintain when every dependency is used directly across the codebase. Database code ends up in request handlers, storage logic gets mixed into business rules, and external API calls become difficult to test.

Feather and FATHOMS encourage a cleaner structure. Your application can depend on simple interfaces instead of depending directly on each database driver, storage provider, email service, or HTTP client implementation.

This makes your code easier to:

- Read, because each part has a clear responsibility.
- Test, because services can be replaced with test implementations.
- Change, because implementation details stay behind stable interfaces.
- Reuse, because the same patterns can be applied across projects.

## Works with your server framework

Because Feather is not an HTTP server, it does not try to replace your routing layer. You can still use the server framework that makes sense for your team.

Both Feather and FATHOMS work with Vapor and Hummingbird. Feather provides the reusable backend components, while FATHOMS provides the application structure around modules, layers, use cases, and adapters.

This keeps your application flexible: the HTTP framework handles requests and responses, while Feather and FATHOMS help with the deeper application concerns beyond those endpoints.
