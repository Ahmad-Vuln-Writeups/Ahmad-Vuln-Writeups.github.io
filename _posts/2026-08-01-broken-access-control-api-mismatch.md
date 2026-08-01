---
title: "The Admin Panel Was Locked Down — Except for One Endpoint Nobody Re-Checked on REDACTED.com"
date: 2026-08-01 11:00:00 +0300
categories: [Bug Bounty, Web Security]
tags: [broken-access-control, api-security, mobile-testing, writeup]
---

A write-up on how a properly protected admin panel still had one API endpoint trusting the wrong signal for authorization.

| Target | Vulnerability | Severity | Status |
| :--- | :--- | :--- | :--- |
| `REDACTED.com` | Broken Access Control (Inconsistent Authorization) | **High** | 🟢 Reported & Patched |

---

### 📌 Starting From "This Looks Fine"

I want to be upfront about how this one started, because it wasn't a dramatic discovery — it was mostly ruling things out.

I tried all the standard admin panel checks first, expecting one of them to work the way these things sometimes do on less careful targets:

* `https://redacted.com/admin`
* `https://redacted.com/admin/dashboard`
* `https://redacted.com/api/admin/users`

Every single one, as a regular authenticated account, correctly returned a 403. No leaked route in the JS bundle, no client-side-only gate. The main admin surface was properly enforcing server-side role checks. Normally at this point I'd move on — a locked-down admin panel isn't usually worth more time.

What kept me digging was noticing that REDACTED.com had both a web app and a separate mobile app, and mobile apps often talk to a slightly different (or older) version of the API than the web frontend does. That mismatch is a pattern worth checking, because backend teams don't always keep every API version's middleware in sync when they update the main one.

### 🔍 Diffing the Two APIs

I intercepted traffic from the mobile app using a proxy and compared the endpoints it hit against the ones the web app used for equivalent features. Most of them matched up — same paths, same auth handling. 

But the mobile app's account management screen was calling a versioned endpoint the web app didn't use at all:

```http
GET /api/v1/support/user-details?userId=4471
```

versus the web app's:

```http
GET /api/v2/users/4471
```

Same underlying purpose — fetch a user's account details — but two different code paths. That's exactly the kind of seam where authorization logic gets duplicated, and duplicated logic is where inconsistencies live.

### 🗺️ Testing the Older Endpoint

I sent the same request the mobile app sent, but from my own low-privilege test account's session, hitting the v1 endpoint directly instead of through the mobile client:

```http
GET /api/v1/support/user-details?userId=4471
Authorization: Bearer <my regular user token>
```

It worked. Full account details for a user that wasn't me, returned without a role check rejecting the request.

### 📥 Figuring Out Why

My first assumption was that this endpoint just had no auth at all, but that turned out to be wrong — removing the token entirely did return a 401. So it was authenticating me fine. It just wasn't authorizing me.

Looking at the response headers and a bit of trial and error, the pattern became clear: this legacy v1 endpoint was checking for the presence of a `X-Client-Type: mobile-support` header to decide whether to apply the stricter role check the v2 endpoint used — a shortcut almost certainly added at some point to let an internal support-tooling version of the mobile app skip a slower permission lookup for performance reasons. 

The v2 web endpoint validated the user's actual role against the database on every request, no matter what. The v1 endpoint, if it saw that header, trusted the caller was already a vetted internal client and skipped the check.

The header wasn't a secret. It wasn't a token, wasn't signed, wasn't tied to anything — it was just a string, and I could send it from any request I wanted:

```http
GET /api/v1/support/user-details?userId=4471
X-Client-Type: mobile-support
Authorization: Bearer <my regular user token>
```

Same result: full user data, no role check, using nothing but a plain low-privilege account and a header I copied straight out of the mobile app's traffic.

### ⚠️ Scoping the Impact

This wasn't as broad as a fully open admin dashboard, but it wasn't narrow either. The `v1/support/*` namespace turned out to cover more than one endpoint once I checked — user detail lookups, subscription status, and support ticket history were all reachable the same way, all behind the same header trick.

I tested exactly enough to confirm the pattern held across a few different data types (again, using a small number of accounts I could verify without touching anyone else's actual private data further than necessary) and stopped there. I didn't attempt to find write-capable endpoints on the same legacy API version — read access was already more than sufficient to demonstrate severity, and hunting for a way to modify data on a version of the API clearly meant to be phased out felt like the wrong kind of thorough.

### 📋 The Report

I documented:
* Both endpoint versions side by side, showing the enforcement gap
* The exact header responsible, and confirmation it required no secret value, signature, or internal-only network position — just knowledge that it existed
* Which additional endpoints under the same `v1/support` path shared the same flaw
* A recommendation to either retire the v1 endpoints entirely if v2 had fully replaced them, or route both versions through the same centralized authorization middleware rather than maintaining separate logic

### 🛠️ The Fix

This took a bit longer to resolve than some of my other reports — about three weeks — which made sense once the team explained why: they had to audit every endpoint still living under `/api/v1/` to check for the same pattern before shipping a fix, not just patch the one I'd found. 

They ended up deprecating the header-based shortcut entirely and pointing all v1 support endpoints through the same role-check logic v2 used.

### 💡 Takeaway

A target can do the "obvious" access control right — properly gated main routes, no client-side-only checks, sensible 403s everywhere you'd expect — and still have a gap sitting in a legacy or secondary code path that never got the same scrutiny. Comparing how different clients (web vs. mobile, old API version vs. new) talk to the same backend is often more productive on a well-hardened target than repeating the same tests against the front door everyone already checked.
