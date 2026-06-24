---
category: "storage"
title: "S3"
description: "Learn how to configure and use the Feather Storage S3 driver."
order: 3
---

# S3
---

[Feather Storage S3](https://github.com/feather-framework/feather-storage-s3) is the S3-compatible driver for Feather Storage.

Use it when files should live in Amazon S3 or an S3-compatible service such as MinIO, Cloudflare R2, DigitalOcean Spaces, or another object storage provider.

## Installation

Add the S3 driver to your `Package.swift` dependencies:

```swift
.package(
    url: "https://github.com/feather-framework/feather-storage-s3",
    exact: "1.0.0-beta.2"
),
```

Then add the product to your target:

```swift
.product(
    name: "FeatherStorageS3",
    package: "feather-storage-s3"
),
```

The driver uses [Soto](https://github.com/soto-project/soto) for AWS and S3-compatible API calls.

## Imports

```swift
import FeatherStorage
import FeatherStorageS3
import Logging
import SotoCore
```

## Create an S3 client

Create a configured Soto `S3` client, then wrap it with `StorageClientS3`:

```swift
let awsClient = AWSClient(
    credentialProvider: .static(
        accessKeyId: "minioadmin",
        secretAccessKey: "minioadmin"
    ),
    httpClientProvider: .createNew
)

let s3 = S3(
    client: awsClient,
    region: .useast1,
    endpoint: "http://127.0.0.1:9000"
)

let storage = StorageClientS3(
    s3: s3,
    bucket: "app-storage",
    logger: Logger(label: "app.storage.s3")
)
```

In production, load credentials and endpoint values from your environment or secret manager. Do not hard-code real credentials.

## Upload and download

The S3 driver uses the same `StorageClient` API as the other drivers:

```swift
try await storage.upload(
    key: "avatars/user-123/profile.png",
    sequence: imageBytes
)

let downloaded = try await storage.download(
    key: "avatars/user-123/profile.png",
    range: nil
)
```

You can also download a byte range:

```swift
let preview = try await storage.download(
    key: "documents/manual.pdf",
    range: 0...4095
)
```

For S3 this maps to a byte range request.

## Object operations

```swift
let exists = try await storage.exists(
    key: "avatars/user-123/profile.png"
)

let size = try await storage.size(
    key: "avatars/user-123/profile.png"
)

let entries = try await storage.list(
    key: "avatars/user-123"
)

try await storage.copy(
    key: "avatars/user-123/profile.png",
    to: "avatars/user-123/profile-backup.png"
)

try await storage.delete(
    key: "avatars/user-123/profile-backup.png"
)
```

S3 does not have real directories in the same way a filesystem does. Prefixes are part of the object key. The driver presents this in a directory-like way so application code can stay simple.

## Multipart upload

The S3 driver supports the common Feather Storage multipart upload flow:

```swift
let uploadId = try await storage.createMultipartId(
    key: "videos/session.mov"
)

let first = try await storage.upload(
    multipartId: uploadId,
    key: "videos/session.mov",
    number: 1,
    sequence: firstPart
)

let second = try await storage.upload(
    multipartId: uploadId,
    key: "videos/session.mov",
    number: 2,
    sequence: secondPart
)

try await storage.finish(
    multipartId: uploadId,
    key: "videos/session.mov",
    chunks: [first, second]
)
```

Always abort incomplete uploads when you know the upload cannot finish:

```swift
try await storage.abort(
    multipartId: uploadId,
    key: "videos/session.mov"
)
```

This helps avoid leaving unfinished multipart upload state in your bucket.

## Local testing with MinIO

For local S3-compatible testing, MinIO is a practical option.

```yaml
services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
```

Start it with:

```sh
docker compose up minio
```

Then create a bucket named `app-storage` through the MinIO console at `http://127.0.0.1:9001`, or create it from your test setup.

## Security notes

For production:

- Use TLS endpoints.
- Store access keys outside source control.
- Prefer short-lived credentials when your platform supports them.
- Restrict credentials to only the buckets and actions your application needs.
- Use private buckets by default.
- Generate public URLs only when the file is intentionally public.

S3-compatible storage is powerful, but credentials are sensitive. Treat them like database passwords.
