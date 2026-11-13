---
category: "mail"
title: "Testing"
description: "Learn how to test Feather Mail code with the in-memory FeatherMailTesting library."
order: 5
---

# Testing
---

Feather Mail provides the `FeatherMailTesting` library for in-memory mail tests and previews. It is included in the main `feather-mail` package and gives tests a `MailClient` implementation that records messages without sending them.

`TestMailClient` conforms to `MailClient`, validates messages, and records them in an actor-isolated in-memory mailbox. It is for tests and previews, not for delivering production mail.

## Installation

Add the package:

```swift
.package(
    url: "https://github.com/feather-framework/feather-mail",
    exact: "1.0.0-rc.3"
),
```

Add both products to the test target that needs them:

```swift
.product(
    name: "FeatherMail",
    package: "feather-mail"
),
.product(
    name: "FeatherMailTesting",
    package: "feather-mail"
),
```

## Send and inspect a message

```swift
import FeatherMail
import FeatherMailTesting
import Testing

let client = TestMailClient()
let mail = Mail(
    from: .init("from@example.com"),
    to: [.init("to@example.com")],
    subject: "Welcome",
    body: .plainText("Hello!")
)

try await client.send(mail)

#expect(await client.getMailbox().map(\.subject) == ["Welcome"])
```

Messages are returned in insertion order. Reset the mailbox between test cases when the same client is shared:

```swift
await client.clearMailbox()
#expect(await client.getMailbox().isEmpty)
```

## Test validation failures

`TestMailbox` accepts a custom validator, so a test can force a specific validation failure without using SMTP or SES:

```swift
let mailbox = TestMailbox(
    validator: TestMailValidator(error: .invalidSubject)
)
let client = TestMailClient(mailbox: mailbox)

do {
    try await client.send(mail)
    Issue.record("Expected validation to fail")
}
catch {
    #expect(await client.getMailbox().isEmpty)
}
```

Use [SMTP](/docs/mail/smtp/) or [Amazon SES](/docs/mail/ses/) when the application should deliver real messages. `FeatherMailTesting` is deliberately an in-memory testing client, not another mail transport.
