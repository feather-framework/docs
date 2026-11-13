---
category: "storage"
title: "Testing"
description: "Learn how to test Feather Storage code with the in-memory FeatherStorageTesting library."
order: 4
---

# Testing
---

Feather Storage provides the `FeatherStorageTesting` library for in-memory storage tests and previews. It is included in the main `feather-storage` package and gives tests a `StorageClient` implementation without writing to disk or contacting an external object store.

`TestStorageClient` implements `StorageClient` in memory, so tests can exercise the same storage abstraction without writing to disk or contacting an external object store.

## Installation

Add the package:

```swift
.package(
    url: "https://github.com/feather-framework/feather-storage",
    exact: "1.0.0-rc.3"
),
```

Add both products to the test target that needs them:

```swift
.product(
    name: "FeatherStorage",
    package: "feather-storage"
),
.product(
    name: "FeatherStorageTesting",
    package: "feather-storage"
),
```

## Create a test client

```swift
import FeatherStorage
import FeatherStorageTesting
import NIOCore

let storage = TestStorageClient()
let sequence = StorageSequence(
    buffer: ByteBuffer(string: "test file")
)

try await storage.upload(
    key: "uploads/test.txt",
    sequence: sequence,
    contentType: "text/plain"
)
```

The client keeps objects, directories, content types, and multipart upload state in memory.

## Assert stored content

```swift
import Testing

#expect(try await storage.exists(key: "uploads/test.txt"))
#expect(try await storage.size(key: "uploads/test.txt") == 9)
#expect(await storage.state.contentType(for: "uploads/test.txt") == "text/plain")
#expect(await storage.state.objectBytes(for: "uploads/test.txt") == [
    116, 101, 115, 116, 32, 102, 105, 108, 101
])
```

`TestStorageState` is actor-isolated and can be supplied explicitly when a test needs to inspect or reset shared state:

```swift
let state = TestStorageState()
let storage = TestStorageClient(state: state)

await state.clear()
```

## Use it behind an application protocol

Application services can depend on `StorageClient` instead of the test implementation:

```swift
struct DocumentService {
    let storage: any StorageClient

    func saveDocument(
        id: String,
        bytes: StorageSequence
    ) async throws {
        try await storage.upload(
            key: "documents/\(id).pdf",
            sequence: bytes,
            contentType: "application/pdf"
        )
    }
}

let service = DocumentService(storage: TestStorageClient())
```

This keeps tests fast and deterministic while exercising the same storage API used by the filesystem and S3 drivers.
