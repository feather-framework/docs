---
title: "Feather Framework - Modular backend components for server-side Swift"
description: "Learn how to build server-side Swift applications with Feather Framework."
image: "/images/defaults/default.jpg"
keyword: "server-side Swift framework"
views:
    html: "docs.home"
---

# Documentation

Welcome to the Feather Framework documentation.

Feather Framework is a collection of reusable Swift components for common backend work: database access, storage, mail, HTTP clients, testing helpers, and OpenAPI integration.

Feather Framework gives you a practical foundation for building backend applications without rebuilding the same infrastructure patterns for every project.

## Start here

If you are new to Feather Framework, begin with [Getting Started](/docs/getting-started/). It explains what Feather Framework is, how the components fit together, and how Feather Framework relates to Vapor and Hummingbird.

If you already know the basic idea, jump into the component guides:

- [Database](/docs/database/) explains the SQL-first database layer, drivers, migrations, access-layer patterns, and advanced database practices.
- [Storage](/docs/storage/) explains the common storage API and drivers for local files, S3-compatible services, and in-memory testing.
- [Mail](/docs/mail/) explains how to send email through a common interface with ephemeral, SMTP, and SES drivers.
- [HTTP](/docs/http/) explains HTTP clients, request testing, and server setup with Hummingbird or Vapor.
- [OpenAPI](/docs/openapi/) explains how to describe API contracts, integrate generated server code, and use generated clients.
## How to read these docs

The guides are written for application developers. They focus on what each component does, when to use it, and how it fits into a real backend.

You do not need to adopt everything at once. A project can start with one Feather Framework component, then add more as the application grows. Feather Framework works with both Hummingbird and Vapor.

The goal is simple: keep the important backend decisions visible, typed, and easy to change.
