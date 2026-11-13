---
category: "mail"
title: "Amazon SES"
description: "Learn how to configure and use the Feather Mail Amazon SES driver."
order: 4
---

# Amazon SES
---

[Feather Mail SES](https://github.com/feather-framework/feather-mail-ses) sends mail through Amazon Simple Email Service.

SES is a strong choice for production transactional email on AWS. It supports high-volume delivery, verified sender identities, domain authentication, and integration with other AWS infrastructure.

## Installation

```swift
.package(
    url: "https://github.com/feather-framework/feather-mail-ses",
    exact: "1.0.0-rc.4"
),
```

Add the product:

```swift
.product(
    name: "FeatherMailSES",
    package: "feather-mail-ses"
),
```

## Imports

```swift
import FeatherMail
import FeatherMailSES
import Logging
import SotoCore
```

## Create the client

Create an AWS client and an SES v2 client, then pass it to `MailClientSES`:

```swift
let awsClient = AWSClient(
    credentialProvider: .default,
    logger: Logger.current
)

let ses = SESv2(
    client: awsClient,
    region: .init(rawValue: "eu-central-1"),
    partition: .aws,
    endpoint: nil,
    timeout: nil,
    byteBufferAllocator: .init(),
    options: []
)

let mailClient = MailClientSES(
    ses: ses,
    encoder: RawMailEncoder()
)
```

Create the AWS client, SES client, and mail client once during application startup. Keep the `AWSClient` in the application's service lifecycle so its networking resources are shut down when the application stops.

In production, prefer platform-managed credentials where possible. On AWS, this usually means IAM roles instead of hard-coded access keys.

## Send a message

```swift
let mail = Mail(
    from: .init("hello@example.com", name: "Example App"),
    to: [.init("user@example.com")],
    subject: "Your receipt",
    body: .html("<p>Thanks for your purchase.</p>")
)

try await mailClient.send(mail)
```

The SES driver validates the mail, encodes it as a raw MIME message, base64-encodes the payload, and sends it through the SES v2 API.

## Local testing with Floci

The Feather CMS development stack uses [Floci](https://floci.io/) for local AWS-compatible services. Floci captures SES messages for inspection, so you can test the complete mail flow with local credentials and a locally verified sender identity.

Start Floci from the Feather CMS repository:

```sh
docker compose up floci
```

For a host process, point the SES client at the published Floci port:

```env
AWS_ACCESS_KEY_ID=test
AWS_SECRET_ACCESS_KEY=test
SES_REGION=eu-central-1
SES_ENDPOINT=http://127.0.0.1:4566
SES_FROM_EMAIL=no-reply@feather.local
```

When the application runs inside the Compose network, use `SES_ENDPOINT=http://floci:4566` instead. Floci verifies the default sender identity on startup and exposes captured messages at `http://127.0.0.1:4566/_aws/ses` from the host.

## Attachment size

The SES driver uses `BasicMailValidator(maxTotalAttachmentSize: 7_500_000)` by default.

This keeps message size under a practical limit before encoding and transport overhead are added. If your application has stricter rules, pass your own validator:

```swift
let mailClient = MailClientSES(
    ses: ses,
    encoder: RawMailEncoder(),
    validator: BasicMailValidator(
        maxTotalAttachmentSize: 2_000_000
    )
)
```

## Production checklist

Before sending real mail through SES:

- Verify the sender email or domain in AWS.
- Configure SPF, DKIM, and DMARC for your domain.
- Move the account out of the SES sandbox if needed.
- Use IAM permissions limited to SES sending.
- Store configuration outside source control.
- Monitor bounces and complaints.

SES is reliable, but deliverability still depends on domain setup and sender reputation.
