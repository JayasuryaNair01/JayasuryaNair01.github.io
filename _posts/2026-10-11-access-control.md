---
title: "Access Control & IDOR Cheat Sheet"
date: 2026-10-11 12:00:00 +0530
categories: [Security Notes, Web Vulnerabilities]
tags: [access-control, idor]
---

# Access Control Cheat Sheet

Access control ensures users only access resources they are permitted to.

## Common Flaws
* **Horizontal Privilege Escalation (IDOR):** Changing parameters like `user_id=101` to `user_id=102`.
* **Vertical Privilege Escalation:** Accessing `/admin` endpoints from an unprivileged account.
