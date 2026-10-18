---
title: "HTTP Request Smuggling: How I Approached It"
date: 2026-10-18 12:40:00 +0530
categories: [Web Security, Request Smuggling]
tags: [request-smuggling, http, burp, portswigger]
---

## What it is

Request smuggling happens when a front-end server (a proxy or load balancer) and the back-end server disagree about where one HTTP request ends and the next begins. They measure request length two ways, `Content-Length` (CL) and `Transfer-Encoding: chunked` (TE), and if each trusts a different one, an attacker can hide a second request inside the first. The front-end sees one request; the back-end splits it into two, and the smuggled part gets prepended to the next visitor's request.

## The three shapes

- **CL.TE** — front-end uses `Content-Length`, back-end uses `Transfer-Encoding`.
- **TE.CL** — the reverse.
- **TE.TE** — both support chunked, but the header is obfuscated (e.g. `Transfer-Encoding: xchunked`, or a space before the colon) so one side ignores it.

## The lab

PortSwigger's CL.TE lab. The front-end doesn't support chunked encoding and only allows GET/POST. The goal is to smuggle a request so the back-end sees the method `GPOST` on the next request.

## What I sent

~~~http
POST / HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

G
~~~

The front-end trusts `Content-Length: 6` and forwards the whole body. The back-end reads `Transfer-Encoding: chunked`, sees the `0` as the end of the body, and treats the leftover `G` as the start of the *next* request. When a real request follows, its line gets glued onto my `G`, so the back-end tries the method `GPOST` and replies: `Unrecognized method GPOST`.

## How I confirmed it

I sent the request twice. The second response came back with `Unrecognized method GPOST`, which is proof the back-end queued my leftover bytes and merged them with the follow-up request.

## Impact

Smuggling lets an attacker poison the request queue: capture other users' requests and sessions, bypass front-end access controls, poison the cache, or turn a reflected input into a stored one for the next visitor. It's high impact because it hits other users, not just yourself.

## How it gets fixed

- Make the front-end reject any request that carries both `Content-Length` and `Transfer-Encoding`.
- Use HTTP/2 end to end; its framing removes the length ambiguity entirely.
- Keep proxies and servers patched and configured to parse strictly.

The fix that doesn't work is hoping the two servers happen to agree. They need to be forced to.

## What I'm still learning

Timing-based detection for blind smuggling is the part I want more reps on, spotting the hang when the back-end waits for chunk bytes that never come.
