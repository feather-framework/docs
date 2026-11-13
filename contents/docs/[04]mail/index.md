---
type: "category"
title: "Mail"
description: "Learn how Feather Mail helps Swift applications send email through a common interface and interchangeable drivers."
order: 6
---

# Mail
---

Feather Mail provides a small, consistent API for sending email from Swift applications.

Most applications eventually need to send sign-in links, invitations, password reset messages, receipts, notifications, or system alerts. Feather Mail keeps that work behind one interface, so application code does not need to know whether delivery happens through SMTP, Amazon SES, or a local test client.

Use `FeatherMailTesting` for in-memory tests and previews, SMTP for standard mail servers, and SES when your infrastructure uses Amazon Simple Email Service.

This section explains the common mail model first, then walks through each driver.
