---
category: "mail"
title: "Overview"
description: "Learn what Feather Mail is and how it helps you send emails from Swift applications."
order: 1
---

# Overview
---

[Feather Mail](https://github.com/feather-framework/feather-mail) is a mail abstraction for Swift applications.

It gives your application a shared way to describe and send email messages while leaving the delivery provider open. The same application code can send through SMTP, Amazon SES, an in-memory client for tests, or a custom implementation.

## Why use Feather Mail?

Sending email looks simple at first, but real applications quickly need more structure.

You may need to send login links, invitations, password reset messages, reports, receipts, or notifications. You may also need different behavior in each environment:

- In production, send real mail through SMTP or Amazon SES.
- In development, avoid sending real messages accidentally.
- In tests, inspect the message that would have been sent.

Feather Mail gives these cases one common API.

## The mail client

The core protocol is `MailClient`:

```swift
public protocol MailClient: Sendable {
    func send(_ mail: Mail) async throws(MailError)
    func validate(_ mail: Mail) async throws(MailValidationError)
}
```

`send` delivers the message through the selected implementation. `validate` checks the message without delivering it.

## Create a message

A `Mail` value describes the message:

```swift
import FeatherMail

let mail = Mail(
    from: .init("hello@example.com", name: "Example App"),
    to: [
        .init("user@example.com", name: "Taylor")
    ],
    subject: "Welcome to Example App",
    body: .plainText("Thanks for signing up.")
)
```

The message can include:

- `from`
- `to`
- `cc`
- `bcc`
- `replyTo`
- `subject`
- `body`
- `reference`
- `attachments`

Bodies can be plain text or HTML:

```swift
let body = Body.html("<p>Thanks for signing up.</p>")
```

Attachments include a filename, content type, and raw bytes:

```swift
let attachment = Attachment(
    name: "invoice.pdf",
    contentType: "application/pdf",
    data: pdfBytes
)
```

## Validation

Feather Mail includes `BasicMailValidator`, which performs practical checks before delivery:

- The sender must not be empty.
- The subject must not be empty.
- At least one recipient must be valid.
- Basic header injection attempts are rejected.
- Attachment size can be limited.

```swift
let validator = BasicMailValidator(
    maxTotalAttachmentSize: 7_500_000
)

try await validator.validate(mail)
```

The SES driver uses a default attachment limit because many real mail providers have strict message size limits.

## Keep application code clean

For small applications, injecting `MailClient` directly is fine.

For larger applications, it is often better to define an application-level protocol and adapt Feather Mail in the server or infrastructure layer. A common pattern is to let the application layer define a simple `MailSender`, then let the server layer adapt it to Feather Mail.

```swift
public protocol MailSender: Sendable {
    func send(_ message: MailMessage) async throws
}

struct FeatherMailSender: MailSender {
    let client: any MailClient

    func send(_ message: MailMessage) async throws {
        try await client.send(
            .init(
                from: .init(message.from.email, name: message.from.name),
                to: message.to.map { .init($0.email, name: $0.name) },
                subject: message.subject,
                body: .plainText(message.body)
            )
        )
    }
}
```

This keeps use cases focused on business behavior. A login use case can ask `MailSender` to send a magic link without knowing whether the message goes through SMTP, SES, or an in-memory mailbox.

## Available implementations

The currently available mail implementations are:

- [SMTP](https://github.com/feather-framework/feather-mail-smtp)
- [Amazon SES](https://github.com/feather-framework/feather-mail-ses)

Use `FeatherMailTesting` for in-memory tests, then configure SMTP or SES for environments that should send real messages.
