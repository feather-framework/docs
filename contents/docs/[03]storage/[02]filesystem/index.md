---
category: "storage"
title: "Filesystem"
description: "Learn how to configure and use the Feather Storage filesystem driver."
order: 2
---

# Filesystem
---

[Feather Storage FS](https://github.com/feather-framework/feather-storage-fs) stores objects on the local filesystem while exposing the common Feather Storage API.

It is a good choice for local development, small deployments, tools, internal services, or any environment where files should live on a disk path controlled by the application.

## Installation

Add the filesystem driver to your `Package.swift` dependencies:

```swift
.package(
    url: "https://github.com/feather-framework/feather-storage-fs",
    exact: "1.0.0-beta.1"
),
```

Then add the product to your target:

```swift
.product(
    name: "FeatherStorageFS",
    package: "feather-storage-fs"
),
```

## Imports

```swift
import FeatherStorage
import FeatherStorageFS
import NIOCore
```

`FeatherStorage` provides the common protocol and byte sequence types. `FeatherStorageFS` provides the filesystem implementation.

## Create a filesystem client

Create the client with a root directory:

```swift
let storage = StorageClientFS(
    rootPath: "/var/app/storage"
)
```

All storage keys are resolved inside this root path.

For example, this key:

```swift
"avatars/user-123/profile.png"
```

is stored under:

```text
/var/app/storage/avatars/user-123/profile.png
```

The driver creates intermediate directories when needed.

## Upload a file

```swift
let buffer = ByteBuffer(string: "Hello from Feather Storage")
let sequence = StorageSequence(
    asyncSequence: ByteBufferSequence(buffer: buffer),
    length: UInt64(buffer.readableBytes)
)

try await storage.upload(
    key: "notes/hello.txt",
    sequence: sequence
)
```

The filesystem driver writes the incoming byte sequence in chunks. By default, it uses a `64 KB` chunk size.

You can change the chunk size when creating the client:

```swift
let storage = StorageClientFS(
    rootPath: "/var/app/storage",
    chunkSize: 256 * 1024
)
```

## Download a file

```swift
let sequence = try await storage.download(
    key: "notes/hello.txt",
    range: nil
)

let buffer = try await sequence.collect(upTo: 1024 * 1024)
```

Use a range when you only need part of the file:

```swift
let preview = try await storage.download(
    key: "videos/intro.mp4",
    range: 0...4095
)
```

The range is inclusive, so `0...4095` reads the first `4096` bytes.

## Work with keys

The filesystem driver protects the configured root path by rejecting unsafe path components such as `.` and `..`.

Prefer application-generated keys:

```swift
let key = "avatars/\(userId)/profile.png"
```

Avoid using raw user-provided filenames as full keys. If a user uploads a file, store the original filename as metadata in your database and generate the storage key yourself.

## List, copy, move, and delete

```swift
let exists = try await storage.exists(key: "notes/hello.txt")
let size = try await storage.size(key: "notes/hello.txt")

let files = try await storage.list(key: "notes")

try await storage.copy(
    key: "notes/hello.txt",
    to: "notes/hello-copy.txt"
)

try await storage.move(
    key: "notes/hello-copy.txt",
    to: "archive/hello.txt"
)

try await storage.delete(key: "archive/hello.txt")
```

`move` is provided by the shared storage abstraction. It checks that the source exists, copies it, then deletes the original.

## Create a prefix

You can create a directory-like prefix:

```swift
try await storage.create(key: "uploads")
```

For local filesystem storage this creates a real directory.

## Where to use it

The filesystem driver works especially well for:

- Local development.
- Small self-hosted deployments.
- Temporary internal tools.
- Services where a mounted volume is part of the deployment.

For distributed production systems, S3-compatible storage is usually a better long-term choice because it is easier to share across multiple server instances.
