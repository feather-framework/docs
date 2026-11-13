---
category: "storage"
title: "S3"
description: "Learn how to configure and use the Feather Storage S3 driver."
order: 3
---

# S3
---

[Feather Storage S3](https://github.com/feather-framework/feather-storage-s3) is the S3-compatible driver for Feather Storage.

Use it when files should live in Amazon S3 or an S3-compatible object storage provider. The Feather CMS development stack uses [Floci](https://floci.io/) as its local AWS-compatible service instead of a separate S3 emulator.

## Installation

Add the S3 driver to your `Package.swift` dependencies:

```swift
.package(
    url: "https://github.com/feather-framework/feather-storage-s3",
    exact: "1.0.0-rc.3"
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

Create a configured AWS client, then wrap it with the current `StorageClientS3` initializer:

```swift
let awsClient = AWSClient(
    credentialProvider: .default,
    logger: Logger.current
)

let storage = StorageClientS3(
    awsClient: awsClient,
    region: "us-east-1",
    endpoint: "http://127.0.0.1:4566",
    bucket: "feather-cms-app"
)
```

For local development, Floci is started by the Feather CMS Docker Compose stack and exposes its AWS-compatible APIs on port `4566`. The application uses the AWS default credential provider, with local `test` credentials supplied by the Compose stack or `scripts/run-local.sh`. In production, leave the endpoint empty for AWS S3 and load credentials from the standard provider chain or a secret manager.

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

## Local testing with Floci

The Feather CMS repository integrates Floci into its root Docker Compose stack. Floci provides the local S3-compatible API, creates the application bucket during startup, and also supplies the local SES and CloudFront-compatible media services.

```yaml
services:
  floci:
    image: floci/floci:latest-compat
    ports:
      - "${FLOCI_PUBLISHED_PORT:-4566}:4566"
    environment:
      FLOCI_HOSTNAME: floci
      FLOCI_STORAGE_MODE: persistent
      FLOCI_STORAGE_PERSISTENT_PATH: /app/data
      FLOCI_DEFAULT_REGION: us-east-1
      STORAGE_S3_BUCKET: feather-cms-app
      STORAGE_S3_REGION: us-east-1
    volumes:
      - floci-data:/app/data

volumes:
  floci-data:
```

Start it with:

```sh
docker compose up floci
```

The local application connects to `http://127.0.0.1:4566` with `STORAGE_S3_REGION=us-east-1` and `STORAGE_S3_BUCKET=feather-cms-app`. The Feather CMS init hook creates the bucket automatically, so no separate web console step is required. When running the application inside Compose, use `http://floci:4566` instead of `127.0.0.1`.

## Security notes

For production:

- Use TLS endpoints.
- Store access keys outside source control.
- Prefer short-lived credentials when your platform supports them.
- Restrict credentials to only the buckets and actions your application needs.
- Use private buckets by default.
- Generate public URLs only when the file is intentionally public.

S3-compatible storage is powerful, but credentials are sensitive. Treat them like database passwords.
