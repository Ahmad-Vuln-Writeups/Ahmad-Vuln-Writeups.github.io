---
title: "One Extra Parameter Was All It Took to Redirect a Password Reset on REDACTED.com"
date: 2026-08-01 12:30:00 +0300
categories: [Bug Bounty, Web Security]
tags: [hpp, parameter-pollution, account-takeover, api-security, writeup]
---

A write-up on how sending the same parameter twice caused a validated reset request to deliver its token to a completely different inbox.

| Target | Vulnerability | Severity | Status |
| :--- | :--- | :--- | :--- |
| `REDACTED.com` | Account Takeover via HTTP Parameter Pollution (HPP) | **Critical** | 🟢 Reported & Patched |

---

### 📌 Starting From "This Looks Fine," Again

Consistent with everything else I've found on this target, the reset flow held up against the standard checks — token randomness, expiry, per-account binding, rate limiting, all solid. I wasn't looking for a flaw in the token itself this time. I was looking at the request that triggers the token, because that's a part of the flow people test less carefully.

The reset request looked like this:

```http
POST /api/reset-password
Content-Type: application/x-www-form-urlencoded

email=victim@example.com
```

Simple, one field, does what you'd expect — sends a reset link to `victim@example.com`.

### 🔍 The Idea

Web frameworks and the servers in front of them don't all agree on what to do when a request contains the same parameter more than once. Some take the first occurrence, some take the last, some silently concatenate them into an array or a comma-separated string, and some just error out. 

That inconsistency is well known, but it only becomes interesting when a request passes through more than one layer — say, a reverse proxy or WAF, and then the application server behind it — and those two layers don't agree with each other on which value wins.

If the layer doing the "is this email address allowed to receive a reset" validation reads one occurrence of the parameter, but the layer that actually sends the token reads a different occurrence, you've got a gap worth testing.

So I sent the parameter twice:

```http
POST /api/reset-password
Content-Type: application/x-www-form-urlencoded

email=victim@example.com&email=attacker@example.com
```

### 🗺️ What Happened

The response came back identical to a normal, successful reset request — same generic "if an account with that email exists, a reset link has been sent" message the app always returns, regardless of whether the email is real (which, correctly, avoids leaking whether an account exists — good practice on its own).

The actual reset email landed in the attacker inbox — the second value — while the response gave no indication anything unusual had happened. No error, no warning, nothing to suggest the request had been interpreted any differently than a normal single-parameter request would have been.

### 📥 Why This Happened

Digging into it, the cause was a mismatch between two layers handling the request, not a single flawed piece of code:

1. **The application's validation logic** — the part checking "does an account with this email exist, and should we proceed" — read the first occurrence of the email parameter, so it validated against the real victim's account.
2. **The mail-dispatch logic** — a separate internal service responsible for actually sending the token — parsed the request differently and picked up the last occurrence when building the recipient address.

Two pieces of code, written and maintained by different parts of the backend, each individually reasonable, each parsing duplicate parameters in a technically valid but different way. Neither was "wrong" in isolation — HTTP doesn't strictly define which duplicate parameter should win, so both interpretations are defensible reads of a malformed-ish request. The problem was that nobody had verified the two services agreed with each other.

### ⚠️ Confirming Impact

To be sure this wasn't some one-off inconsistency, I repeated the test against a second test account I controlled, this time reversing which email I put first versus second, to check whether the behavior was consistent (last-value-wins for dispatch) or just something order-dependent and unreliable. It held consistently: first occurrence validated, last occurrence received the token, every time.

At that point the chain was obvious and complete: knowing nothing about a victim beyond their email address — no password, no session, no prior access — I could trigger a reset that the system validated correctly against their account, but delivered the actual reset link into an inbox I controlled. 

From there, setting a new password and logging in as them was trivial, so I didn't need to actually complete that last step to prove severity.

### 📋 Scoping the Report Responsibly

I tested this exclusively across accounts I created and controlled myself, confirming the parameter-order behavior held consistently before writing anything up. I didn't run it against any real account outside of my own test setup — the two consistent, controlled tests were enough to demonstrate the flaw clearly without needing to touch anyone else's data.

### 📋 The Report

I documented:
* The exact duplicate-parameter request and the mismatched outcome (validated against value one, delivered to value two)
* An explanation of the two-layer parsing mismatch as the root cause, rather than framing it as a single broken function
* Two controlled, reproducible test cases showing the behavior was consistent, not a fluke
* A recommended fix: reject any request containing duplicate instances of a security-sensitive parameter outright, rather than silently picking one, and — more importantly — ensure every service touching that request parses it identically, ideally by validating and normalizing input at a single point before it fans out to other internal services

### 🛠️ The Fix

This one got escalated internally faster than most of my other reports, which made sense given how little an attacker needed to know to use it. Within a few days they'd added strict rejection of duplicate parameters at the API gateway layer — returning a 400 if email appeared more than once — closing the gap regardless of how the downstream services individually parsed things. 

They mentioned auditing other sensitive endpoints (account recovery, invite flows) for the same class of inconsistency as a follow-up.

### 💡 Takeaway

Parameter pollution bugs are easy to overlook because they don't look like an attack — there's no injection syntax, no encoded payload, nothing that trips a WAF or looks unusual in a log. It's just the same field, twice. On targets with more than one service touching the same request, that alone can be enough to make two layers disagree about something as fundamental as who's supposed to receive a password reset token.
