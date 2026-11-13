---
category: "mail"
title: "SMTP"
description: "Learn how to configure and use the Feather Mail SMTP driver."
order: 3
---

# SMTP
---

[Feather Mail SMTP](https://github.com/feather-framework/feather-mail-smtp) sends mail through an SMTP server.

SMTP is a good fit when your hosting provider, company infrastructure, or email service gives you SMTP credentials. Many transactional email providers support SMTP because it is widely understood and easy to integrate.

## Installation

```swift
.package(
    url: "https://github.com/feather-framework/feather-mail-smtp",
    exact: "1.0.0-rc.3"
),
```

Add the product:

```swift
.product(
    name: "FeatherMailSMTP",
    package: "feather-mail-smtp"
),
```

## Imports

```swift
import FeatherMail
import FeatherMailSMTP
import NIOSMTP
```

## Create the client

`MailClientSMTP` is initialized with an SMTP configuration, a mail encoder, and an optional validator.

```swift
let configuration = Configuration(
    hostname: "smtp.example.com",
    port: 587,
    username: "smtp-user",
    password: "smtp-password"
)

let mailClient = MailClientSMTP(
    configuration: configuration,
    mailEncoder: RawMailEncoder(),
    validator: BasicMailValidator()
)
```

The exact `Configuration` options come from `NIOSMTP`, the underlying SMTP library used by the driver.

## Send a message

```swift
let mail = Mail(
    from: .init("hello@example.com", name: "Example App"),
    to: [.init("user@example.com")],
    subject: "Your login link",
    body: .plainText("Use this link to continue.")
)

try await mailClient.send(mail)
```

The SMTP driver validates the message, encodes it as an SMTP-compatible payload, and sends it through the configured SMTP server.

## TLS and credentials

Use TLS whenever your SMTP provider supports it. For production systems:

- Prefer ports and settings recommended by your mail provider.
- Store SMTP credentials in environment variables or a secret manager.
- Use a dedicated SMTP user with limited permissions.
- Avoid logging message bodies or credentials.
- Keep validation enabled before delivery.

SMTP credentials can usually send real mail, so treat them like production secrets.

## Where to configure it

Create the SMTP client during server startup and inject it into your application services.

For larger applications, prefer an app-level protocol such as `MailSender`. The infrastructure layer can adapt `MailClientSMTP` to that protocol, while use cases remain independent from SMTP details.
