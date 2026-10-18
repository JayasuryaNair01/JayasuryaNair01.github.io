---
title: "DOM Clobbering to Bypass an HTML Filter: How I Approached It"
date: 2026-10-18 12:55:00 +0530
categories: [Web Security, XSS]
tags: [dom-clobbering, xss, mystery-lab, burp, portswigger]
---

## What DOM clobbering is

DOM clobbering is when HTML you control creates elements whose `id` or `name` overrides a property that JavaScript later reads. The page thinks it's looking at a real DOM property (like `.attributes`), but it's actually looking at your element. That confusion can bypass filters that walk the DOM to decide what to keep.

## The lab

PortSwigger's "Clobbering DOM attributes to bypass HTML filters." Comments go through the HTMLJanitor sanitizer. The goal is to call `print()` in the victim's browser. Chrome only; the intended solution doesn't work in Firefox.

## What the sanitizer allowed

The page constructed HTMLJanitor with a tight whitelist:

~~~javascript
let janitor = new HTMLJanitor({
  tags: {
    input: { name: true, type: true, value: true },
    form: { id: true },
    i: {},
    b: {},
    p: {}
  }
});
~~~

Only those tags and attributes survive. Everything else is stripped. The library decides what to keep by reading each node's `.attributes` property.

## The bypass

I posted this as a comment:

~~~html
<form id=x tabindex=0 onfocus=print()><input id=attributes>
~~~

Why it survives:

- `<form>` and `<input>` are on the whitelist, and `id` is an allowed attribute on `form`.
- An `<input id=attributes>` clobbers the parent form's `.attributes` property in Chrome.
- When HTMLJanitor later checks `.attributes` on the form, it sees the clobbered (broken) property instead of the real attribute list, so `tabindex` and `onfocus` are never stripped.

After posting, a form/textbox showed up in the comment. That was confirmation the clobber worked and the extra attributes lived.

## Triggering it

`onfocus` only fires when the element receives focus. Using the exploit server, I delivered an iframe that loads the post, waits briefly for the comment to appear, then appends `#x` so the browser focuses the form:

~~~html
<iframe src="https://LAB-ID.web-security-academy.net/post?postId=6"
        onload="setTimeout(()=>this.src=this.src+'#x',500)">
</iframe>
~~~

Delivering that to the victim called `print()` and solved the lab.

## Impact

A "safe" client-side sanitizer becomes a stored XSS channel. Anyone viewing the page runs attacker JavaScript: session theft, account actions, further pivots. The interesting part for real apps is that the filter looked correct on paper and still failed because it trusted a clobberable DOM property.

## How it gets fixed

- Prefer a battle-tested sanitizer that is not vulnerable this way (and keep it updated), rather than a small custom whitelist walker.
- Never read clobberable DOM properties (`.attributes`, `.children`, named forms) to make security decisions without defending against clobbering.
- Serve user HTML with a strict Content-Security-Policy so even a bypassed filter can't reach `print()`-equivalent code easily.

The fix that doesn't work is "we only allow form and input." Those are exactly the tags used for the clobber.

## What I'm still learning

I want more reps spotting which properties a sanitizer reads, then testing whether `id`/`name` on an allowed tag can override them. That's the habit, more than memorizing one payload.
