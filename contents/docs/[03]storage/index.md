---
type: "category"
title: "Storage"
description: "Learn how Feather Storage gives Swift applications one common API for files, object storage, and test storage."
order: 5
---

# Storage
---

Feather Storage gives your application a common way to store and read files.

The storage backend can change without changing the rest of your application. You can use the filesystem driver for local development, an S3-compatible driver for production object storage, or the in-memory `FeatherStorageTesting` client for tests.

The goal is to keep file handling simple at the application boundary. Your use cases can depend on a storage interface, while the server or infrastructure layer decides where the bytes actually live.

Start with the overview, then choose the driver guide that fits your environment.
