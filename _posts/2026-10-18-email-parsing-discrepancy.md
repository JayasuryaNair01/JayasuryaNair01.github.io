---
title: "Bypassing Access Controls via Email Parsing Discrepancies (Mystery Lab): How I Approached It"
date: 2026-10-18 13:05:00 +0530
categories: [Web Security, Business Logic]
tags: [business-logic, email-parsing, mystery-lab, burp, portswigger]
---

## What this is

Two pieces of code can read the same email address differently: the validation layer that checks "is this domain allowed?" and the mail library that actually decides where to send. When they disagree, you can craft an address that passes the check but delivers somewhere else.

## The lab

PortSwigger's "Bypassing access controls using email address parsing discrepancies." Registration only allows `@ginandjuice.shop`. The goal is to register an account, reach the admin panel, and delete `carlos`.

## How I figured out what it was

`foo@exploit-server.net` was blocked with "email domain must be ginandjuice.shop," so there's a domain check. The interesting question became: does the validator parse encoded-word email syntax the same way the mail server does?

## Probing the encodings

I tested encoded-word formats one at a time:

- `=?iso-8859-1?q?=61=62=63?=foo@ginandjuice.shop` → blocked ("Registration blocked for security reasons"). The server detects Q-encoding.
- `=?utf-8?q?=61=62=63?=foo@ginandjuice.shop` → blocked the same way.
- `=?utf-7?q?&AGEAYgBj-?=foo@ginandjuice.shop` → **no error.** The validator doesn't recognize UTF-7 as a threat.

That gap is the finding: UTF-7 encoding slips past validation but still gets decoded by the mail layer.

## The exploit

~~~
=?utf-7?q?attacker&AEA-[YOUR-EXPLOIT-SERVER-ID]&ACA-?=@ginandjuice.shop
~~~

This decodes to `attacker@[YOUR-EXPLOIT-SERVER-ID] ?=@ginandjuice.shop`, with the `@` and a space UTF-7 encoded. Validation sees the trailing `@ginandjuice.shop` and allows it. The mail server decodes the UTF-7 and sends the confirmation to `attacker@[YOUR-EXPLOIT-SERVER-ID]`.

I opened the email client, clicked the confirmation link, logged in, reached the admin panel, and deleted carlos.

## Impact

Register under a trusted domain you don't own. That can mean access to internal-only features, admin panels, or anything gated by "must be an @company.com email."

## How it gets fixed

- Parse the email once, with the same library the mail system uses, then validate the parsed result. Never validate the raw string and send with a different parser.
- Reject encoded-word syntax in registration emails entirely.
- Verify ownership by sending to the final resolved address and requiring a click, rather than trusting the domain substring.

The fix that doesn't work is blacklisting specific encodings (ISO-8859-1, UTF-8). UTF-7 walked straight through that.

## What I'm still learning

This is from Gareth Heyes' "Splitting the Email Atom." I want more reps recognizing when two parsers touch the same input, because that's the whole family: this, WCD, SSRF filters, request smuggling.
