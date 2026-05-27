---
category: "mail"
title: "Ephemeral"
description: "Learn how to use the in-memory Feather Mail client for tests and local development."
order: 2
---

# Ephemeral
---

[Feather Mail Ephemeral](https://github.com/feather-framework/feather-mail-ephemeral) is an in-memory mail client.

It validates messages and stores them in an actor-isolated mailbox instead of sending them over the network. This makes it ideal for tests, previews, and local development.

## Installation

```swift
.package(
    url: "https://github.com/feather-framework/feather-mail-ephemeral",
    exact: "1.0.0-beta.2"
),
```

Add the product:

```swift
.product(
    name: "FeatherMailEphemeral",
    package: "feather-mail-ephemeral"
),
```

## Create the client

```swift
import FeatherMail
import FeatherMailEphemeral

let mailClient = MailClientEphemeral()
```

## Send and inspect mail

```swift
let mail = Mail(
    from: .init("hello@example.com"),
    to: [.init("user@example.com")],
    subject: "Welcome",
    body: .plainText("Hello!")
)

try await mailClient.send(mail)

let mailbox = await mailClient.getMailbox()
```

The mailbox returns messages in insertion order, so tests can make clear assertions:

```swift
#expect(mailbox.count == 1)
#expect(mailbox.first?.subject == "Welcome")
```

## Clear the mailbox

```swift
await mailClient.clearMailbox()
```

This is useful when a test performs multiple steps and needs a clean mailbox between them.

## Use a custom validator

The underlying `EphemeralMailbox` accepts a custom `MailValidator`.

```swift
let mailbox = EphemeralMailbox(
    validator: BasicMailValidator(
        maxTotalAttachmentSize: 1_000_000
    )
)

let mailClient = MailClientEphemeral(mailbox: mailbox)
```

This lets tests verify validation behavior without sending any real messages.

## Why this is good for tests

The ephemeral client behaves like a real mail client in the important ways:

- It conforms to `MailClient`.
- It validates messages before storing them.
- It is concurrency-safe.
- It lets you inspect sent messages.

This means your tests can check that a use case sends the correct email without depending on SMTP, Amazon SES, or external network access.
