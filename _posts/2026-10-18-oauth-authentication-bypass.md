---
title: "OAuth Authentication Bypass: How I Approached It"
date: 2026-10-18 14:00:00 +0530
categories: [Web Security, Authentication]
tags: [oauth, authentication, burp, portswigger]
---

## What OAuth is doing here

OAuth 2.0 lets an app say "log in with Google/Apple" without ever touching your password. Three parties talk: you, the client app, and the authorization server. When any of them trusts the wrong thing, an attacker can steal a token, force an account link, or log in as someone else.

## The lab

PortSwigger's authentication-bypass OAuth lab. The blog lets you "Log in with social media." The goal is to log in as `carlos`.

## What I looked at first

I proxied a normal OAuth login through Burp and watched the whole handshake in HTTP history, starting from:

```http
GET /auth?client_id=...&redirect_uri=...&response_type=...
After the OAuth service handed identity info back, the blog itself logged me in with its own request:
POST /authenticate HTTP/2
Host: <blog>

email=<my-email>&...&access_token=<token>
That POST is the interesting one. The blog is taking an email from the request body and deciding "this is who you are." The OAuth token proves someone authenticated. It does not prove that the email in the body matches that someone.

Breaking the trust
I sent POST /authenticate to Repeater and changed only the email:
POST /authenticate HTTP/2
Host: <blog>

email=carlos@carlos-montoya.net&...&access_token=<my-token>
No error. Then I used Burp's "Request in browser → In original session," opened that URL, and I was logged in as Carlos. The lab was solved.

Why this worked
The client app trusted a client-controlled field (the email) instead of trusting only what the authorization server asserted. My access token was real. The email was a lie. The server mixed the two and gave me Carlos's session.

Impact
Full account takeover of any user whose email you know. On a real product that would mean reading their data, changing their settings, or chaining into whatever that account can do.

How it gets fixed
Treat identity claims as coming only from the authorization server (or from a userinfo endpoint you call yourself with the token). Never take email / sub / username from the client's POST body.
Validate that the access token actually belongs to the identity you are about to create a session for.
Prefer the authorization-code grant with PKCE. Drop the implicit flow.
Always bind a random, session-tied state parameter so the callback cannot be CSRF'd.
A "fix" that does not work: checking that an access token is present, then still reading the email from the body. That is exactly what failed here.

What I'm still learning
I want to get faster at mapping every hop of an OAuth flow and asking, at each hop, "who asserted this value, and who is trusting it?" The same question shows up in SAML and in JWT kid / algorithm confusion bugs.
