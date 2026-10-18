---
title: "Web Cache Deception: How I Approached It"
date: 2026-10-18 12:15:00 +0530
categories: [Web Security, Web Cache Deception]
tags: [web-cache-deception, caching, burp, portswigger]
---

## What it is

Web cache deception tricks a cache (a CDN or reverse proxy) into storing someone's private response and serving it to a stranger. It works because the cache and the origin server disagree about one thing: what counts as a static file.

The attacker gets a logged-in victim to load a URL like `/my-account` dressed up to look static. The origin ignores the disguise and returns the victim's private data. The cache sees "static" and stores it. The attacker then requests the same URL with no cookies and gets the cached private copy.

## The two disagreements that make it work

Every WCD bug comes down to one of these:

- **Path delimiters.** The origin treats a character (`;`, `?`, `#`) as "the path ends here," so `/my-account;x.js` still resolves to `/my-account`. The cache doesn't, so it sees a `.js` file and caches it.
- **Normalization.** The origin resolves `..%2f`, so `/resources/..%2fmy-account` resolves to `/my-account`. The cache doesn't resolve it, sees the `/resources` static prefix, and caches it.

Either way, the two parsers read the same URL differently, and that gap is the whole vulnerability.

## The lab (path delimiters)

PortSwigger's "exploiting path delimiters" lab. Logged in as `wiener`, `/my-account` returns my API key. The goal is to steal carlos's key.

## How I worked it

First I found which characters the origin treats as delimiters. I sent `/my-account/abc` (404) and `/my-accountabc` (404) as my baselines, then fuzzed one character at a time in Intruder with URL-encoding turned off:

~~~http
GET /my-account§x§abc HTTP/2
~~~

Sorting by status code, `;` and `?` both returned 200 with my API key. So the origin stops parsing the path at `;` and `?`.

Then I checked whether the cache agrees. `/my-account?abc.js` showed no caching (the cache also treats `?` as a delimiter), but `/my-account;abc.js` came back `X-Cache: miss`, and on a second send flipped to `X-Cache: hit`. That's the finding: the origin ignores everything after `;`, so it serves my account page, but the cache keeps the `;...js` and caches it as a static file.

## Landing it on carlos

Using the exploit server, I sent carlos a page that navigates him to the crafted URL, with a unique string so his own response gets cached under a fresh key:

~~~html
<script>document.location="https://LAB-ID.web-security-academy.net/my-account;wcd.js"</script>
~~~

When carlos loaded it, the cache stored his account page. I then opened the same URL with no session and read his API key straight from the cache.

## Impact

Any logged-in user's private response can be cached and handed to an anonymous attacker: API keys, CSRF tokens, personal data, sometimes session material. Zero interaction beyond getting the victim to load a page, and it scales to every user you can lure.

## How it gets fixed

- Send `Cache-Control: no-store` on every dynamic, per-user response.
- Make the cache and origin normalize paths and parse extensions identically, so they can never disagree.
- Have the cache decide what's cacheable by the response `Content-Type`, not the URL's extension.
- `SameSite=Lax`/`Strict` cookies blunt the cross-site `<img>`/redirect delivery.

The fix that doesn't work is trusting the file extension in the URL. That's exactly the signal the attack forges.

## What I'm still learning

The normalization variant (`..%2f` tricks) is the one I want more reps on, because which side decodes the dot-segment flips the whole exploit, and telling them apart quickly takes practice.
