---
title: "An IDOR That Let Me View Anyone's Invoices on REDACTED.com"
date: 2026-08-01 10:37:00 +0300
categories: [Bug Bounty, Web Security]
tags: [idor, access-control, web-security, writeup]
---

A write-up on how a predictable ID and a missing ownership check turned into full account data exposure.

| Target | Vulnerability | Severity | Status |
| :--- | :--- | :--- | :--- |
| `REDACTED.com` | Insecure Direct Object Reference (IDOR) | **High** | 🟢 Reported & Patched |

---

### 📌 Where It Started

This one came out of just using the app normally. I'd signed up for a test account, poked around the dashboard, and landed on the billing section, where I could view my own invoices. Nothing unusual — until I looked at the URL for one of them:

```url
https://redacted.com/account/invoices/8841
```

A plain numeric ID sitting right there in the path. My brain does this thing now where any sequential-looking ID in a URL immediately makes me wonder: what happens if I just... change the number.

### 🔍 The First Test

I dropped the ID down by one:

```url
https://redacted.com/account/invoices/8840
```

And it loaded. A full invoice — name, billing address, purchase details, partial payment info — that clearly wasn't mine.

No error, no redirect, no "you don't have permission" page. Just someone else's private billing data, rendered exactly like it belonged to me.

### 🗺️ Making Sure It Wasn't a Fluke

Before getting excited, I wanted to rule out the boring explanations — maybe it was test data, maybe IDs weren't actually tied to individual accounts, maybe I'd stumbled onto some kind of shared demo invoice.

So I tried a handful of other IDs, spaced out and randomized rather than just walking sequentially:

* `/account/invoices/7213`
* `/account/invoices/9502`
* `/account/invoices/6688`

Every single one returned a different real invoice, with different names and different billing details. That ruled out demo data — this was live production data, and the endpoint clearly wasn't checking whether the invoice ID actually belonged to the logged-in user.

### 📥 Why It Happened

The likely explanation, based on the behavior, was pretty simple: the backend was probably doing something like:

1. Take `invoice_id` from the URL
2. Query the invoice with that ID
3. Return it

...without a step in between checking "does this invoice belong to the account making this request?" The user was authenticated, sure — logged in with a valid session — but authentication isn't authorization. The app confirmed who I was, but never checked whether I was allowed to see the specific resource I was asking for.

That distinction is basically the entire IDOR bug class in one sentence.

### ⚠️ Scoping the Impact

Since invoice IDs looked sequential, this wasn't just "one person's data leaked by coincidence" — it meant the entire invoice history of the platform was enumerable. Anyone could write a simple loop incrementing the ID and pull invoices for what looked like every user who'd ever been billed.

I didn't do that. I confirmed the pattern with a small, manually-checked sample — enough to demonstrate the vulnerability clearly and prove it wasn't isolated — and stopped there. Mass-scraping user data, even to "prove severity," crosses from proof-of-concept into actual harm, and that's not something I do during testing.

### 📋 The Report

I documented:
* The vulnerable endpoint and parameter (`invoice_id`)
* A small number of ID/response pairs showing the pattern (redacted before sending, obviously)
* An explanation of why this was enumerable at scale, without actually enumerating it myself
* A suggested fix: server-side ownership checks on every object fetch tied to a user ID or session, plus considering non-sequential identifiers (UUIDs) so IDs aren't easily guessable in the first place

### 🛠️ The Fix

The team's first response added an ownership check on the invoice endpoint, which addressed the core issue quickly. 

In a follow-up release a few weeks later, they migrated invoice IDs to UUIDs as a secondary hardening measure — which doesn't fix a missing authorization check on its own, but does remove the "just guess the next number" trivial exploitation path.

### 💡 Takeaway

IDORs are boring to explain and devastating in practice, which is exactly why they're worth hunting for. There's no clever payload, no filter to bypass, no injection syntax to craft — just permission checks, and whether someone remembered to write them. Any time you see a raw ID in a URL, path, or request body, it costs nothing to try the number next to it.
