---
title: "URL-Based Access Control Can Be Circumvented: How I Approached It"
date: 2026-10-18 12:00:00 +0530
categories: [Web Security, Access Control]
tags: [access-control, idor, burp, portswigger]
---

## What access control actually is

Access control is the server deciding whether the person making a request is
allowed to do the thing they're asking for. It breaks in two ways: horizontal
(I reach another user's data at my own privilege level, classic IDOR) and
vertical (a normal user reaches admin functions that were never meant for them).

The interesting part is almost never the URL. It's what the server *trusts* when
it decides who you are and what you can touch.

## The lab

PortSwigger's "URL-based access control can be circumvented." Visiting `/admin`
gets blocked. The goal is to reach the admin panel anyway and delete the user
`carlos`.

## What I noticed first

When I requested `/admin`, the block came back as a very plain response, not the
styled application error I'd expect from the app itself. That plainness is the
tell: it suggests a front-end component (a proxy or gateway) is making the
access decision *before* the request reaches the back-end, and the back-end
itself may not be checking at all.


## Testing the theory

Front-end proxies often support a header like `X-Original-URL` to tell the
back-end which path was really requested. So I sent `GET /` but added
`X-Original-URL: /invalid`. The response came back "Not Found" rather than the
home page.

![X-Original-URL invalid returns 404](/assets/img/access-control/x-original-invalid.png)

That 404 is the whole finding. The back-end ignored the real request line `/`
and acted on `/invalid` from my header. It trusts the header. So the front-end
guards `/admin`, but the back-end reads the path from somewhere the front-end
never inspects.

## Exploiting it

From there it's direct:

- `X-Original-URL: /admin` reached the admin panel the front-end was hiding.
- To delete carlos, I kept `X-Original-URL: /admin/delete` and put
  `?username=carlos` on the real query string (the front-end passes the query
  through, the back-end acts on the header path).

![Admin page reached via X-Original-URL](/assets/img/access-control/admin-reached.png)

## Impact

A standard or unauthenticated user performs admin actions, here deleting another
user. Any endpoint protected only at the front-end by path is reachable this
way: data exposure, account takeover, destructive actions.

## How it gets fixed

Enforce authorization at the back-end, on the actual authenticated session and
the actual resource, not on a path a front-end rewrote. Deny by default and
require explicit allow rules per role. The fix that *doesn't* work is blocking
`/admin` at the proxy and assuming you're done, which is exactly what failed
here.

## What I'm still learning

Spotting when two systems disagree about "what URL is this" is a pattern I want
to get faster at, because the same gap shows up in request smuggling and cache
deception too.
