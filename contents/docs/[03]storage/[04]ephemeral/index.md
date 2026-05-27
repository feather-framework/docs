---
category: "storage"
title: "Ephemeral"
description: "Learn how to use the in-memory Feather Storage driver for tests and local workflows."
order: 4
---

# Ephemeral
---

[Feather Storage Ephemeral](https://github.com/feather-framework/feather-storage-ephemeral) is an in-memory storage driver.

It implements the same `StorageClient` protocol as the filesystem and S3 drivers, but it keeps all objects in memory. It does not write to disk and it does not perform network requests.

## When to use it

Use the ephemeral driver for:

- Unit tests.
- Integration tests that should not touch external storage.
- Local previews.
- Temporary workflows where stored files can disappear when the process exits.

Do not use it for production data. Everything is lost when the process ends.

## Installation

Add the package:

```swift
.package(
    url: "https://github.com/feather-framework/feather-storage-ephemeral",
    exact: "1.0.0-beta.2"
),
```

Then add the product:

```swift
.product(
    name: "FeatherStorageEphemeral",
    package: "feather-storage-ephemeral"
),
```

## Create the client

```swift
import FeatherStorage
import FeatherStorageEphemeral

let storage = StorageClientEphemeral()
```

Each client has isolated in-memory state.

## Example test

```swift
import FeatherStorage
import FeatherStorageEphemeral
import NIOCore
import Testing

@Test
func uploadStoresObject() async throws {
    let storage = StorageClientEphemeral()

    let buffer = ByteBuffer(string: "test file")
    let sequence = StorageSequence(
        asyncSequence: ByteBufferSequence(buffer: buffer),
        length: UInt64(buffer.readableBytes)
    )

    try await storage.upload(
        key: "uploads/test.txt",
        sequence: sequence
    )

    #expect(try await storage.exists(key: "uploads/test.txt"))
    #expect(try await storage.size(key: "uploads/test.txt") == 9)
}
```

Because the driver conforms to `StorageClient`, the same application code can run against S3, filesystem storage, or in-memory storage.

## Test application services

The ephemeral driver is especially useful when your application hides storage behind a domain-specific protocol:

```swift
struct DocumentService {
    let storage: any StorageClient

    func saveDocument(
        id: String,
        bytes: StorageSequence
    ) async throws {
        try await storage.upload(
            key: "documents/\(id).pdf",
            sequence: bytes
        )
    }
}
```

Your test can provide `StorageClientEphemeral()`:

```swift
let service = DocumentService(
    storage: StorageClientEphemeral()
)
```

This keeps tests fast and predictable while still exercising the same storage API your production code uses.
