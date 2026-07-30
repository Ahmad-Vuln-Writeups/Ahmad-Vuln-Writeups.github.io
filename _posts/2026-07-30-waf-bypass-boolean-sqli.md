---
title: "Bypassing a WAF to Land SQL Injection on REDACTED.com"
date: 2026-07-30 14:10:00 +0300
categories: [Bug Bounty, Web Security]
tags: [sqli, waf-bypass, blind-sqli, writeup]
---

A write-up on what happens when the vulnerability is real, but every obvious payload gets blocked.

| Target | Vulnerability | Severity | Status |
| :--- | :--- | :--- | :--- |
| `REDACTED.com` | SQL Injection (WAF Bypass, Boolean-Based Blind) | **High** | 🟢 Reported & Patched |

---

### 📌 The Annoying Part First

Usually when I find a parameter that smells vulnerable, confirming it is quick — throw in a quote, see what breaks, move on. This one wasn't quick. It was annoying, in the specific way that tells you a WAF is sitting in front of the app doing its job.

The target was a filter parameter on a listing page:

```url
https://redacted.com/items?sort=price_asc
```

Swapping sort for something unexpected caused a subtle change in behavior — not an error, just a difference in how many results came back. That's usually a decent sign something's being evaluated server-side rather than just used as a static label.

### 🔍 Getting Blocked, Repeatedly

I tried the basics first, the stuff that would work on an unprotected endpoint:

```sql
sort=price_asc'+OR+'1'='1
sort=price_asc'+UNION+SELECT+NULL--
```

Both got me an instant 403 with a generic "Request Blocked" page. No stack trace, no useful error — just a wall. That's a WAF signature-matching common SQLi keywords and blocking them outright, which meant the underlying vulnerability might still be very real; I just needed a payload that didn't trip the filter.

### 🗺️ Figuring Out What It Was Blocking

Rather than guessing randomly, I started isolating which specific tokens triggered the block. I sent a series of near-identical requests, changing one thing at a time:

* `UNION+SELECT` -> blocked
* `union+select` (lowercase) -> blocked
* `UNION`+alone -> blocked
* `OR+'1'='1` -> blocked
* Just `'` alone -> not blocked

So the WAF wasn't blocking the quote character itself — it was pattern-matching on specific keyword combinations like UNION SELECT and OR 1=1, case-insensitively. That's a fairly common and fairly weak WAF setup: keyword-based, not behavior-based.

### 🧪 Working Around It

Once I knew it was matching literal keyword patterns, the goal was to express the same logic without matching the pattern. A few things that got through where the obvious versions didn't:

* Inline comments to split keywords the WAF was matching as whole strings, e.g. breaking UNION into `UNI/**/ON`
* Using OR logic phrased differently, since `'1'='1'` was flagged but equivalent boolean logic phrased another way wasn't
* URL-encoding specific characters that the WAF appeared to inspect pre-decode

I want to be careful here and not turn this into a step-by-step WAF evasion cheat sheet — the specific bypass string isn't the point. The point is the underlying technique: WAFs that rely on keyword blacklists can almost always be bypassed by restructuring the query, not by avoiding SQL logic altogether. The vulnerability was never in the WAF's hands — it was in the app not reserving input at the query layer, and the WAF was just a band-aid on top of it.

Once I had a payload structured to slip past the filter, I confirmed the injection was real using boolean-based blind testing — comparing response differences between a condition I knew was true versus one I knew was false, rather than relying on error messages or timing strings.

### ⚠️ Why This Still Mattered

A WAF stopping the obvious payloads gives a false sense of security — to the dev team and sometimes to less careful testers too. But a WAF is a mitigation, not a fix. The actual root cause — unsanitized input reaching a raw SQL query — was completely untouched by the WAF's presence. Anyone with enough patience to iterate on payloads would eventually get through, same as I did.

### 📄 The Report

I reported two things, deliberately kept separate:

1. **The real bug**: Unsanitized input in the sort parameter reaching the query layer — fix with parameterized queries, not more WAF rules.
2. **The false confidence problem**: I flagged that the WAF's keyword-blacklist approach was giving a false sense of protection, and recommended it be treated as defense-in-depth, not the actual fix.

The team's initial instinct was to add a new WAF rule for the specific pattern I'd used. I pushed back a bit and pointed them toward fixing the query itself — which they did, about two weeks later, alongside tightening the WAF rules as a secondary layer.

### 💡 Takeaway

A WAF blocking your payload isn't a "no," it's a "not like that." If a parameter still feels off after the standard tests get blocked, it's worth asking what the WAF is actually matching on rather than assuming the endpoint is safe. Just make sure that when you do find a bypass, the write-up focuses on the underlying flaw and the lesson — not on handing out working evasion strings.
