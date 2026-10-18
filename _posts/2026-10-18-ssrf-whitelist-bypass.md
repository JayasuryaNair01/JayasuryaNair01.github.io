---
title: "SSRF with Whitelist Bypass (Mystery Lab): How I Approached It"
date: 2026-10-18 12:50:00 +0530
categories: [Web Security, SSRF]
tags: [ssrf, mystery-lab, burp, portswigger]
---

## What SSRF is

Server-Side Request Forgery is when the application makes an HTTP request on your behalf, and you get to choose (or influence) the URL. If that URL can point inside the network, you can hit admin panels, metadata endpoints, or other hosts the outside world should never reach.

## The lab

PortSwigger's "SSRF with whitelist-based input filter." There's a "Check stock" feature that fetches a URL server-side. The goal is to reach `http://localhost/admin` and delete `carlos`, past a whitelist that only allows `stock.weliketoshop.net`.

## How I figured out what it was

The interesting parameter was `stockApi`. Changing it to `http://127.0.0.1/` got a whitelist error, which told me two things at once: the server really does fetch that URL, and it parses out the hostname before allowing the request. That combination is classic SSRF with a hostname allowlist.

## Probing the parser

I tested URL features one by one and watched for a *different* error:

- `http://username@stock.weliketoshop.net/` was accepted. The parser understands embedded credentials (`user@host`).
- Putting a `#` in the username got rejected.
- Double-URL-encoding the `#` to `%2523` returned `500 Internal Server Error` ("Could not connect"). That 500 was the whole finding: the filter and the actual request parser had disagreed. The filter still saw a whitelisted host, but the request side treated the part before `%2523` as the real host and tried to connect to it.

## The working payload

~~~http
stockApi=http://localhost:80%2523@stock.weliketoshop.net/admin/delete?username=carlos
~~~

Reading it the way each side does:

- **Filter:** host = `stock.weliketoshop.net` → allowed.
- **Request parser (after one decode):** `http://localhost:80#@stock.weliketoshop.net/...` → host becomes `localhost:80`, and everything after `#` is a fragment it ignores. So it actually requests `http://localhost:80/admin/delete?username=carlos`.

That deleted carlos and solved the lab.

## Impact

Whitelist SSRF bypasses usually mean full reach into the internal network: admin panels, other services, sometimes cloud metadata. Here it was account deletion; on a real product it can be data theft or remote compromise of anything listening on localhost.

## How it gets fixed

- Don't let users supply full URLs. Prefer an allowlisted set of resource IDs the server maps to URLs itself.
- If a URL must be accepted, parse it with one well-defined library, then re-serialize and fetch that result. Never validate with one parser and fetch with another.
- Block credentials, fragments, and unexpected ports in the allowlist check.
- Deny private/link-local ranges even after the hostname check passes.

The fix that doesn't work is "check the hostname string contains `stock.weliketoshop.net`." String checks and real URL parsers disagree, and that disagreement is the bypass.

## What I'm still learning

I want to get faster at spotting the "different error" moment. The 500 was the signal the filter had been left behind; noticing that quickly is what turns a blocked SSRF into a real one.
