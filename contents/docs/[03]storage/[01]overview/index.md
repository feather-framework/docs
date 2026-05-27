---
category: "storage"
title: "Overview"
description: "Learn what Feather Storage is, why it is useful, and how it helps you work with files in Swift."
order: 1
---

# Overview
---

[Feather Storage](https://github.com/feather-framework/feather-storage) is a file storage abstraction for Swift applications. It gives your application one clear API for storing, reading, listing, copying, moving, and deleting files.

This is useful because file storage often changes between environments. During development you might store files on the local filesystem. In production you might use an S3-compatible service. In tests you may want an in-memory implementation that never touches the network or disk.

Feather Storage keeps those choices separate from your application logic.

## Why use Feather Storage?

Most applications eventually need to handle files: user uploads, images, exports, private documents, generated reports, or cached assets.

Without a storage abstraction, file handling code can become tied to a specific provider. That makes tests harder, and it can make future infrastructure changes more expensive.

Feather Storage gives you:

- One common `StorageClient` protocol.
- Streaming upload and download support.
- A consistent key-based model for files and prefixes.
- Basic object operations such as `exists`, `size`, `copy`, `move`, `list`, and `delete`.
- Multipart upload support for large objects.
- Driver implementations for filesystem, S3-compatible storage, and in-memory testing.

In simple terms: your application asks for storage work to happen, and the selected driver decides where the bytes actually go.

## Storage keys

Feather Storage uses string keys to identify stored objects.

A key is similar to a path:

```swift
"avatars/user-123/profile.png"
"reports/2026/april.csv"
"exports/orders.json"
```

The exact storage backend may not have real folders. S3, for example, stores objects by key and treats `/` as part of the name. The filesystem driver stores keys under a root directory. Feather gives both cases the same shape in your code.

## Upload and download

Uploads and downloads use `StorageSequence`, which is an asynchronous byte sequence. This keeps the API suitable for larger files because the content does not need to be modeled as one special provider-specific type.

A small upload can start from a `ByteBuffer`:

```swift
import FeatherStorage
import NIOCore

let buffer = ByteBuffer(string: "Hello storage")
let sequence = StorageSequence(
    asyncSequence: ByteBufferSequence(buffer: buffer),
    length: UInt64(buffer.readableBytes)
)

try await storage.upload(
    key: "notes/hello.txt",
    sequence: sequence
)
```

To read the object back:

```swift
let downloaded = try await storage.download(
    key: "notes/hello.txt",
    range: nil
)

let data = try await downloaded.collect(upTo: 1024 * 1024)
```

You can also request a byte range:

```swift
let firstBytes = try await storage.download(
    key: "videos/intro.mp4",
    range: 0...1023
)
```

Range downloads are useful for previews, resumable work, and APIs that only need part of a file.

## Basic object operations

The storage client includes common file operations:

```swift
let exists = try await storage.exists(key: "notes/hello.txt")
let size = try await storage.size(key: "notes/hello.txt")

try await storage.copy(
    key: "notes/hello.txt",
    to: "notes/hello-copy.txt"
)

try await storage.move(
    key: "notes/hello-copy.txt",
    to: "archive/hello.txt"
)

let entries = try await storage.list(key: "notes")

try await storage.delete(key: "archive/hello.txt")
```

These operations keep day-to-day storage work readable and consistent across drivers.

## Multipart uploads

Multipart uploads are useful when an object is too large to upload comfortably in one request.

The common API supports this flow:

```swift
let uploadId = try await storage.createMultipartId(
    key: "videos/large.mov"
)

let firstChunk = try await storage.upload(
    multipartId: uploadId,
    key: "videos/large.mov",
    number: 1,
    sequence: partOne
)

let secondChunk = try await storage.upload(
    multipartId: uploadId,
    key: "videos/large.mov",
    number: 2,
    sequence: partTwo
)

try await storage.finish(
    multipartId: uploadId,
    key: "videos/large.mov",
    chunks: [firstChunk, secondChunk]
)
```

If something fails before the upload is complete, abort the upload:

```swift
try await storage.abort(
    multipartId: uploadId,
    key: "videos/large.mov"
)
```

The filesystem, S3, and ephemeral drivers all expose this same shape, even though the internal implementation is different.

## Recommended application structure

For small projects, using `StorageClient` directly is fine.

For larger applications, it is often better to define an application-level protocol and adapt Feather Storage at the infrastructure boundary. This is the same pattern you can use across a real application for storage, mail, HTTP clients, and other external services.

```swift
public protocol AvatarStorage: Sendable {
    func saveAvatar(
        userId: String,
        bytes: StorageSequence
    ) async throws
}

struct FeatherAvatarStorage: AvatarStorage {
    let storage: any StorageClient

    func saveAvatar(
        userId: String,
        bytes: StorageSequence
    ) async throws {
        try await storage.upload(
            key: "avatars/\(userId)/profile.png",
            sequence: bytes
        )
    }
}
```

This keeps business logic focused on application concepts such as avatars, invoices, or exports. The storage driver remains a configuration choice.

## Available drivers

The currently available storage implementations are:

- [Filesystem](https://github.com/feather-framework/feather-storage-fs)
- [S3](https://github.com/feather-framework/feather-storage-s3)
- [Ephemeral](https://github.com/feather-framework/feather-storage-ephemeral)

Use the filesystem driver for local disk storage, the S3 driver for S3-compatible object storage, and the ephemeral driver for tests or temporary local workflows.
