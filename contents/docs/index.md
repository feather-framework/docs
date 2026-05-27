---
type: page
views:
    html: docs.home
title: "Documentation"
description: "Learn how to build modular server-side Swift applications with Feather Framework and FATHOMS architecture."
---

# Documentation

Welcome to the Feather documentation.

Feather Framework is a collection of reusable Swift components for common backend work: database access, storage, mail, HTTP clients, testing helpers, and OpenAPI integration.

FATHOMS is the architecture guide that shows how to organize those pieces into clear modules, layers, use cases, and adapters. It helps keep a server-side Swift application understandable as it grows.

Together, Feather and FATHOMS give you a practical foundation for building modular backend applications without rebuilding the same infrastructure patterns for every project.

## Start here

If you are new to Feather, begin with [Getting Started](/docs/getting-started/). It explains what Feather is, how the components fit together, and how Feather relates to Vapor, Hummingbird, and FATHOMS.

If you already know the basic idea, jump into the component guides:

- [Database](/docs/database/) explains the SQL-first database layer, drivers, migrations, access-layer patterns, and advanced database practices.
- [Storage](/docs/storage/) explains the common storage API and drivers for local files, S3-compatible services, and in-memory testing.
- [Mail](/docs/mail/) explains how to send email through a common interface with ephemeral, SMTP, and SES drivers.
- [HTTP](/docs/http/) explains HTTP clients, request testing, and server setup with Hummingbird or Vapor.
- [OpenAPI](/docs/openapi/) explains how to describe API contracts, integrate generated server code, and use generated clients.
- [FATHOMS](/docs/fathoms/) explains the application architecture behind larger Feather projects, including modules, the server layer, and RBAC.

## How to read these docs

The guides are written for application developers. They focus on what each component does, when to use it, and how it fits into a real backend.

You do not need to adopt everything at once. A project can start with one Feather component, then add more structure as the application grows. Feather works with both Hummingbird and Vapor, while FATHOMS gives you a consistent way to organize the code around that HTTP layer.

The goal is simple: keep the important backend decisions visible, typed, and easy to change.
